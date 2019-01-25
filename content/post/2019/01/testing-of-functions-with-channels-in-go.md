+++
title = "Testing of functions with channels in Go"
date = "2019-01-24T19:25:04+01:00"
tags = ["go"]
+++

[Go] or [Golang] has a very nice mechanisms for dealing with concurrency: [channels] and [goroutines]. This post describes my approach about testing of functions, which are used within goroutines and consume input or provide output via channels. There is no rocket science at all, just the typical channel handling in the tests.

<!--more-->

# Sequential generator with tests

Given, you have the following generator function `generateInts`:

```golang
package main

import "fmt"

func main() {
	fmt.Printf("%v\n", generateInts())
}

func generateInts() []int {
	r := []int{}
	for i := 0; i < 10; i++ {
		r = append(r, i)
	}
	return r
}
```

Tests, generated with [gotests] would be like:

```golang
package main

import (
	"reflect"
	"testing"
)

func Test_generateInts(t *testing.T) {
	tests := []struct {
		name string
		want []int
	}{
		{
			name: "default testing case",
			want: []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9},
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := generateInts(); !reflect.DeepEqual(got, tt.want) {
				t.Errorf("generateInts() = %v, want %v", got, tt.want)
			}
		})
	}
}
```

# Concurrent generator with tests

Implementation of `generateInts` with channels would be like:

```golang
package main

import "fmt"

func main() {
	for i := range generateInts() {
		fmt.Printf("%v\n", i)
	}
}

func generateInts() <-chan int {
	rc := make(chan int)
	go func() {
		defer close(rc)

		for i := 0; i < 10; i++ {
			rc <- i
		}
	}()
	return rc
}
```

Tests, generated with [gotests] would be:

```golang
package main

import (
	"reflect"
	"testing"
)

func Test_generateInts(t *testing.T) {
	tests := []struct {
		name string
		want <-chan int
	}{
		// TODO: Add test cases.
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := generateInts(); !reflect.DeepEqual(got, tt.want) {
				t.Errorf("generateInts() = %v, want %v", got, tt.want)
			}
		})
	}
}
```

So, generated tests handle the channel like a common return value. The intention is however to compare and match the values going through the channel. One possible solution might be:

```golang
package main

import (
	"reflect"
	"testing"
)

func Test_generateInts(t *testing.T) {
	tests := []struct {
		name string
		want []int
	}{
		{
			name: "default testing case",
			want: []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9},
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			cgot := generateInts()
			var got []int

			for i := range cgot {
				got = append(got, i)
			}

			if !reflect.DeepEqual(got, tt.want) {
				t.Errorf("generateInts() = %v, want %v", got, tt.want)
			}
		})
	}
}
```

# Generic solution with helper function

It would be nice to have a helper test function for such cases. Real generic example would required generics :-) (see [FAQ] and [proposal] on this topic):

```golang
package main

import (
	"reflect"
	"testing"
)

func getValues(c <-chan GENERIC-TYPE) []GENERIC-TYPE {
	var r []GENERIC-TYPE

	for i := range c {
		r = append(r, i)
	}

	return r
}

func Test_generateInts(t *testing.T) {
	tests := []struct {
		name string
		want []int
	}{
		{
			name: "default testing case",
			want: []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9},
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := getValues(generateInts())

			if !reflect.DeepEqual(got, tt.want) {
				t.Errorf("generateInts() = %v, want %v", got, tt.want)
			}
		})
	}
}
```

As generics are not available yet, we might take the approach with empty interfaces and [type assertion] (please take it as POC code only):

```golang
package main

import (
	"reflect"
	"testing"
)

func getValues(c interface{}) []interface{} {
	var r []interface{}
	fAddInts := func(c <-chan int) {
		for i := range c {
			r = append(r, i)
		}
	}
	fAddStr := func(c <-chan string) {
		for i := range c {
			r = append(r, i)
		}
	}

	switch c.(type) {
	case <-chan int:
		fAddInts(c.(<-chan int))
	case <-chan string:
		fAddStr(c.(<-chan string))
	default:
		panic("Not supported")
	}

	return r
}

func toIntSlice(si []interface{}) []int {
	var r []int

	for _, i := range si {
		r = append(r, i.(int))
	}

	return r
}

func toStrSlice(si []interface{}) []string {
	var r []string

	for _, i := range si {
		r = append(r, i.(string))
	}

	return r
}

func Test_generateInts(t *testing.T) {
	tests := []struct {
		name string
		want []int
	}{
		{
			name: "default testing case",
			want: []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9},
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			vals := getValues(generateInts())
			got := toIntSlice(vals)

			if !reflect.DeepEqual(got, tt.want) {
				t.Errorf("generateInts() = %v, want %v", got, tt.want)
			}
		})
	}
}
```

Note, in `getValues` we can't use the `<-chan interface{}` as its completely different type then `<-chan int`. We have to match the channel with empty interface completely (`interface{}`) and make the [type assertion].


# See too

- [Testing over Golang Channels](https://www.hugopicado.com/2016/10/01/testing-over-golang-channels.html)

[Go]: https://golang.org
[Golang]: https://golang.org
[channels]: https://golang.org/doc/effective_go.html#channels
[goroutines]: https://golang.org/doc/effective_go.html#goroutines
[gotests]: https://github.com/cweill/gotests
[FAQ]: https://golang.org/doc/faq#generics
[proposal]: https://github.com/golang/go/issues/15292
[type assertion]: https://golang.org/doc/effective_go.html#interface_conversions
