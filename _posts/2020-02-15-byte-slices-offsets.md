---
title: "Byte slices with offsets for read only slices of strings"
date: 2020-02-15T07:03:53+02:00
---

I recently needed a big slice of strings while coding something in Go. Well, big-ish, it was maybe hundreds of megabytes in raw form, but it then exploded in memory, it was close to 10x. I sort of knew the reasons - if you have a string slice, `[]string`, it's basically pointer chasing. And these are fat pointers as well. Plus it wasn't very performant as it was allocating left and right, garbage everywhere etc.

It's a common problem and one I had to deal with on many occasions. In this case, I was lucky enough that this was **read only** data - well, read only once I constructed my containers. For that very reason, I could easily employ a method as old as time - byte slices with offsets.

The idea is as follows - you have two structures - one blob of bytes and one "index" array - this index array basically says where your strings start and end. You get a few benefits:

1. One or a few allocations, depending on what you know upfront.
2. Cache locality - no need to jump around in memory.
3. Little to no serialization if you need to read/write this data.

So the basis of this container is a struct

```go
type column struct {
	data    []byte
	offsets []uint32
}
```

I'm using uint32, so the maximum addressable space is 4 gigs - enough for my use case, but your mileage may vary.

Let's preallocate stuff in a constructor, both to avoid nil pointers, but also to add a zero to our offsets - this will be fairly redundant (but cheap), but it will lead to branchless code when reading and writing.

```go
func newColumn() *column {
	return &column{
		data:    make([]byte, 0),
		offsets: []uint32{0},
	}
}
```

Then all we need is a getter and a setter.

```go
func (c *column) addValue(s string) {
	bs := []byte(s)
	c.data = append(c.data, bs...)
	newOffset := c.offsets[len(c.offsets)-1] + uint32(len(bs))
	c.offsets = append(c.offsets, newOffset)
}

func (c *column) nthValue(n int) string {
	if n+1 >= len(c.offsets) {
		panic("out of bounds")
	}

	return string(c.data[c.offsets[n]:c.offsets[n+1]])
}
```

Too bad Go does not have iterators, this would be a lovely use case for them. You could somewhat replace them by adding a `Len` attribute to our struct and then use a simple for loop. It's not quite the same in terms of ergonomics, but you get what you get.

This is no rocket science, but I wanted these snippets saved some place, together with some narrative. It did save me two thirds of memory usage after all (it can save even more if you have longer strings and your offsets "amortize" within them).


The whole thing in one piece:

```go
package main

import "fmt"

type column struct {
	data    []byte
	offsets []uint32
}

// you might want to add a capacity argument here, to preallocate
// known capacities
func newColumn() *column {
	return &column{
		data:    make([]byte, 0),
		offsets: []uint32{0},
	}
}

func (c *column) addValue(s string) {
	bs := []byte(s)
	c.data = append(c.data, bs...)
	newOffset := c.offsets[len(c.offsets)-1] + uint32(len(bs))
	c.offsets = append(c.offsets, newOffset)
}

func (c *column) nthValue(n int) string {
	if n+1 >= len(c.offsets) {
		panic("out of bounds")
	}

	return string(c.data[c.offsets[n]:c.offsets[n+1]])
}

func main() {
	col := newColumn()
	col.addValue("ahoy")
	col.addValue("reader")
	col.addValue("how are ya")

	fmt.Println(col.nthValue(0))
	fmt.Println(col.nthValue(1))
	fmt.Println(col.nthValue(2))
}
```