# Learning Path

## Basic function

### Synchronous programming

Try to run this function and see what happens.

```
func main() {
	count("arremesso")
	count("cesta")
}

func count(thing string) {
	for i := 1; true; i++ {
		fmt.Println(i, thing)
		time.Sleep(time.Millisecond * 500)
	}
}
```

Since it is a synchronous function this is the expected output.

```
1 arremesso
2 arremesso
3 arremesso
...
```

It is going to keep printing `arremesso` until I kill the process with `ctrl+c`.

### Go routines

Try adding the `go` keyword in front of the first `count` call.

```
func main() {
	go count("arremesso")
	count("cesta")
}

func count(thing string) {
	for i := 1; true; i++ {
		fmt.Println(i, thing)
		time.Sleep(time.Millisecond * 500)
	}
}
```

Adding the keyword `go` in front of the first `count`call creates a go routine, which doesn't block the code. It creates a concurrency process that manages the first function call.

Now, the output would change to something like this.

```
1 cesta
1 arremesso
2 arremesso
2 cesta
3 cesta
3 arremesso
4 arremesso
...
```

It's going to keep printing both `arremesso` and `cesta` alternately.

### Go routines - Unexpected outcome

Try adding another `go` keyword in front of the second `count` call now.

```
func main() {
	go count("arremesso")
	go count("cesta")
}

func count(thing string) {
	for i := 1; true; i++ {
		fmt.Println(i, thing)
		time.Sleep(time.Millisecond * 500)
	}
}
```

This time the output is confusing.
Nothing shows up!

This behaviour happens because when running the code, it started **two go routines** but since there was nothing more happening in the main process reached the end and was finished and the program existed. In the previous example the main process was responsible of a loop, therefore when the go routine finishes processing it would be able to print the output.

```
func main() {
	go count("arremesso")
	go count("cesta")

    time.Sleep(time.Second * 2)
}

func count(thing string) {
	for i := 1; true; i++ {
		fmt.Println(i, thing)
		time.Sleep(time.Millisecond * 500)
	}
}
```

The code above creates a two seconds timeframe before the main process finishes. So it is possible to see the `go routines` outputs in the meanwhile.

### Go routines - Wait Group

A good way to deal with `go routines` and make sure that all started go routines have their process outputed while the main process is running is using a Wait Group.

The code below creates a `Wait Group` and adds 1 to the `wg` variable saying there is one go routine to be waited.
Then, after wrapping the `count` call into a immediately invoking anonymous function call it is invoked the `wg.Done()`to decrement the wg go routine counter to be waited.
After that, the `wg.Wait()` will wait until the `go routine` is finished.

```
func main() {
	var wg sync.WaitGroup
	wg.Add(1)

	go func() {
		count("arremesso")
		wg.Done()
	}()

	wg.Wait()
}

func count(thing string) {
	for i := 1; i <= 5; i++ {
		fmt.Println(i, thing)
		time.Sleep(time.Millisecond * 500)
	}

}
```

### Go routines - Channels

Channels are a way for go routines to communicate with each other.
The code below creates a channel `c` that awaits to receive a message to be printed.
Inside de `count` function it receives a channel as a parameter and sent to this channel a message.
After waiting for 500 miliseconds, the message is printed.

```
func main() {
	c := make(chan string)

	go count("arremesso", c)

	msg := <- c // msg receives whatever is being sent through the channel
	fmt.Println(msg)
}

func count(thing string, c chan string) {
	for i := 1; i <= 5; i++ {
		c <- thing // sends the message to the channel
		time.Sleep(time.Millisecond * 500)
	}
}
```

Notice that although the for loop iterates 5 times, only one `arremesso` is printed. This is because the `fmt.Println()` blocks the code until it is executed, which happens after one message transmission.

It is possible to receive all messages wrapping the `msg` receiver and the `fmt.Println()`.

