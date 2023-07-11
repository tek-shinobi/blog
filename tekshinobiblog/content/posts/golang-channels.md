---
title: "Golang Channels"
date: 2021-10-11T16:24:24+03:00
draft: false 
categories: ["golang"]
tags: ["golang", "concurrency"]
---

When dealing with conccurency problems, it is harder to reason about when moving down the stack of abstraction (machine, process, thread, hardware components, etc). Most programming languages use thread as its highest level of abstraction. Fortunately, Go builds on top of that & introduced `Goroutine`.

> “Share memory by communicating, don’t communicate by sharing memory.” 
>
> - One of Go’s mottos

Although Golang provides traditional locking mechanism in `sync` package, its philosophy prefers “share memory by communicating”. Therefore, Golang introduces `channel` as the medium for `Goroutines` to communicate with each other.
```go
//goroutine
go func() { 
	// do some work
}()

//channel
dataStream := make(chan interface{})
```
## Fork-join model

![Fork-join concurrency model that Go follows](/Go_con_model.png)

###### Fork-join concurrency model that Go follows

This concurrency model is used in Golang. At anytime; a child Goroutine can be ***forked*** to do concurrent work with its parent & then will join back ***at some point***.

Every Go program has a main **Goroutine**. The main one can be exit earlier than its children; as a result, a join point is needed to make sure children Goroutine has chance to finish.

## Channel
Channel holds the following properties:

*   Goroutine-safe (Multiple Goroutines can access to shared channel without race condition)
*   FIFO queue semantics

Channel always returns 2 values: 1 is object returned, 1 is status (`true` means valid object, `false` means no more values will be sent in this channel)
```go
intStream := make(chan int)
close(intStream)
integer, ok := <- intStream 
fmt.Printf("(%v): %v", ok, integer) // (false): 0
```
## Channel direction
```go
var dataStream <-chan interface{} //read from only channel
var dataStream chan<- interface{} // write to only channel
var dataStream chan interface{} // 2 ways
```
## Channel capacity
Default capacity of a channel is 0 (unbuffered channel). Reading from empty or writing to full channel is blocking.
```go
c := make(chan int,10) //buffered size of 10
c := make(chan int) //unbuffered channel
```
> if a buffered channel is empty and has a receiver, the buffer will be bypassed and the value will be passed **directly from the sender to the receiver**.

## Channel behavior against Goroutine
When a Goroutine read from or write to a channel, various of behaviours might happen depending on channel state.
```go
intStream := make(chan int) // 0 capacity channel

go func() {
	defer close(intStream) 
	for i:=1; i<=5; i++{
	  intStream <- i
  }
}()

//range from a channel
for integer := range intStream { // always blocked until the channel is closed
	fmt.Printf("%v ", integer)
}
```
Below table summarizes all the behaviour:

 Operations | Channel state | Result 
------------ | ---------------- | ------- 
Read | nil | Block
| | Open & not empty | Value
| | Open & empty | Block
| | Closed | [default value], `false`
| | Write only | Compile error
Write | nil | Block
| | Open & full | Block
| | Open & not full | Write value
| | Closed | `panic`
| | Receive only | Compile error
close |	nil | `panic`
| | Open & not empty | Closes channel. Subsequent reads succeed value until channel is empty, then it reads deafult value.
| | Open & empty | Closes channel. Reads produces default value.
| | Closed | `panic`
| | Receive only | Compile Error

As the behaviour is complex, we should have a way to make a robust and scalable program.

## Robust & scalable way when working with channel

Here is 1 suggestion way:

*   At most 1 Goroutine have the ownership of a channel. The channel ownership should be small to be managable.
*   Channel owner have a write-access (chan←);
*   While consumer only have read-only view (←chan).

    **Responsibilities:**
1.  Channel owners
    +   Init channel
    +   Do write to channel or pass ownership to another goroutine
    +   Close channel
    +   Encapsulate & expose channel as a reader channel
2.  Channel consumer
    +   Handle when channel is closed
    +   Handle blocking behaviour when reading from channel
```go
chanOwner := func() <-chan int { 
	resultStream := make(chan int, 5) //init
	go func() {
		defer close(resultStream) // close
		for i:=0;i<=5;i++{ 
			resultStream <- i // write
	} }()
	return resultStream // read channel returned
}

resultStream := chanOwner()
for result := range resultStream {
  fmt.Printf("Received: %d\n", result)
}
fmt.Println("Done receiving!")
```