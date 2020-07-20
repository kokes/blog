---
title: "Performance: Using arrays instead of tiny maps in Go"
date: 2020-07-20T07:03:53+02:00
---

I wrote about performance characteristics of maps vs. slices/arrays [a while ago](https://kokes.github.io/blog/2018/10/09/big-o-performance.html) and I stumbled upon this again today, so I figured I'd describe the issue at hand with a bit more context.

I have some text data and I want to infer for each datapoint, if it's an integer, a float etc. My data types look something like this:

```go
type Dtype uint8

const (
	DtypeInvalid Dtype = iota
	DtypeNull
	DtypeString
	DtypeInt
	DtypeFloat
    DtypeBool
    // thanks to DtypeMax, I can add new types here without worrying about anything
	DtypeMax
)
```

Notice the `DtypeMax`, that's the key to our success here. But first let's generate some data

```go
data := make([]Dtype, n)
for j := 0; j < n; j++ {
    data[j] = Dtype(rand.Intn(int(DtypeMax)))
}
```

An intuitive way to count the occurence of each type would be to do something like this:

```go
tgmap := make(map[Dtype]int)
for _, el := range data {
    // work to determine type
    tgmap[el]++
}
```

This implementation worked just fine for months, but earlier today, I was wondering if the work that goes into populating the map isn't significant after all. I leveraged two factors in my rewrite:

1. I know how many types I have, so I can create an array - no manual allocation (unlike slices or maps), size known at compile time.
2. The type constants are numeric, so I can use them as indexes into this array - much cheaper than hash lookups.

```go
var tgarr [DtypeMax]int
for _, el := range data {
    // work to determine type
    tgarr[el]++
}
```

How did it go? I ran each with a million entries ten times. I didn't use Go's testing package, because I wanted to play around in `main()`, it's usually enough for small tests with big differences in performance (I don't need statistical tests here, otherwise I'd use `testing` and `benchstat`).

```
took 31.441122ms to populate a map
took 29.453898ms to populate a map
took 29.518603ms to populate a map
took 29.254208ms to populate a map
took 31.35607ms to populate a map
took 28.851067ms to populate a map
took 29.209222ms to populate a map
took 29.355206ms to populate a map
took 29.451898ms to populate a map
took 30.235153ms to populate a map
took 521.01µs to populate an array
took 512.882µs to populate an array
took 516.915µs to populate an array
took 517.194µs to populate an array
took 541.786µs to populate an array
took 534.958µs to populate an array
took 525.162µs to populate an array
took 597.569µs to populate an array
took 534.92µs to populate an array
took 532.322µs to populate an array
```

So without any work involved in the tight loop, I get a 60x speedup. Obviously this won't give you a 60x end-to-end speedup, but depending on how expensive your loop iteration is, this may be significant (it gave me 10-20%, which is nice).

Takeaway: if you have a map with few entries, known domain, and constrained range (i.e. max-min is reasonable), it may be very efficient to use a slice or an array instead, as it has much cheaper read/write.

The whole code is here:

```go
package main

import (
	"fmt"
	"log"
	"math/rand"
	"time"
)

func main() {
	if err := run(); err != nil {
		log.Fatal(err)
	}
}

type Dtype uint8

const (
	DtypeInvalid Dtype = iota
	DtypeNull
	DtypeString
	DtypeInt
	DtypeFloat
	DtypeBool
	DtypeMax
)

func run() error {
	work := func() {
		time.Sleep(time.Nanosecond * 100)
	}
	_ = work

	n := 1000_000
	loops := 10
	data := make([]Dtype, n)
	for j := 0; j < n; j++ {
		data[j] = Dtype(rand.Intn(int(DtypeMax)))
	}

	for l := 0; l < loops; l++ {
		tgmap := make(map[Dtype]int)
		t := time.Now()
		for _, el := range data {
			// work()
			tgmap[el]++
		}
		fmt.Printf("took %v to populate a map\n", time.Since(t))
	}

	for l := 0; l < loops; l++ {
		var tgarr [DtypeMax]int
		t := time.Now()
		for _, el := range data {
			// work()
			tgarr[el]++
		}
		fmt.Printf("took %v to populate an array\n", time.Since(t))
	}

	return nil
}
```
