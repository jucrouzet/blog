---
title: "A little bit of context"
date: 2022-06-11T12:04:16-05:00
tags: ["go"]
---


Go's [`context`](https://pkg.go.dev/context) is a very central package and can be very useful and
powerful if it's used correctly. For example, the [`net/http`](https://pkg.go.dev/net/http) or the
the [`grpc`](https://pkg.go.dev/google.golang.org/grpc) packages rely on it.
On the other hand, its concepts can be a little difficult to understand and can be confusing, leading
developpers to skip it or use it in a wrong manner.

Let's try to explore it !

<!--more-->

### What is `context` ?

As the name suggests, `context` allows to pass a context to a program or a part of a program, like a
function, a request handler or a goroutine ; where a context can be some arbitrary or well-defined
values or a deadline.

For example, a context can be :

* A maximum allowed duration to execute a network call (aka "timeout")
* Some request-specific values in a HTTP handler like an authentification
* An application configuration passed to a sub package

A context is stored in a variable implementing the [`context.Context`](https://pkg.go.dev/context#Context)
interface, which is, by convention or as a best practice, passed to functions as their first argument, for example :

{{< highlight go >}}
func myContextedFunc(ctx context.Context, foo string, bar interface{}) error {
    // ...
    return nil
}
{{< /highlight >}}

This way, a developer knows that function taking a `context.Context` as its first argument will accept
and use context.

### So, how to use it ?

#### Root contexts

Contexts are almost all derived from another context, like russian dolls, meaning that they're created
by saying "I take a context, I add or remove these properties, and here's a new context".
Two notable exceptions are `context.Background()` and `context.TODO()`.

These two functions creates a new and empty context, with no values or deadline, and so they don't
take any arguments :

{{< highlight go >}}
ctx := context.Background()
ctx2 := context.TODO()
{{< /highlight >}}

In fact, if you look at the [`context` package source](https://cs.opensource.google/go/go/+/refs/tags/go1.18.3:src/context/context.go;l=199),
you'll see that they have the same value, `emptyCtx`, an empty context with no value or deadline.

So what's the difference between `context.Background()` and `context.TODO()` ? It's mostly lexical.

`context.Background()` should be called only once by program, in the highest level, like a main function
or an initializer package.
In the russian dolls image, it'll be the outer doll, the one containing all the others. The context
returned should be sent to the first function call that takes a package and derives it :

{{< highlight go >}}
package main

func main() {
    ctx := context.Background()

    ctx = otherPackageA.Init(ctx)
    ctx = otherPackageB.Init(ctx)

    // ...
}
{{< /highlight >}}

The context created in the highest level function will now "flows" through the program, each part
inheritng from previous context and adding its own specifics for the next ones.

`context.TODO()` should be used where you find yourself asking *"I must send a context, but I don't
know which one yet.*. You can see it as adding a comment like :

{{< highlight go >}}
// @TODO : We pass a new context here, it should be defined later accordingly
package.FuncReceivingContext(context.TODO(), otherArg)
{{< /highlight >}}

`FuncReceivingContext` will receive a new context, not derived from any other, with no value and no
deadline.

Some linters and static analysis tools like [contextcheck](https://github.com/sylvia7788/contextcheck)
will ensure that all contexts are derived from a single `context.Background()` and fails if code is
creating another background context elsewhere in the code. That's where `context.TODO()` is ok to use.

#### Derived contexts

The [`context`](https://pkg.go.dev/context) package offers 4 functions to create derived contexts.

##### `context.WithCancel(ctx context.Context) (context.Context, func())`

A call to `WithCancel` returns a new context derived from the context passed as argument and a function.

A call to the `Done()` method of the new context returns a [channel](https://go.dev/tour/concurrency/2)
that is closed when calling the function returned, called the *cancel function*.

Once the `Done()` channel is closed, the `Err()` method of the new context returns the `context.Canceled`
error (whose message is *"context canceled"*).

This pattern is used to indicate to a function when to stop something, for example :

{{< highlight go >}}
package main

import (
	"context"
	"fmt"
	"time"
)

func counter(ctx context.Context) {
	start := time.Now()

	// We wait for the Done() channel to be closed
	<-ctx.Done()
	// And we check the duration since we started waiting
	fmt.Printf("You waited %.2f seconds before calling cancel()\n", time.Since(start).Seconds())
}

func main() {
	// We create a new context with a cancel from the background context
	ctx, cancel := context.WithCancel(context.Background())
	// We launch counter() in background with the created context
	go counter(ctx)
	// We wait 0.3 seconds
	time.Sleep(300 * time.Millisecond)
	// And we call cancel() to stop the execution of counter()
	cancel()

	// Extra sleep is here to let the time for fmt.Printf to complete
	time.Sleep(time.Millisecond)
}

// You waited 0.30 seconds before calling cancel()
{{< /highlight >}}
*([try it !](https://go.dev/play/p/gKODzi_T46d))*

`WithCancel` is oftenly used in combination with [`defer()`](https://go.dev/tour/flowcontrol/12)
to run a task in background during a program execution :

{{< highlight go >}}
package main

import (
	"context"
	"fmt"
	"time"
)

func runInBackground(ctx context.Context) {
    tick := time.NewTicker(60 * time.Second)
    go func() {
        for {
            select {
            case <-ctx.Done():
                // main has stopped, we stop this background run
                tick.Stop()
                return // to stop goroutine execution (goroutine leak)
            case <- tick.C:
                // Do something every minute like cleaning up resources
            }
        }
    }()
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    runInBackground(ctx)

    // launch program ...
}
{{< /highlight >}}

If the context passed to `WithCancel` is already a cancel context, the `Done()` channel will be closed
when the returned cancel function is called or when the parent's context cancel function is called,
whichever comes first.

##### `context.WithDeadline(ctx context.Context, time.Time) (context.Context, func())`

`WithDeadline` creates a cancel context ([see above](#contextwithcancelctx-contextcontext-contextcontext-func)),
but with the assurance that `Done()` will be closed when the deadline passed as second argument is passed.

If the `Done()` channel is closed because the cancel function was called, the `Err()` method will return
the `context.Canceled` error, but if it's closed because the deadline has expired, it will return the
`context.DeadlineExceeded` error (whose message is *"context deadline exceeded"*).

This pattern is used to give a task a deadline to complete :

{{< highlight go >}}
package main

import (
	"context"
	"fmt"
	"time"
)

func doSomethingSlow(ctx context.Context) {
	select {
	case <-time.After(1 * time.Second):
		fmt.Printf("could do something slow\n")
	case <-ctx.Done():
		fmt.Printf("could not do something slow because %s happened\n", ctx.Err())
	}
}

func doSomethingFast(ctx context.Context) {
	select {
	case <-time.After(1 * time.Microsecond):
		fmt.Printf("could do something fast\n")
	case <-ctx.Done():
		fmt.Printf("could not do something fast because %s happened\n", ctx.Err())
	}
}

func main() {
	deadline := time.Now().Add(500 * time.Millisecond)
	ctx, cancel := context.WithDeadline(context.Background(), deadline)
	defer cancel()

	go doSomethingSlow(ctx)
	go doSomethingFast(ctx)

	time.Sleep(time.Second)
}

// could do something fast
// could not do something slow because context deadline exceeded happened
{{< /highlight >}}
*([try it !](https://go.dev/play/p/xH_-YgtnNAg))*

If the parent context (the one passed as first argument) already has a deadline set, the new context
will be set with the sooner deadline between parent's and passed ones.

Even if the deadline has passed, this is still the function creating the new context's responsability
to call the cancel function, as calling it will release resources allocated to create the deadline.
So **ALWAYS** call cancel() when creating Deadline context.

##### `context.WithTimeout(ctx context.Context, time.Duration) (context.Context, func())`

`WithTimeout` works exactly the same way as [`WithDeadline`](#contextwithdeadlinectx-contextcontext-timetime-contextcontext-func)
does, except that the second argument is not a time but a duration, to be added to current time.

Instead of doing

{{< highlight go >}}
deadline := time.Now().Add(500 * time.Millisecond)
newContext, cancel := context.WithDeadline(previousContext, deadline)
defer cancel()
{{< /highlight >}}

You just do

{{< highlight go >}}
newContext, cancel := context.WithTimeout(previousContext, 500 * time.Millisecond)
defer cancel()
{{< /highlight >}}


##### `context.WithValue(context.Context, key, value any) context.Context`

`WithValue` is used to create a new context with any arbitrary value in it. It can be very useful if
used wisely, respecting a few guidelines.

Basically, any value can be passed for the `key` and `value` as long as they're comparable:

{{< highlight go >}}
newContext := context.WithValue(previousContext, "answer", 42)
{{< /highlight >}}

But as a best practice (seriously, follow this one), it is important to not use any built-in type
(string, number, known struct or interface) for the `key` value.
For an even best practice, always use an unexported type of the package using the context's value.

The reason behind is that everyone can add a new value to any context using `WithValue`, that means that
if you use a string as key for example, a collision with any other package using the same string can
happen and the value will be replaced.

A typical implementation of this guideline can be :

{{< highlight go >}}
package myownpackage

type contextValueKeyType string

var contextValueKey = contextValueKeyType("my key name")

func SetContextValue(ctx context.Context, value string) context.Context {
    return context.WithValue(ctx, contextValueKey, value)
}

func GetContextValue(ctx context.Context) (string, error) {
    v, ok := context.Value(ctx, contextValueKey).(string)
    if !ok {
        return "", errors.New("value was not set in context")
    }
    return v, nil
}

{{< /highlight >}}

So it'll be used in other package without any possible collision :

{{< highlight go >}}
ctx := context.Background()

value, err := myownpackage.GetContextValue(ctx)
// err = errors.New("value was not set in context")

ctx = myownpackage.SetContextValue(ctx, "test")
value, err := myownpackage.GetContextValue(ctx)
// value = "test"

{{< /highlight >}}

It is important to keep in mind that context values are **not** meant to pass parameters to a function
but to keep a/some values in a context, for example, an authentification result through an http request
handling or a configuration through a program.

### Some examples

#### Passing a context : Fetching an http url with a timeout 

{{< highlight go >}}
package main

import (
	"context"
	"net/http"
	"time"
)

func main() {
    // We create a deadline context that expires in 2 seconds
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    // cancel() is called after execution to release resources
	defer cancel()
    
    // we pass ctx so that client will use this deadline context
	req, err := http.NewRequestWithContext(ctx, "GET", "http://www.google.com", nil)

	if err != nil {
		panic(err)
	}

	client := http.Client{}
	resp, err := client.Do(req)

	if err != nil {
		if err == context.DeadlineExceeded {
			panic("fetch took more than 2 seconds")
		}
		// another error than timeout
		panic(err)
	}
	println(resp)
}
{{< /highlight >}}

#### Using a context : :A function that accepts a deadline to call a function

{{< highlight go >}}
package main

import (
	"context"
	"fmt"
	"time"
)

// slowFunction is a slow function that takes 200 milliseconds to run
func slowFunction() int {
	time.Sleep(200 * time.Millisecond)
	return 42
}

// wrapWithDeadline will ensure that running slowFunction() is done before the deadline specified
// in ctx, if any.
func wrapWithDeadline(ctx context.Context) (int, error) {
	// We launch the execution of slowFunction() in a goroutine, result will be sent in a channel
	res := make(chan int)
	go func() {
		res <- slowFunction()
	}()

	// If the result channel receives the result of slowFunction first, the result is returned
	// If the deadline hits first, the context.DeadlineExceeded error is returned
	select {
	case <-ctx.Done():
		return 0, ctx.Err()
	case v := <-res:
		return v, nil
	}
}

func main() {
	ctx := context.Background()

	ctx1, cancel1 := context.WithTimeout(ctx, 100*time.Millisecond)
	defer cancel1()
	res, err := wrapWithDeadline(ctx1)
	fmt.Println(res) // 0
	fmt.Println(err) // context deadline exceeded

	ctx2, cancel2 := context.WithTimeout(ctx, 300*time.Millisecond)
	defer cancel2()
	res, err = wrapWithDeadline(ctx2)
	fmt.Println(res) // 42
	fmt.Println(err) // <nil>
}
{{< /highlight >}}
*([try it !](https://go.dev/play/p/omNlNVjsayK))*