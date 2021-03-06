# TDD in action : Debouncer in Go

> Applying TDD principles to the making of a helper in Go

## Introduction

I may be new to the Golang world, but here is something I am already familiar with: **Building things in Go feels great !**

What feels event better is to build those things with the absolute certainty that it works the way it was meant to be.

I am leaving the preaching and teaching parts to *Kent Beck* and consort. I can, however, tell you that the *Test Driven Development* (or *TDD*) principles are perfect to give you that certainty.

Following the TDD principles in Go does not require any particular setup. No dependency or IDE plugin to install and configure. You just grab your favorite text editor, a terminal plus a notebook and a pen if it suits you better !

## Writing tests in Go

Follow those few steps to start writing tests in Go :

1. In the same folder as the file under test, write a new file following this pattern : `yourFile_test.go`;
2. In this test file, create a new function with the specific name and signature : `func TestXxx(*testing.T)`
3. Manage your test the way you need to, using the API provided by the `testing.T` object;
4. Run the `go test` command
5. Analyse the result to see if the tests succeed

The standard API does not provide any fancy testing feature, but all we need for writing basic tests is there *out of the box*.

## The context

I am currently building a development environment in Go. This dev env will be capable of running all of the tests and building and executing the program under development.

To do so, I need a files watcher, notifying the dev env each time a file is created, modified or deleted.

The main issue I ran into is that the event triggered by the files watcher may be emitted too many times at once.

Nothing too fancy here, all I need is a rather common feature called **debouncing**.

I am sure that a lot of libraries may tackle this issue in Go. However, I found it to be a great challenge for the beginner that I am with this language.

So, let's write this in tiny little tested steps, making *Kent Beck* and the Clean Code community pride in the process !

## How does a debouncer look like ?

I am willing to create a debouncer in its most common API.

This means providing a debouncer as a helper function which takes a function as his first parameter and a timer as the second one. The timer represents how long should the program wait after the last call before executing the method.

The user will wrap his treatment in a debounced method, and call the wrapped function :

```go
// Wrapping the debounced treatment
myDebouncedFunction := debounce(func() {
    // The treatment...
}, 500)

// Example 1 - Simple call
myDebouncedFunction() // The method will be called in 500ms

// Example 2 - Multiple calls
myDebouncedFunction() // Method's execution cancelled by the next call to myDebouncedFunction
time.Sleep(100 * time.Millisecond)
myDebouncedFunction() // Method's execution cancelled by the next call to myDebouncedFunction
time.Sleep(100 * time.Millisecond)
myDebouncedFunction() // The method will only be called once in 500ms
```

## TDD

As I said earlier, I am not going into much details about what is TDD and why it is great. If you have not already, please read "Test Driven Development: By Example" by Kent Beck and try to apply the principles described there if they suits you.

Here is my take on the subject, if you are not familiar with it :

In order to build a product that really fits the needs that it addresses, the *TDD* proposes to first write the tests that the program should be able to pass.

To do so, we need to reduce the feedback loop to the bare minimum by following those three steps :

1. Write a test with a dreamed API, the way you think the program should work. This test should not go green yet;
2. Write the simpliest, dumbest implementation to make the test succeed. This way, you will know if the following modifications break the promise;
3. Refactor the code (and the test if needed) in order to really solve the problem raised by the test. **You have restrain yourself to the scope of the test**, do not refactor or optimise the code to solve future features just now.

Then you repeat the process with the next test, until all of the use cases are tackled and all of the features implemented

Let's jump into it, one test at the time !

## Calling the provided method after the timer has ended

The first thing I want to check is very basic, *is the provided method called ?*

Using only the standard API, and following the guideline introduced in the "*How does a debouncer look like ?*" chapter, one could implement this test like this :

```go
func TestShouldCallTheFunctionAfterTheProvidedTime(t *testing.T) {
    called := false
    debouncedMethod := Debounce(func() {
        called = true
    }, 1)

    debouncedMethod()

    time.Sleep(10 * time.Millisecond)

    if called == false {
        t.Error("The method was not called")
    }
}
```

The test wraps a method in a very short debouncer, then it waits for the debounce to happen. The last step is to check if the method was called, throwing an error if it does.

Now that we have a test in place, we can try to make it pass with the most obvious implementation.

We can just return a function that calls the provided method, without taking care of the timer :

```go
func Debounce(function func(), executeAfter int) func() {
    return func() {
        function()
    }
}
```

