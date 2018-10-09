---
title: "Big-O doesn't mean performance cannot differ greatly"
date: 2018-10-09T12:03:53+02:00
draft: false
---

Hash maps are the go-to data structure for many use cases where you have hashable keys. While they do offer average O(1) performance and O(n) space, it's not entirely obvious that there can be major performance differences between O(1) data structures. This is a quick note about a use case I encountered recently.

When working with hash maps, I usually deal with dozens of millions of operations per second per core. That's all well and good, but sometimes that's too slow. Can it be improved without extra hassle? Oftentimes the answer is yes, especially if the key domain is quite small.

Hash maps are brilliant in that no matter the size of the domain, they can accomodate a key-value relationship. But this comes at a cost, because each key gets hashed and this hash determines the bucket in which your data is stored (roughly, there are several algorithms to deal with hash collisions). If the values come from a limited domain, you can forgo the whole hashing bit and store the data in a more accessible manner, e.g. in a plain array. Let's look at an example.

Let's say we have an array of values, all within a [0, 255] range, and we want to know their histogram - the frequency of all its values. We know the domain is constrained (256 values), so we can preallocate an array of integers and then use the values as bucket addresses directly. We first do this with a map, then with an array.

Here's a snippet of Go code:

```
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	n := 1000 * 1000
	data := make([]uint8, n)
	for j := 0; j < n; j++ {
		data[j] = uint8(rand.Int63n(256))
	}

	t := time.Now()
	res := make(map[uint8]int)
	for _, el := range data {
		res[el] += 1
	}
	fmt.Println("map", time.Since(t))

	// ---------

	t = time.Now()
	res2 := make([]int, 256)
	for _, el := range data {
		res2[el] += 1
	}
	fmt.Println("array", time.Since(t))
}
```

We didn't sacrifice any readability (and we gained trivial serialisability), what about performance? The map example runs in 54 milisecond on my machine, the array version only takees 750 microseconds, an almost two orders of magnitude difference. The one downside? The array version has fixed memory consumption - whereas the map's memory footprint changes as the map grows and shrinks, this array stays fixed thanks to our knowledge of the value domain. (This could of course be made dynamic, but we'd lose many of the advantages it offers.)

Next, let's look at an even more constrained domain - booleans without nulls, just true and false. Let's imagine we have a million true/false values and we want to know the frequency of each of the two. There are at least four different ways of doing this:

1. Hash map
2. Array
3. Explicit variables for each
4. Bitmap

This is illustrated in the following code:

```
package main

import (
	"fmt"
	"math/bits"
	"math/rand"
	"time"
)

func main() {
	n := 1000 * 1000
	data := make([]bool, n)
	for j := 0; j < n; j++ {
		if rand.Float32() < float32(.5) {
			data[j] = true
		} else {
			data[j] = false
		}
	}

	t := time.Now()
	res := make(map[bool]int)
	for _, el := range data {
		if el {
			res[true] += 1
		} else {
			res[false] += 1
		}
	}
	fmt.Println("map", time.Since(t))

	// ---------

	t = time.Now()
	res2 := make([]int, 2)
	for _, el := range data {
		if el {
			res2[0] += 1
		} else {
			res2[1] += 1
		}
	}
	fmt.Println("array", time.Since(t))

	// ---------

	t = time.Now()
	var trueval, falseval int
	for _, el := range data {
		if el {
			trueval += 1
		} else {
			falseval += 1
		}
	}
	fmt.Println("variables", time.Since(t))

	// ---------

	ln := n / 64
	if n%64 > 0 {
		ln += 1
	}
	bdata := make([]uint64, ln)
	for j, el := range data {
		if el {
			bdata[j/64] |= 1 << uint(j%64)
		}
	}

	t = time.Now()
	trueval = 0
	for _, el := range bdata {
		trueval += bits.OnesCount64(el)
	}
	fmt.Println("bits", time.Since(t))
}
```

What's the verdict? Map: 33 miliseconds, array: 4.7 miliseconds, variables: 372 microseconds, bitmap: 16 microseconds. (Note that in the last solution, we cheated a bit by rearanging the dataset - this is a fixed cost that would need to get amortised.)

*Side note: the last result means we could assess 62 **billion** true/false statements per second per core. That's almost 10 gigabytes of data. And this is a slow implementation - it's not even using vectorised operations, which could theoretically speed this up eight times, at which point you'd probably max out your RAM's throughput.*

*Side note #2: I didn't use the `testing` package, which would give us more stable performance numbers, feel free to retest this.*

Between the three solutions, we got a ~10x speedup each time, the final solution gave us a 20x speedup over the fastest of the first three. So while we did a linear walkthrough an array, O(n) with a O(1) operation on each element, we got wildly different performance, the fastest solution being 2000 times faster than the slowest.

This goes to show that even approaches within the same class can lead to wildly different experiences.
