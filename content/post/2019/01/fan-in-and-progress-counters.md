+++
title = "Fan in and progress counters pattern"
date = "2019-01-31T22:30:33+01:00"
tags = ["go"]
+++

The [pipeline blogpost] of Sameer Ajmani gives a good overview over the design pattern "fan-in" in Go. Sometimes, you might have a slightly different situation within fan-in code, where you need a different approach.

Example: you might need some progress counters, which get incremented during the fan-in operation, e.g. to display the processing progress of some operation to the user.

Let's have a look how this can be done.

<!--more-->

## Introduction

Given, you have some pipeline with fan-in pattern, which returns a slice of collected data:

```golang
package main

import (
	"fmt"
	"math/rand"
	"time"
)

// getData invokes a goroutine, which sends the
// ints between given start and end with a tiny random delay
func getData(start, end int) <-chan int {
	ret := make(chan int)
	go func() {
		for i := start; i <= end; i++ {
			ret <- i
			time.Sleep(time.Millisecond * 50 * time.Duration(rand.Intn(10)))
		}
		close(ret)
	}()
	return ret
}

// fanInData consumes the data from given channels
// and returns a slice with all merged data
func fanInData(ch1, ch2, ch3 <-chan int) []int {
	ret := []int{}
	for {
		select {
		case v, ok := <-ch1:
			if ok {
				ret = append(ret, v)
			} else {
				ch1 = nil
			}
		case v, ok := <-ch2:
			if ok {
				ret = append(ret, v)
			} else {
				ch2 = nil
			}
		case v, ok := <-ch3:
			if ok {
				ret = append(ret, v)
			} else {
				ch3 = nil
			}
		}

		if ch1 == nil && ch2 == nil && ch3 == nil {
			break
		}
	}

	return ret
}

func main() {
	nums10 := getData(1, 10)
	nums50 := getData(51, 60)
	nums80 := getData(81, 90)

	nums := fanInData(nums10, nums50, nums80)

	for _, i := range nums {
		fmt.Println(i)
	}
}
```

`getData()` is the pipeline source or producer.

