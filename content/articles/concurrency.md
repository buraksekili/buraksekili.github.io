---
editPost:
    URL: "https://github.com/buraksekili/buraksekili.github.io/blob/main/content/articles/concurrency.md"
    Text: "Edit this page on " # edit text
author: "Burak Sekili"
title: "Concurrency Notes in Go"
date: "2024-01-24"
description: "Concurrency notes in Go"
tags: [
    "Go",
    "Concurrency"
]
TocOpen: true
---

## Concurrency

## Channels

### Unbuffered channel

If channel is unbuffered
```go
ch := make(chan struct{})
```

sending a data to channel will block the goroutine as the channel is nil.

```go
package main

func main() {
	ch := make(chan struct{})

	ch <- struct{}{}
}
```

Output of this program is:
```bash
$ go run main.go
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
        /Users/buraksekili/projects/concur/main.go:35 +0x30
exit status 2

```

As reading from non-nil channel blocks goroutine and since there is no other
goroutine reading from this channel exists, the program panics.

So, to simply put, we have 1 goroutine in this program and it is blocked
while sending the channel, all goroutines in this simple application stucked as blocked.
Hence, the program fails as `all goroutines are asleep - deadlock!`

So, if we have another goroutine that helps main goroutine by receiving a data from the channel,
the program won't be paniced.

```go
package main

import "fmt"

func main() {
	ch := make(chan struct{})

	go func() {
		<-ch
	}()

	ch <- struct{}{}
	fmt.Println("Done")
}
```
Output:
```bash
$ go run main.go 
Done
```

First listen channel in another goroutine, then write data to it. 
In the example above, even the goroutine that receives a data from the channel executes the statement `<-ch` 
after main goroutine sends data to channel `ch <- struct{}{}`, as long as the goroutines are running, the program won't panic.

```go
func main() {
	ch := make(chan struct{})

	go func() {
		time.Sleep(3 * time.Second)
		<-ch
	}()

	fmt.Println("hangs here for 3 secs")
	ch <- struct{}{}
	fmt.Println("done")
}
```

Main goroutine hangs while sending data to channel for 3 seconds while other goroutine was sleeping.
This won't cause deadlock, as sleeping or doing other computations like fetching data from database,
waiting a response from HTTP server do not cause goroutine to fall into blocking state from running state.

### Buffered Channel

Again, if we have the same example as above, but with buffered channels:

```go
package main

import "fmt"

func main() {
	ch := make(chan struct{}, 1)

	ch <- struct{}{}
	fmt.Println("done")
}
```
Output:
```bash
$ go run main.go
done
```
the program will not panic due to deadlock, as we specify capacity (buffer) for the channel.
Main goroutine will write data to the buffer and continues to the execution.

---

Regardless of whether the channel is buffered or unbuffered, receiving from a closed channel will NOT 
block your goroutine.

```go
package main

import "fmt"

func main() {
	ch := make(chan struct{})
	close(ch)
	v, ok := <-ch
	fmt.Printf("v: %v, ok: %v\n", v, ok)
}
```
Output:
```bash
$ go run main.go
v: {}, ok: false
```

Of course, you cannot send any value to closed channel, which will cause panic.

> The second parameter `ok` corresponds to boolean value showing that if the value
is sent to the channel before closing the channel. Hence, there was no value in the channel
before closing it, the `ok` is `false`.

```go
package main

import "fmt"

func main() {
	ch := make(chan int, 1)
	ch <- 3
	close(ch)
	v, ok := <-ch
	fmt.Printf("v: %v, ok: %v\n", v, ok)
}
```
Output:
```bash
$ go run main.go
v: 3, ok: true
```