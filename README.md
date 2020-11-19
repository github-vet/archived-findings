# rangeloop-findings
Rangeloop findings in GitHub projects.

## Where's the Code?!

This repository is "issue-only". It exists to capture the findings of [a static analysis bot](https://github.com/github-vet/vet-bot), which reads Golang repositories found on GitHub and captures examples of the [range loop capture error](https://github.com/golang/go/wiki/CommonMistakes#using-reference-to-loop-iterator-variable).

Range-loop capture is the reason this code prints `4, 4, 4, 4,` instead of what you might expect.

```go
xs := []int{1, 2, 3, 4}
for _, x := range xs {
    go func() {
        fmt.Printf("%d, " x)
    }()
}
```

But why raise issues for all these code examples?

## Range Loop Capture Considered Dangerous

Members of the Go language team have indicated a willingness to modify the behavior of range loop variable capture to make the behavior of Go more intuitive. This change could theoretically be made despite the [strong backwards compatibility guarantee](https://golang.org/doc/go1compat) of Go version 1, **only if** we can ensure that the change will not result in incorrect behavior in current programs.

To make that determination, a large number of "real world" `go` programs would need to be vetted. If we find that, in every case, the current compiler behavior results in an undesirable outcome (aka bugs), we can consider making a change to the language.

The goal of the [github-vet](https://github.com/github-vet) project is to motivate such a change by gathering static analysis results from Go code hosted in publicly available GitHub repositories, and crowd-sourcing their human analysis.

## How Can I Help?

We're glad you're here! We need the help of the Go community to examine our bot's findings to determine if they represent a bug or not.

Each instance falls into one of three buckets (explained further below).
1. **Bug** - as written, the captured loop variable could lead to undesirable behavior.
1. **Desirable Behavior** - the current capture semantics are required for the correct operation of the program. Changing the behavior of the compiler would break this code.
1. **Mitigated** - the captured loop variable does not escape the block of the for loop.

### Bugs

An example is a bug when the captured loop variable causes "undesirable" or "confusing" behavior. These terms are somewhat subjective but it depends on the programmer's intent. While we can't _know_ the programmer's original intent, we _can_ ask whether the actual behavior could be better achieved in a different way.

For example, [the below finding](https://github.com/github-vet/rangeloop-findings/issues/176) is an example of undesirable behavior. The loop variable `i` is captured within the block of the for loop. There is a race-condition between updating the value of `i` at the beginning of each loop and reading the value of `i` within the goroutine. Typically, this will result in printing `9` 10 times. Of course, if the programmer had actually intended to print `9` 10 times, they would probably have written different code. We therefore mark this as a bug.
```go
	for i := 0; i <= 9; i++ {
		go func() {
			fmt.Println(i)
		}()
	}
```

As another example, in [the below finding](https://github.com/github-vet/rangeloop-findings/issues/189) the undesirable behavior also occurs. Updates to the value of `n` race against the start of the goroutine, ensuring there is no guarantee that `.Run()` will be called on every value in `g.nodes`, which is clearly the intent of the range loop.
```go
for _, n := range g.nodes {
  wg.Add(1)
  go func() {
    err := n.Run()
    if err != nil {
      c <- err
    }
  }()
}
```

### Mitigated

These examples aren't bugs because they have been written in such a way that the capture is mitigated. "Mitigated" examples aren't as interesting as desirable behavior, but we want to mark them separately, so we can consider augmenting the analysis procedure to detect them separately.

##### Examples
For instance, the [below finding](https://github.com/github-vet/rangeloop-findings/issues/320) is 'mitigated' by explicitly passing the `index` variable into the function started via a goroutine.
```go
for index := 0; index < len(cmdsToTest); index++ {
  go func(index int) {
    errChan <- testCommand(cmdsToTest[index], newEnvs, scanForHome, home)
  }(index)
}
```
Another common instance of mitigated use is found in tests like [this one](https://github.com/github-vet/rangeloop-findings/issues/316) (which is too large to quote here). The `done` channel in this example is used to ensure that the goroutine started finishes before the end of the loop. This means that the value of `c` is not overwritten before the goroutine completes, so there is no race condition.

### Desirable Behavior

We don't currently have any examples of desirable behavior which depends on the current semantics of range-loop captures. The goal of the project is to determine whether such an example exists.

We don't think such an example exists. This means that any examples marked as 'Desirable' will be faced with skepticism and a high degree of scrutiny. This should not discourage reporting that they suspect desirable behavior. Let us know! We will chime in and give our view. Just be aware that we may disagree, so is important to engage with a sense of professional detachment.
