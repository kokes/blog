---
title: "Benchmarking deserealising of ints from byte arrays"
date: 2019-03-19T07:03:53+02:00
draft: false
---

I was reading Dan Lemire's post about [uint deserialisation](https://lemire.me/blog/2019/03/18/dont-read-your-data-from-a-straw/) and wanted to give it a go myself. I ran a few benchmarks and here's what I learned.

tl;dr: Use raw bytes if you need some speed. Use `unsafe` if you need some extra speed, but beware, this may bite you.


## Prep
I first wrote a bit of setup in Go. I'm going to generate 8192 random numbers and serialise them into a byte array.

```go
package bench

import (
	"bytes"
	"encoding/binary"
	"log"
	"math/rand"
	"testing"
	"unsafe"
)

var n int
var arr []byte
var nums []uint32

func init() {
	n = 1024 * 8
	arr = make([]byte, n*4)
	nums = make([]uint32, n)
	for j := 0; j < n; j++ {
		rnd := rand.Int31()
		nums[j] = uint32(rnd)
		binary.LittleEndian.PutUint32(arr[4*j:4*(j+1)], uint32(rnd))
	}
}
```

## Benchmarks
### Reading bytes
First, I'll just read the bytes directly, since I know at which offset I can find each number. I allocate one array of uints and then deserialise one by one.

```go
func BenchmarkReadBytes(b *testing.B) {
	for it := 0; it < b.N; it++ {
		bnums := make([]uint32, n)
		for j := 0; j < n; j++ {
			bnums[j] = binary.LittleEndian.Uint32(arr[4*j : 4*(j+1)])
		}
	}
}
```

Results?

```go
BenchmarkReadBytes-4           	  100000	     21295 ns/op
```

### Preallocating our array

Next, I'll do the same thing as above, but I'll preallocate the array where I save my uints. The rationale here is that you have a stream of incoming data, but you only work with a chunk of them at a time, so you can afford to allocate the array once and reuse it.

```go
func BenchmarkReadBytesOneAlloc(b *testing.B) {
	bnums := make([]uint32, n)
	for it := 0; it < b.N; it++ {
		for j := 0; j < n; j++ {
			bnums[j] = binary.LittleEndian.Uint32(arr[4*j : 4*(j+1)])
		}
	}
}
```

Results?

```go
BenchmarkReadBytesOneAlloc-4   	  100000	     14871 ns/op
```

That's about 30% faster. Good.

### Reading bytes, not allocating anything

Now let's say we're yet again doing the same thing, but we're not saving our new uints, we're using them outright. Here I'm summing them, just so that the compiler doesn't optimise this all away.

```go
func BenchmarkReadBytesNoAlloc(b *testing.B) {
	for it := 0; it < b.N; it++ {
		var sum uint64
		for j := 0; j < n; j++ {
			sum += uint64(binary.LittleEndian.Uint32(arr[4*j : 4*(j+1)]))
		}
	}
}
```

Results?

```go
BenchmarkReadBytesNoAlloc-4    	  200000	      9353 ns/op
```

Another third off, so we're at 2X performance at this point.

### Using `io.Reader`

Go offers a popular Reader interface, which allows you to chain readers, add buffering and other high level nice things. But what's the performance penalty here?

```go
func BenchmarkReaderAll(b *testing.B) {
	for it := 0; it < b.N; it++ {
		bnums := make([]uint32, n)
		err := binary.Read(bytes.NewReader(arr), binary.LittleEndian, &bnums)
		if err != nil {
			log.Fatal(err)
		}
	}
}
```

Results?

```go
BenchmarkReaderAll-4           	   10000	    204548 ns/op
```

Similar to Dan Lemire's results, this abstraction is 10 times slower than the byte reader above.

### `io.Reader` with a twist

Here I'm using `io.Reader` again, but I'm deserialising the uints one by one while reusing a byte buffer for each number. I'm guessing that `binary.Read` above creates new buffers for each element in that slice.

```go
func BenchmarkReaderManual(b *testing.B) {
	for it := 0; it < b.N; it++ {
		bnums := make([]uint32, n)
		rd := bytes.NewReader(arr)
		buf := make([]byte, 4)
		for j := 0; j < n; j++ {
			n, err := rd.Read(buf)
			if n != 4 {
				log.Fatalf("not enough bytes read, got %d, expected 4", n)
			}
			if err != nil {
				log.Fatal(err)
			}
			bnums[j] = binary.LittleEndian.Uint32(buf)
		}
	}
}
```

Results?

```go
BenchmarkReaderManual-4        	   20000	     58058 ns/op
```

Whoa, 70% off that `io.Reader` + `binary.Read` implementation, all thanks to reusing of an intermediate buffer?

### `io.Reader` and no allocations

Similarly to the raw bytes readers above, we're now not saving these uints at all, we're just reading them one by one.

```go
func BenchmarkReaderNoAlloc(b *testing.B) {
	for it := 0; it < b.N; it++ {
		rd := bytes.NewReader(arr)
		buf := make([]byte, 4)
		var sum uint64
		for j := 0; j < n; j++ {
			n, err := rd.Read(buf)
			if n != 4 {
				log.Fatalf("not enough bytes read, %d", n)
			}
			if err != nil {
				log.Fatal(err)
			}
			sum += uint64(binary.LittleEndian.Uint32(buf))
		}
	}
}
```

Results?

```go
BenchmarkReaderNoAlloc-4       	   30000	     53272 ns/op
```

About a 10% improvement, not as great as earlier, but not too shabby either.

### Casting

When you look at it, those bytes are the exact binary representation of your uints, you're just copying stuff over and adding some metadata on top. Surely you could just tell Go "hey, these are actually uints, not bytes". So let's.


```go
func BenchmarkCast(b *testing.B) {
	for it := 0; it < b.N; it++ {
		narr := *(*[]uint32)(unsafe.Pointer(&arr))
		narr = narr[:n]
	}
}
```

Results?

```go
BenchmarkCast-4                	2000000000	         0.48 ns/op
```

Yeah, well, this is doing barely anything, so it's understandable, that you could do this 2 billion times. Per second.

**Caveat: here I'm assuming that the producer and consumer have the same architecture, so while with the solutions above, you could potentially tweak the endianness settings, this solution does not give you that luxury and you may end up reading garbage.**

## Conclusion

While there is tons of conventional wisdom about performance, it's always good to benchmark things yourself. I was quite glad to see that operating on raw bytes is quite fast and in combination with the `binary` package it's quite safe as well. And sure, there's still the lightspeed option of casting that array, but that's usually not an option, since the underlying byte array tends to change (e.g. in a buffered read from a file), so we're just asking for trouble there.

All of the results in one place:

```go
$ go test --bench .
goos: darwin
goarch: amd64
BenchmarkReadBytes-4           	  100000	     21295 ns/op
BenchmarkReadBytesOneAlloc-4   	  100000	     14871 ns/op
BenchmarkReadBytesNoAlloc-4    	  200000	      9353 ns/op
BenchmarkReaderAll-4           	   10000	    204548 ns/op
BenchmarkReaderManual-4        	   20000	     58058 ns/op
BenchmarkReaderNoAlloc-4       	   30000	     53272 ns/op
BenchmarkCast-4               2000000000	      0.48 ns/op
```