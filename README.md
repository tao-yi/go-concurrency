## Generator: function that returns a channel

```go
func boring(msg string) <-chan string {
	c := make(chan string)
	go func() {
		for i := 0; ; i++ {
			c <- fmt.Sprintf("%s %d", msg, i)
			time.Sleep(time.Duration(rand.Intn(1e3)) * time.Millisecond)
		}
	}()
	return c
}
```

### Multiplexing

Synchronization causes blocking

```go
func main() {
	joe := boring("Joe")
	ann := boring("Ann")
	for i := 0; i < 5; i++ {
    fmt.Println(<-joe)
    // this is blocked
		fmt.Println(<-ann)
	}
	fmt.Println("You're boring, I'm leaving")
}
```

We can instead use a fan-in function to let whosoever is ready to talk

```go
func main() {
	c := fanIn(boring("Joe"), boring("Ann"))
	for i := 0; i < 10; i++ {
		fmt.Println(<-c)
	}
	fmt.Println("You're boring, I'm leaving")
}

func fanIn(input1, input2 <-chan string) <-chan string {
	c := make(chan string)
	go func() { for {	c <- <-input1 } }()
	go func() { for {	c <- <-input2 } }()
	return c
}
```

### Restoring sequencing

Send a channel on a channel, making goroutine wait its turn.

Receives all messages, then enable them again by sending on a private channel.

First we define a message type that contains a channel for the reply.

```go
type Message struct {
  str string
  wait chan bool
}
```

### Select

The select statement provides another way to handle multiple channels.

It's like a switch, but each case is a communication:

- All channels are evaluated
- Selection blocks until one communication can proceed, which then does
- If multiple can proceed, select chooses pseudo-randomly
- A default claus, if present, executes immediate if no channel is ready

```go
select {
case v1 := <-c1:
  fmt.Printf("received %v from c1\n", v1)
case v2 := <-c2:
  fmt.Printf("received %v from c2\n", v2)
case c3 := 23:
  fmt.Printf("sent %v to c3\n", 23)
default:
  fmt.Printf("no one was ready to communicate\n")
}
```

Reward fanIn function, only one goroutine is needed:

```go
func fanInV2(input1, input2 <-chan string) <-chan string {
	c := make(chan string)
	go func() {
		for {
			select {
			case s := <-input1:
				c <- s
			case s := <-input2:
				c <- s
			}
		}
  }()
  return c
}
```

### Timeout using select

the `time.After` returns a channel that blocks for the specified durating

```go
func main() {
	c := boring("Joe")
	for {
		select {
		case s := <-c:
			fmt.Println(s)
		case <-time.After(1 * time.Second):
			fmt.Println("You're too slow.")
			return
		}
	}
}
```

time out the entire conversation

```go
func main() {
  c := boring("Joe")
  timeout := time.After(5 * time.second)
	for {
		select {
		case s := <-c:
			fmt.Println(s)
		case <-timeout:
			fmt.Println("You talk too much.")
			return
		}
	}
}
```

---

### It's easy to go, but how to stop?

Long-lived programs need to cleanup.

The core is Go's select statement: like a switch, but the decision is made based on the ability to communicate.

```go
select {
case xc <- x:
	// send x on xc
case y := <-yc:
	// received y from yc
}
```

### Structure: for-select loop

loop runs in its own goroutine

`select` lets `loop` avoid blocking indefinitely in any one state

```go
func (s *sub) loop() {
	// ... declare mutable state
	for {
		// ... set up channels for cases
		select {
		case <-c1:
			// ... read/write state ...
		case c2 <-x:
			// ... read/write state ...
		case y := <-c3:
			// ... read/write state ...
		}
	}
}
```

### Select and nil channels

Sends and receivs on nil channels block.

Select never selects a blocking case.

```go
func main() {
	a, b := make(chan string), make(chan string)
	go func() {
		a <- "a"
	}()
	go func() {
		b <- "b"
	}()
	if rand.Intn(2) == 0 {
		a = nil
		fmt.Println("nil a")
	} else {
		b = nil
		fmt.Println("nil b")
	}
	select {
	case s := <-a:
		fmt.Println("got", s)
	case s := <-b:
		fmt.Println("got", s)
	}
}
```

by setting to nil, turn certain cases off that you don't need

### Where channels fail

[Concurrency Patterns in Go](https://www.youtube.com/watch?v=YEKjSzIwAdA&t=1664s&ab_channel=CodingTech)

- You can create deadlocks with channels
- Channels pass around copies, which can impact performance
- Passing pointers to channels can create race condition
- What about "naturally shared" structures like caches or registries?

### Mutexs are not an optimal solution

- Mutexes are like toilets. The longer you occupy them, the longer the queue gets
- Read/write mutexes can only reduce the problem
- Using multiple mutexes will cause deadlocks sonner or later
- All-in-all not the solution we're looking for

### Atomic operations

- `sync.atomic` package
- `Store`, `Load`, `Add`, `Swap`, `CompareAndSwap`
- Mapped to thread-safe CPU instructions
- These instructions only work on integer types
- Only about 10 - 60x slower than their non-atomic counterparts
