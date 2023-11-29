---
title: 'The Ultimate Guide to Graceful Applications in Golang'
date: 2023-11-17T22:35:05+03:30
draft: true
---

It's important to close that database connection or let a running loop finish before exiting.
In this blog post, We will explore ways of writing graceful Go applications and the constructs
that the language itself brings to the table. We start with a simple example and make it better 
each step of the way until we've reached the best solution (in my opinion). This is my first 
ever blog. I started blogging because I felt that I was losing precious time and not learning
anything new. This is my attempt to **Reinforce my current learnings** and **Force myself to learn new things** 
to be able to keep writing.

I hope you find this useful and leave my blog with a smile on your face ❤️.


# TL;DR


# Singal Handling

Signals are asynchronous events sent to a process to indicate an external event or condition. They are typically used to notify a process of a change in its environment, such as a user pressing Ctrl+C to terminate a program or the operating system sending a signal to indicate a system error.

## Signal Handling Process
When a signal is generated (either by the operating system or by another process), the normal 
flow of the executing program is interrupted, and the signal handler is invoked. A signal handler 
is a function or routine that is executed in response to a specific signal.

## Default Actions

Each signal has a default action associated with it, which is taken if the process does not explicitly handle the signal.
Common default actions include terminating the process (SIGTERM), aborting the process (SIGABRT), or ignoring the signal.
Registering Signal Handlers:

In many programming languages, developers can register their signal handlers to override the default behavior.
This is typically done using functions like signal() or platform-specific APIs.

## Common Signals
Some common signals include: 
- SIGINT (interrupt from keyboard)
- SIGSEGV (segmentation fault)
- SIGKILL (kill the process)
- SIGTERM (termination request)

## Signal Handling in Go

In Go (Golang), signal handling involves using the `os/signal` package to capture and respond to signals. 
The `os/signal` package provides a way to intercept signals sent to the program and perform custom actions 
in response. Here's a basic overview of how signal handling works in Go:

```golang
package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	sig := make(chan os.Signal, 1)

	signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)

	go func() {
		s := <-sig
		fmt.Println("received signal:", s)
		os.Exit(0)
	}()

	for {
		fmt.Println("doing stuff...")
		time.Sleep(time.Second)
	}
}
```

In this example, the `signal.Notify()` function registers the `sig` channel to receive notifications of the `SIGINT` and `SIGTERM` signals. The go function creates a goroutine to handle the signal. When a signal is received, the goroutine prints a message and then exits the program.

# Basic Example

This is a simple example that simulates an application with a very useful functionality, logging.
This application logs `doing stuff...`  to infinity and beyond until it receives an interrupt signal.
It also has a clean-up function which is required when your program is doing something this important.

```golang
package main

import (
	"log"
	"os"
	"time"
)

func init() {
	log.SetFlags(0)
	log.SetOutput(os.Stdout)
}

func main() {
	go runApplication()
	sig := make(chan os.Signal, 1)
	signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
	s := <-sig
	log.Println("received signal:", s)
}

func runApplication() {
	defer cleanupApplication()

	for {
		log.Println("doing stuff...")
		time.Sleep(time.Second)
	}
}

func cleanupApplication() {
	time.Sleep(5 * time.Second)
	log.Println("application done.")
}
```

However, when you exit this application (by pressing `Ctrl + C`), `cleanupApplication` doesn't have any
chance of running.

# Graceful using done channel and Waitgroups

Our simple example has 2 major problems:
1. It will never exit out of the loop
2. Our main function will never wait for the application cleanup


To overcome the initial challenge, We can utilize a pattern known as the Done Channel. For the additional concern, Go offers the WaitGroup solution from the sync package.

## Done Channels
A done channel is a simple channel used to signal that a goroutine has finished its work. It is typically used in conjunction with a wait group to ensure that all goroutines have been completed before proceeding with further execution. To put this into practice, we initiate a channel and transfer it to our application. Subsequently, within the application, we observe the channel, and the program concludes when the channel is closed.

1. Firstly, we modify the application to accept a done channel:

```golang{hl_lines=[1]}
func runApplication(done <-chan struct{}) {
	// application logic
}
```

It's a good practice to pass the channel as read-only (`<-chan`). The rule of thumb is that whoever creates the channel gets to write to it or close it and others should read from it. Here, the application receives a channel so it should only be able to read from the channel. 

