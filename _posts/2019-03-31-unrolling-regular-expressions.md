---
title: "Unrolling regular expressions"
date: 2019-03-31T07:03:53+02:00
draft: false
---

I fondly remember those days when I first learned regular expression, I felt [quite invincible](https://xkcd.com/208/). Over time, I used them here or there, for various string validation or extraction. But once I moved to more performance-critical development areas, I quickly noticed the performance implications of using regular expressions. I'll walk you through a few examples of this.

_Disclaimer: This post contains a number of (micro)benchmarks. These should be taken with a grain of salt. Always benchmark things yourself, because these results only reflect my (contrived) examples, my runtime, language implementation etc._

Similarly to my post on [big O and performance](https://kokes.github.io/blog/2018/10/09/big-o-performance.html), the key message here is that implementation details can influence your runtime performance more than a language choice or other common performance tips.

One last thing before we get started - only experiment with things in this post if the (potential) performance benefits are worth it. While I used regexps to validate config files, which were loaded once per run, I didn't need any of this. But then I started parsing larger amounts of data and using validation everywhere, that's when all sorts of performance boosts were really helpful.

**tl;dr: You can gain [100X or more](#benchmarks-summary) by replacing regular expressions with funcionally equivalent explicit lookups. This has the cost of being less explicit, so only use it in performance critical pieces of code.**

## Setup

Let's take a dataset that doesn't fit into my CPU caches, I chose this [list of tweets](https://github.com/fivethirtyeight/russian-troll-tweets/) and I'll scan the file for matches of a few strings.

_Side note: I'm intentionally reading the file line by line, even though it's not the best thing for performance. For more info, check out this great and well known e-mail explaining why [GNU grep is so fast](https://lists.freebsd.org/pipermail/freebsd-current/2010-August/019310.html)._

Let's download the data and read the whole file into a byte buffer, so that we can eliminate disk I/O from our benchmarks. I'll be using [Go](https://golang.org/), we'll talk about implementation details later.

```go
package matching

import (
	"bufio"
	"bytes"
	"io"
	"io/ioutil"
	"log"
	"net/http"
	"os"
	"regexp"
	"testing"
)

var dt []byte

func init() {
	fn := "tweets.csv"
	if _, err := os.Stat(fn); os.IsNotExist(err) {
		url := "https://raw.githubusercontent.com/fivethirtyeight/russian-troll-tweets/master/IRAhandle_tweets_1.csv"
		h, err := http.Get(url)
		if err != nil {
			log.Fatal(err)
		}
		f, err := os.Create(fn)
		if err != nil {
			log.Fatal(err)
		}
		io.Copy(f, h.Body)
		h.Body.Close()
		f.Close()
	}

	var err error
	dt, err = ioutil.ReadFile(fn)
	if err != nil {
		log.Fatal(err)
	}
}
```

## Baseline

Let's set a baseline - how fast can we scan lines in a byte buffer? This should be quite performant, because lines in this file are fairly consistent, so the underlying buffer (the scanning buffer, not the initial one) doesn't need much resizing.

_Side note: It's not as performant as it could be, we could do away without the `io.Reader` abstraction and [gain a bit](https://kokes.github.io/blog/2019/03/19/deserialising-ints-from-bytes.html), we could also leverage SIMD to detect newlines in parallel. But neither of this really affects the point of this post._


```go
func BenchmarkScanner(b *testing.B) {
	for j := 0; j < b.N; j++ {
		sc := bufio.NewScanner(bytes.NewReader(dt))

		for sc.Scan() {
		}
		if sc.Err() != nil {
			b.Fatal(sc.Err())
		}
	}
	b.SetBytes(int64(len(dt)))
}
```

We're a bit north of 3 GB/s, good.

```
$ go test -benchmem -bench .
BenchmarkScanner-4                          	      50	  28179908 ns/op	3348.90 MB/s	    4144 B/op	       2 allocs/op
```

## Looking for matches

Let's now try a simple scan through each byte array containing a line from our CSV. This is a simple proposition, usually implemented using the [Boyer-Moore algorithm](https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore_string-search_algorithm).

```go
func BenchmarkContains(b *testing.B) {
	for j := 0; j < b.N; j++ {
		sc := bufio.NewScanner(bytes.NewReader(dt))

		q := []byte("CNN")
		matches := 0
		for sc.Scan() {
			match := bytes.Contains(sc.Bytes(), q)
			if match {
				matches++
			}
		}
		if sc.Err() != nil {
			log.Fatal(sc.Err())
		}
	}
	b.SetBytes(int64(len(dt)))
}
```

This cuts a third from our trivial scan above, but 2 GB/s is still pretty respectable. Notice that this algorithm doesn't allocate anything on top of the scanner.

```
BenchmarkContains-4                         	      30	  47060866 ns/op	2005.31 MB/s	    4144 B/op	       2 allocs/op
```

Let's now try regular expressions. We'll go ahead a bit and match two strings, so that the regular expression is not optimised away.

_Side note: always compile your regular expressions, no matter the language. You shouldn't set up your regular expressions in loops, not if they are constant for all loop iterations._

```go
func BenchmarkRegexp(b *testing.B) {
	for j := 0; j < b.N; j++ {
		sc := bufio.NewScanner(bytes.NewReader(dt))

		q := regexp.MustCompile("(cnn|CNN)")
		matches := 0
		for sc.Scan() {
			match := q.Match(sc.Bytes())
			if match {
				matches++
			}
		}
		if sc.Err() != nil {
			log.Fatal(sc.Err())
		}
	}
	b.SetBytes(int64(len(dt)))
}
```

The result surprised me, we're getting less than 1% of our prior implementation, though we're doing something a little different. So let's compare apples to apples.

```
BenchmarkRegexp-4                           	       1	6577523534 ns/op	  14.35 MB/s	   44112 B/op	      29 allocs/op
```

We'll now replicate this regexp match by looking up the two strings, `cnn` and `CNN` explicitly.

_Side note: You might be tempted to save those two `bytes.Contains` calls to two variables and then OR them. While a competent compiler might produce the same optimised binary code as the snippet below, it might not and in that case, you'd be needlessly computing both scans. But if these scans are tested in this if statement as an OR condition, if the first one results in a true outcome, the second one isn't run._

```go
func BenchmarkContainsCaseInsensitiveOr(b *testing.B) {
	for j := 0; j < b.N; j++ {
		sc := bufio.NewScanner(bytes.NewReader(dt))

		q1 := []byte("cnn")
		q2 := []byte("CNN")
		matches := 0
		for sc.Scan() {
			if bytes.Contains(sc.Bytes(), q1) || bytes.Contains(sc.Bytes(), q2) {
				matches++
			}
		}
		if sc.Err() != nil {
			log.Fatal(sc.Err())
		}
	}
	b.SetBytes(int64(len(dt)))
}
```

In any case, these matches are so rare that both branches will be called anyway, so we can expect performance of about 50% of the single string matching code. Sure enough, we get 1 GB/s.

```
BenchmarkContainsCaseInsensitiveOr-4        	      20	  86961289 ns/op	1085.22 MB/s	    4144 B/op	       2 allocs/op
```

Let's now say we're just gonna do a case insensitive match. There are a few ways of doing that. Let's first do a case insensitive regexp match.

```go
func BenchmarkRegexpCaseInsensitive(b *testing.B) {
	for j := 0; j < b.N; j++ {
		sc := bufio.NewScanner(bytes.NewReader(dt))

		q := regexp.MustCompile("(?i)CNN")
		matches := 0
		for sc.Scan() {
			match := q.Match(sc.Bytes())
			if match {
				matches++
			}
		}
		if sc.Err() != nil {
			log.Fatal(sc.Err())
		}
	}
	b.SetBytes(int64(len(dt)))
}
```

We're back in the MB/s land.

```
BenchmarkRegexpCaseInsensitive-4            	       1	4488722225 ns/op	  21.02 MB/s	   43056 B/op	      22 allocs/op
```

One could approach by normalising both sides - we'll just lowercase both our input and the query.

```go
func BenchmarkContainsCaseInsensitiveLower(b *testing.B) {
	for j := 0; j < b.N; j++ {
		sc := bufio.NewScanner(bytes.NewReader(dt))

		q := []byte("cnn")
		matches := 0
		for sc.Scan() {
			match := bytes.Contains(bytes.ToLower(sc.Bytes()), q)
			if match {
				matches++
			}
		}
		if sc.Err() != nil {
			log.Fatal(sc.Err())
		}
	}
	b.SetBytes(int64(len(dt)))
}
```

This doesn't get much faster than the regexp and it's 20 times slower than the explicit lookup. You can see the issue in the allocations column - this implementation allocates like crazy, each lowercased row is a copy of the original.

```
BenchmarkContainsCaseInsensitiveLower-4     	       1	1820126594 ns/op	  51.85 MB/s	97825232 B/op	  243894 allocs/op
```

We can sometimes (not always!) afford to mutate our underlying byte buffer. Let's see if that makes a difference.

```go
func asciiLower(s []byte) []byte {
	diff := byte('a') - byte('A')
	for j, el := range s {
		if el >= 'A' && el <= 'Z' {
			s[j] += diff
		}
	}
	return s
}
func BenchmarkContainsCaseInsensitiveInPlace(b *testing.B) {
	for j := 0; j < b.N; j++ {
		sc := bufio.NewScanner(bytes.NewReader(dt))

		q := []byte("cnn")
		matches := 0
		for sc.Scan() {
			lower := asciiLower(sc.Bytes())
			match := bytes.Contains(lower, q)
			if match {
				matches++
			}
		}
		if sc.Err() != nil {
			log.Fatal(sc.Err())
		}
	}
	b.SetBytes(int64(len(dt)))
}
```

This looks much better, we're now at 1/3 the speed of the explicit lookup (and again, we're not doing exactly the same thing, but close enough). Notice that we're not allocating anything anew.

```
BenchmarkContainsCaseInsensitiveInPlace-4   	       5	 288746980 ns/op	 326.83 MB/s	    4144 B/op	       2 allocs/op
```

## Prefix queries

The benchmarks above covered cases where we need to lookup a finite and known list of values, what if we know the position as well? We could be looking for a prefix or a suffix of a string. This can be very performant, because thanks to early exit algorithms, we can abandon a haystack once we've most past `len(needle)` bytes (or earlier).

Let's look for lines that start with one of two numbers.

```go
func BenchmarkRegexpPrefix(b *testing.B) {
	for j := 0; j < b.N; j++ {
		sc := bufio.NewScanner(bytes.NewReader(dt))

		q := regexp.MustCompile("^(895|165)")
		matches := 0
		for sc.Scan() {
			match := q.Match(sc.Bytes())
			if match {
				matches++
			}
		}
		if sc.Err() != nil {
			log.Fatal(sc.Err())
		}
	}
	b.SetBytes(int64(len(dt)))
}
```

Even this regexp implementation runs at a respectable speed of 1.1 GB/s.

```
BenchmarkRegexpPrefix-4                     	      20	  84453321 ns/op	1117.44 MB/s	    8558 B/op	      26 allocs/op
```

An explicit lookup would look like this.

```go
func BenchmarkPrefix(b *testing.B) {
	for j := 0; j < b.N; j++ {
		sc := bufio.NewScanner(bytes.NewReader(dt))

		q1 := []byte("895")
		q2 := []byte("165")
		matches := 0
		for sc.Scan() {
			if bytes.HasPrefix(sc.Bytes(), q1) || bytes.HasPrefix(sc.Bytes(), q2) {
				matches++
			}
		}
		if sc.Err() != nil {
			log.Fatal(sc.Err())
		}
	}
	b.SetBytes(int64(len(dt)))
}
```

Resulting in more than twice the speed, but the throughput of both of these benchmarks is misleading, because the lookup code doesn't actually scan that much data, it abandons each byte slice way before reaching the end of it.

```
BenchmarkPrefix-4                           	      50	  35950263 ns/op	2625.07 MB/s	    4144 B/op	       2 allocs/op
```

## Benchmarks summary

Regular expressions offer immense flexibility and richness and are often the best tool to express complex parsing logic. However, in performance critical pieces of code, they can often be a bottleneck.

```
BenchmarkScanner-4                          	      50	  28179908 ns/op	3348.90 MB/s	    4144 B/op	       2 allocs/op
BenchmarkContains-4                         	      30	  47060866 ns/op	2005.31 MB/s	    4144 B/op	       2 allocs/op
BenchmarkRegexp-4                           	       1	6577523534 ns/op	  14.35 MB/s	   44112 B/op	      29 allocs/op
BenchmarkContainsCaseInsensitiveOr-4        	      20	  86961289 ns/op	1085.22 MB/s	    4144 B/op	       2 allocs/op
BenchmarkRegexpCaseInsensitive-4            	       1	4488722225 ns/op	  21.02 MB/s	   43056 B/op	      22 allocs/op
BenchmarkContainsCaseInsensitiveLower-4     	       1	1820126594 ns/op	  51.85 MB/s	97825232 B/op	  243894 allocs/op
BenchmarkContainsCaseInsensitiveInPlace-4   	       5	 288746980 ns/op	 326.83 MB/s	    4144 B/op	       2 allocs/op
BenchmarkRegexpPrefix-4                     	      20	  84453321 ns/op	1117.44 MB/s	    8558 B/op	      26 allocs/op
BenchmarkPrefix-4                           	      50	  35950263 ns/op	2625.07 MB/s	    4144 B/op	       2 allocs/op
```

## Unrolling further

I once worked with a piece of code that did hundreds of regexp lookups in strings of data. The regexps, however, looked like `industr(y|ies)` or `Spider\-?Man`. I realised that these could be easily unrolled into exact matches of a handful of expressions each. After doing this manually for a while, I started googling around and found this wonderful library, [exrex](https://github.com/asciimoo/exrex). You feed it a regular expression and it generates all the variations that would be matched by this regexp.

In no time, I converted a couple dozen regular expressions into low hundreds of exact matches (in some cases the set of possible matches was infinite, so I would occasionally fall back to regular expressions). At this point, we couldn't leverage techniques mentioned above, not in this exact form. Once you have a large number of needles, you need to convert them into specialised data structures that allow for fast matchings - e.g. [Aho-Corasick](https://en.wikipedia.org/wiki/Aho%E2%80%93Corasick_algorithm).

This allowed us to go from dozens of hours (!) of runtime to 10 minutes (!!) instead.

## Implementation details

As I noted above, this all depends largely on implementation of your language runtime, compiler, and regular expression engine. Ross Cox of the Go team has [written on this topic extensively](https://swtch.com/~rsc/regexp/), as did [Andrew Gallant](https://blog.burntsushi.net/ripgrep/#regex-engine), the author of Rust's regular expression engine and the wonderful tool, one you should all use, [ripgrep](https://github.com/BurntSushi/ripgrep). Andrew's article also talks about what I called unrolling, he refers to it as _literal optimizations_.