The test is green, we could now focus on refactoring. However, we will wait for the next test to be written before doing any refactoring, you will understand why shortly.

## Calling the method only after the timer

Next, we need to check that the method was only called after the provided duration.

The test will look very similar to the last one, but we will check the opposite assertion and wait less than the timer :

```go
func TestShouldNotCallTheFunctionBeforeTheProvidedTime(t *testing.T) {
    called := false
    debouncedMethod := Debounce(func() {
        called = true
    }, 10)

    debouncedMethod()

    time.Sleep(1 * time.Millisecond)

    if called == true {
        t.Error("The method was called too early")
    }
}
```

If you run it immediately, this test will not pass. The simplest way of changing that is to wait for the provided time before executing the function :

```go
func Debounce(function func(), executeAfter int) func() {
    return func() {
        go (func() {
            time.Sleep(time.Duration(executeAfter) * time.Millisecond)
            function()
        })()
    }
}
```

We need to wrap the method in a goroutine in order for the program to continue its execution, and not wait for the `time.Sleep` method to finish. Remember, the point here is to make the tests pass, and both did !

Now, we clearly have a design issue, even a standard API misusage. Time for a little refactoring !

We can change the behavior, while launching the tests at every step to assure everything still runs smoothly.

We have used the `time.Sleep` where the [`NewTimer` method of the same package](https://golang.org/pkg/time/#NewTimer) seems more appropriate.

First, we need to create a new timer based on the parameters and immediatly stop it :

```go
func Debounce(function func(), executeAfter int) func() {
    duration := time.Duration(executeAfter) * time.Millisecond
    t := time.NewTimer(duration)
    t.Stop()
    // [...]
```

A goroutine can be started next with an infinite loop, watching for the timer to be finished and calling the debounced method :

```go
    // [...]
    go (func() {
        for {
            select {
            case <-t.C:
                log.Println("Executing the debounced method")
                go function()
            }
        }
    })()
    // [...]
```

Then we can return a function that restarts the timer :

```go
    return func() {
        log.Println("Reset the debouncer timer")
        t.Reset(duration)
    }
```

This implementation lets the tests green, we can now move to the next tests with a little surprise...

## The last tests

Finally, the goal of the debouncer is to ensure that the method is only executed once if called multiple times.

We can check that by creating a new test that keeps track of the execution count. If this number is not one, the test fails :

```go
func TestShouldCallTheFunctionOnlyOnceAfterTheProvidedTime(t *testing.T) {
    executionCount := 0
    debouncedMethod := Debounce(func() {
        executionCount++
    }, 1)

    debouncedMethod()
    debouncedMethod()
    debouncedMethod()
    debouncedMethod()

    time.Sleep(5 * time.Millisecond)

    if executionCount != 1 {
        t.Errorf("The method was not called only once, called %v time(s)", executionCount)
    }
}
```

The good news is... our implementation already validates this rule ! No refactoring seems to be needed, we are just making our API more robust with the previous test and the following.

We also need to check that the method is executed multiple times if enough time was spent between the calls.

We just need to tweak the previous test a little :

```go
func TestShouldBeAbleToCallTheFunctionAgainAfterTheTimer(t *testing.T) {
    executionCount := 0
    debouncedMethod := Debounce(func() {
        executionCount++
    }, 5)

    debouncedMethod()

    time.Sleep(10 * time.Millisecond)

    debouncedMethod()

    time.Sleep(10 * time.Millisecond)

    if executionCount != 2 {
        t.Errorf("The method was not called twice, called %v time(s)", executionCount)
    }
}
```

## Conclusion

I cannot think of any other use case for this basic debouncer, and the API feels fine to me.

The complete implementation looks like this :

```go
func Debounce(function func(), executeAfter int) func() {
    duration := time.Duration(executeAfter) * time.Millisecond
    t := time.NewTimer(duration)
    t.Stop()

    go (func() {
        for {
            select {
            case <-t.C:
                log.Println("Executing the debounced method")
                go function()
            }
        }
    })()

    return func() {
        log.Println("Reset the debouncer timer")
        t.Reset(duration)
    }
}
```

This may not be the best debouncer out there, nor that it is the best scenario of a *test driven development*.

Yet I think it illustrates how natural it is to write tests and little helpers in Go.

I started writing the debouncer with no tests, leading to a wanky unpredictable API, that I had to test manually with a real world example to validate.

Starting over and driving the develoment by the tests helped me writing a way simpler helper that I could rely on !
