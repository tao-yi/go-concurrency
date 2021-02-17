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