2. Then we watch the channel and terminate the application when the channel gets closed.

```golang {hl_lines=["3-7"]}
for {
	select {
	case _, ok := <-done:
		if !ok {
			// exit from application when the channel is closed
			return 
		}
	default:
		log.Println("doing stuff...")
		time.Sleep(time.Second)
	}
}
```

Reads from a closed channel will result in `zero-value, false` therefore `ok` will be `false` when the channel gets closed so we can safely terminate the application.

3. Now in `main()` we create the channel and pass it to the application. Then when a signal is received, we close the channel.

```golang {hl_lines=["2-4", 10]}
func main() {
	// create the channel and pass it to application
	done := make(chan struct{})
	go runApplication(done)

	sig := make(chan os.Signal, 1)
	signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
	s := <-sig
	log.Println("received signal:", s)
	close(done) // we close the channel when a signal is received
}
```

### Putting it all together

```golang {hl_lines=["15-17", 23, "31-35"],linenos=inline,anchorlinenos=true,lineanchors="done-channel-final"}
package main

import (
	"log"
	"os"
	"time"
)

func init() {
	log.SetFlags(0)
	log.SetOutput(os.Stdout)
}

func main() {
	// create the channel and pass it to application
	done := make(chan struct{})
	go runApplication(done)

	sig := make(chan os.Signal, 1)
	signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
	s := <-sig
	log.Println("received signal:", s)
	close(done) // we close the channel when a signal is received
}

func runApplication(done <-chan struct{}) {
	defer cleanupApplication()

	for {
		select {
		case _, ok := <-done:
			if !ok {
				// exit from application when the channel is closed
				return 
			}
		default:
			log.Println("doing stuff...")
			time.Sleep(time.Second)
		}
	}
}

func cleanupApplication() {
	time.Sleep(5 * time.Second)
	log.Println("application done.")
}
```

Our code still has a major problem, we are not waiting for application cleanup. Here is the output of the code above:
```bash
$ go run main.go   
doing stuff...
doing stuff...
^Creceived signal: interrupt
```

We are not seeing `application done.` in the `cleanupApplication()`. `main()` should wait for our application to finish its work and then terminate. This is done with the help of `WaitGroups`.

## WaitGroups
A wait group is a sophisticated mechanism for coordinating goroutines. It allows you to specify the number of goroutines that need to be finished before proceeding with further execution. To utilize WaitGroup, we first create a `sync.WaitGroup` instance `wg`. Then, for each goroutine we launch, we call the `wg.Add(1)` method to increment the counter of pending goroutines. Once all goroutines have been started, we call the `wg.Wait()` method in the `main()` function. This blocks the main function until all goroutines have finished executing. To signal the completion of a goroutine, we call the `wg.Done()` method within the goroutine's code. This decrements the counter, indicating that one goroutine has finished its task.

1. We begin with `main()`
```golang {hl_lines=["5-11", "19-20"],linenos=inline,anchorlinenos=true,lineanchors="waitgroup-main"}
func main() {
	// create the channel and pass it to application
	done := make(chan struct{})
	
	// create a sync.WaitGroup and add 1 to it for each go-routine
	wg := sync.WaitGroup{}	
	wg.Add(1)	
	go func() {
		defer wg.Done()
		runApplication(done)
	}()

	sig := make(chan os.Signal, 1)
	signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
	s := <-sig
	log.Println("received signal:", s)
	close(done) // we close the channel when a signal is received
	
	// we wait for all the go-routines
	wg.Wait()
}

```

Notice that we called `wg.Add(1)` outside the go-routine. The reason is that the time that the go-routine will start is not in our hands and it's up to the go runtime and the operating system. It's good that we call `wg.Add(1)` ASAP. We are also deferring the call to `wg.Done()` avoid a dead-lock if our application panics for some reason.

Our application doesn't need any further modifications so let's run the code and observe the output.
```bash {hl_lines=[5]}
$ go run main.go          
doing stuff...
doing stuff...
^Creceived signal: interrupt
application done.
```

We can see that `application done.` is being logged. This means that our application finished its work before termination.


# Replacing done channel
# Replacing wait groups
# Writing tests
# Call to action (ask readers to go and make their programs graceful)