```
func main() {
	c := make(chan string)

	go count("arremesso", c)

    for{
        msg := <- c // msg receives whatever is being sent through the channel
        fmt.Println(msg)
    }
}

func count(thing string, c chan string) {
	for i := 1; i <= 5; i++ {
		c <- thing // sends the message to the channel
		time.Sleep(time.Millisecond * 500)
	}
}
```

Despite iterating 5 times, the output is a error.

```
arremesso
arremesso
arremesso
arremesso
arremesso
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
        /Users/fabioarazaki/projects/go/go-by-example/concurrency/go-routines.go:14 +0xb2
exit status 2
```

This happens because de `count` function is finished after iterating 5 times, but the main process still waiting to receive `msg` on the channel. But nothing else will be send through this channel. It would wait forever, but Golang has a way to detect this problem during the runtime.

To solve this I can close the channel after finishing sending through the channel and checking if it open while "receiving".

```
func main() {
	c := make(chan string)

	go count("arremesso", c)

    for {
        msg, open := <- c // msg receives whatever is being sent through the channel. The second value passes to the receiver is a boolean which tells if the channel is open or not.
        if !open {
            break
        }
        fmt.Println(msg)
    }
}

func count(thing string, c chan string) {
	for i := 1; i <= 5; i++ {
		c <- thing // sends the message to the channel
		time.Sleep(time.Millisecond * 500)
	}
    close(c)
}
```

One important take-away is to let the sender context deal with it because it is the sender's concern to know if the channel wouldn't be use anymore.

Using a differente `for` sintaxe, it is possible to save some code.

```
func main() {
	c := make(chan string)

	go count("arremesso", c)

	for msg := range c { // while its open, keep looping
		fmt.Println(msg)
	}
}

func count(thing string, c chan string) {
	for i := 1; i <= 5; i++ {
		c <- thing // sends the message to the channel
		time.Sleep(time.Millisecond * 500)
	}
	close(c)
}
```

### Channels Buffer

The code below shows that the code is blocked when a message is sent to a channel until the message is received.

```
func main() {
	c := make(chan string)
	c <- "Hello" // sends Hello string to the channel

	msg := <-c // receives from whatever value comes from channel c
	fmt.Println(msg)
}
```

To receive the message it would be necessary to use a go routine to create another process and unblocked the `main`.
Another way to deal with this is to use `channel buffer`.

When creating a channel, passing a second argument to the `make` function call creates a buffer limited to the specify integer passed.

```
func main() {
	c := make(chan string, 2)
	c <- "Hello" // sends Hello string to the channel
	c <- "Wolrd" // since there is enough "space" in the buffer, it is possbile to send another message.

	msg := <-c // receives from whatever value comes from channel c
	fmt.Println(msg)

	msg = <-c
	fmt.Println(msg)
}
```

In the code above the buffer limit is `2`. So if I try to send another message through the `c` channel it will thrown an error and the process will get deadlock.

### Select Statement

The code below shows an interesting behaviour. Although each go routine has different time intervals, the output is ordered.
This happens because the code is blocked in the `fmt.Println()` call. So we can assume that the longer interval (2 seconds) is slowing down the faster go routine.

```
func main() {
	c1 := make(chan string)
	c2 := make(chan string)

	go func() {
		for {
			c1 <- "Every 500ms"
			time.Sleep(time.Millisecond * 500)
		}
	}()

	go func() {
		for {
			c2 <- "Every two seconds"
			time.Sleep(time.Second * 2)
		}
	}()

	for {
		fmt.Println(<-c1)
		fmt.Println(<-c2)
	}
}

```

To solve this I can use the select statement.

```
func main() {
	c1 := make(chan string)
	c2 := make(chan string)

	go func() {
		for {
			c1 <- "Every 500ms"
			time.Sleep(time.Millisecond * 500)
		}
	}()

	go func() {
		for {
			c2 <- "Every two seconds"
			time.Sleep(time.Second * 2)
		}
	}()

	for {
		select {
		case msg1 := <-c1:
			fmt.Println(msg1)
		case msg2 := <-c2:
			fmt.Println(msg2)

		}

	}
}
```