`fanInData()` is the pipeline sink or consumer, it collects all data via given channels. When channels are closed, it returns a slice with collected data (the select breaking idea is based on [this](https://stackoverflow.com/a/13666733) answer in Stack Overflow).

We completely fulfill here the pattern from the [pipeline blogpost]:

> - stages close their outbound channels when all the send operations are done.
> - stages keep receiving values from inbound channels until those channels are closed.

## Concurrent function for progress display

Now we want to display the progress of the fan-in operation to the user. Let's introduce another concurrent function for displaying of live progress. This function will accept counter channels as parameters. Each send operation to the channel increases the according counter. In order to fulfill the pattern from the [pipeline blogpost] again, the goroutine exists when all channels are closed.

```golang
func printProgress(cnums10counter, cnums50counter, cnums80counter <-chan bool) {
	go func() {
		var nums10counter, nums50counter, nums80counter int

		// print newline character by leaving the progress printing routine
		defer func() {
			fmt.Printf("\n")
		}()

		for {
			select {
			case _, ok := <-cnums10counter:
				if ok {
					nums10counter++
				} else {
					cnums10counter = nil
				}
			case _, ok := <-cnums50counter:
				if ok {
					nums50counter++
				} else {
					cnums50counter = nil
				}
			case _, ok := <-cnums80counter:
				if ok {
					nums80counter++
				} else {
					cnums80counter = nil
				}
			}

			if cnums10counter == nil && cnums50counter == nil &&
				cnums80counter == nil {
				return
			}

			fmt.Printf("\rProgress: num10 - %v, num50 - %v, num80 - %v",
				nums10counter, nums50counter, nums80counter,
			)
		}
	}()
}
```

## Integration of counters to the fan-in operation

Let's describe the ideal pattern for channels nased on the [pipeline blogpost] and include the lifecycle perspective:

- channel should be created within sender function
- channel should be closed within sender function
- created channel should be returned for consumers from sender function
- the consumer function only reads the data from the channel until it gets closed by the sender

This meaningful approach isn't possible in this particular case:

- our sender function for counter channels would be `fanInData()`
- `fanInData()` blocks and returns the complete slice as a result

So, we do not have a way to return the created counter channels from `fanInData()` and to pass them to `printProgress()`.

We could create the counter channels within `printProgress()` and then pass them as parameter to the `fanInData()`: this would perfectly match to the program flow. However, this approach violates the pipeline pattern - we would create the channels within receiver and close them in the different place. This unusual way is definitely not the expected pattern and might result to the complex or even faulty situations. Just think about the error handling within a such pipeline: the common `ch := make(...); defer close(ch)` pattern isn't possible here.

If we can't solve this problem on this level, what about moving the channel lifecycle one level up to the `main()`:

```golang
func main() {
	nums10 := getData(1, 10)
	nums50 := getData(51, 60)
	nums80 := getData(81, 90)

	// create the counter channels
	cnums10 := make(chan bool)
	cnums50 := make(chan bool)
	cnums80 := make(chan bool)

	// invoke the printing routine
	printProgress(cnums10, cnums50, cnums80)

	// fan in the data and use counters
	nums := fanInData(
		nums10, nums50, nums80,
		cnums10, cnums50, cnums80,
	)

	// close the counter channels
	close(cnums10)
	close(cnums50)
	close(cnums80)

	for _, i := range nums {
		fmt.Println(i)
	}
}
```

This idea fulfills the pipeline patterns:

- the lifecycle of counter channels is controlled in the same place (`main()` function)
- the consumer `printProgress()` only reads from the channels until they get closed
- the sender `fanInData()` can be seen as a part of `main()`, as we can move its code completely to the main function. We are not doing what only because of delegation of details: we try to keep the `main()` small and simple

So, to sum up: our sender is `main()` and it controls the lifecycle and invokes the send operations (via a call of `fanInData()`)

We have to update `fanInData()` and use the counters. The entire example code below:

```golang
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func getData(start, end int) <-chan int {
	ret := make(chan int)
	go func() {
		for i := start; i <= end; i++ {
			ret <- i
			time.Sleep(time.Millisecond * 50 * time.Duration(rand.Intn(10)))
		}
		close(ret)
	}()
	return ret
}

func fanInData(
	ch1, ch2, ch3 <-chan int,
	cch1, cch2, cch3 chan<- bool,
) []int {
	ret := []int{}
	for {
		select {
		case v, ok := <-ch1:
			if ok {
				ret = append(ret, v)
				cch1 <- true
			} else {
				ch1 = nil
			}
		case v, ok := <-ch2:
			if ok {
				ret = append(ret, v)
				cch2 <- true
			} else {
				ch2 = nil
			}
		case v, ok := <-ch3:
			if ok {
				ret = append(ret, v)
				cch3 <- true
			} else {
				ch3 = nil
			}
		}

		if ch1 == nil && ch2 == nil && ch3 == nil {
			break
		}
	}

	return ret
}

func printProgress(cnums10counter, cnums50counter, cnums80counter <-chan bool) {
	go func() {
		var nums10counter, nums50counter, nums80counter int

		// print newline character when leaving the progress printing routine
		defer func() {
			fmt.Printf("\n")
		}()

		for {
			select {
			case _, ok := <-cnums10counter:
				if ok {
					nums10counter++
				} else {
					cnums10counter = nil
				}
			case _, ok := <-cnums50counter:
				if ok {
					nums50counter++
				} else {
					cnums50counter = nil
				}
			case _, ok := <-cnums80counter:
				if ok {
					nums80counter++
				} else {
					cnums80counter = nil
				}
			}

			if cnums10counter == nil && cnums50counter == nil &&
				cnums80counter == nil {
				return
			}

			fmt.Printf("\rProgress: num10 - %v, num50 - %v, num80 - %v",
				nums10counter, nums50counter, nums80counter,
			)
		}
	}()
}

func main() {
	nums10 := getData(1, 10)
	nums50 := getData(51, 60)
	nums80 := getData(81, 90)

	// create the counter channels
	cnums10 := make(chan bool)
	cnums50 := make(chan bool)
	cnums80 := make(chan bool)

	// invoke the printing routine
	printProgress(cnums10, cnums50, cnums80)

	// fan in the data and use counters
	nums := fanInData(
		nums10, nums50, nums80,
		cnums10, cnums50, cnums80,
	)

	// close the counter channels
	close(cnums10)
	close(cnums50)
	close(cnums80)

	for _, i := range nums {
		fmt.Println(i)
	}
}
```

## Error handling

If `fanInData()` would require error handling, you should first close the channels and then do the error handling:

```golang
// create the counter channels
	cnums10 := make(chan bool)
	cnums50 := make(chan bool)
	cnums80 := make(chan bool)

	// invoke the printing routine
	printProgress(cnums10, cnums50, cnums80)

	// fan in the data and use counters
	nums, err := fanInData(
		nums10, nums50, nums80,
		cnums10, cnums50, cnums80,
	)
	close(cnums10)
	close(cnums50)
	close(cnums80)

	if err != nil {
		...
	}
```

The reason for this is simple - to avoid unreleased channel resources in any case, in error situations too.

## Final words and some patterns

If you meet following prerequisites:

- the sender is a blocking function, so it's not possible to control the lifecycle of channels within it
- you could move the senders code one level up (to the function calling sender) without changes and impact

you can apply this pattern:

- create the channel one level up
- close the channel one level up, but only after the blocking call to the initial sender function
- if any error handling for the initial sender function call is required, channels should be closed prior to it


[pipeline blogpost]: https://blog.golang.org/pipelines
