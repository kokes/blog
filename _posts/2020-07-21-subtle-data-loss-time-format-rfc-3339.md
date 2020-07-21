---
title: "Subtle data loss in Go: printed timestamps are not ordered"
date: 2020-07-21T07:03:53+02:00
---

When I wrote about [high performance data loss](https://kokes.github.io/blog/2020/06/12/high-performance-data-loss.html), many of the scenarios were fairly obvious. You don't adhere to a standard, you get bitten. Here's another example of non-adherance, which can bite you in rather unexpected ways.

Keeping time in computers is extremely tricky, it's riddled with exceptions, changes, corner cases, it's just hard to implement correctly. But when done right, you are rewarded - e.g. when you print time as strings, especially if it's ISO-8601 or RFC3339, you get a really nice benefit, [these strings are ordered](https://tools.ietf.org/html/rfc3339#section-5.1). That way, you know that `2020-07-21` comes after `2020-02-13` or `1970-01-05` and you know this without parsing these strings.

Why is that important? Two reasons.

1. Parsing time and date is expensive. It might not seem that way, but try parsing thousands or millions of dates and you'll see it.
2. In some scenarios parsing is not suitable - e.g. file listing - many filesystems or object stores give you files in sorted order and if you prepend each filename with a date, you can ensure they are sorted by time as well.

Now to our problem at hand. If you use Go and its `time` package, it will deviate from the standard in the subtlest of ways: it truncates trailing zeroes in mili/micro/nanoseconds, so `2020-07-21T08:21:00.629280Z` becomes `2020-07-21T08:21:00.62928Z`. Doesn't seem like a big deal, but it breaks the ordering guarantee we had originally.

This now means that if you prepend your filenames or if you have objects stored as `${time}.csv`, you can't just invoke a listing function and process data in sorted order, you need to load _all_ you filenames, parse the dates within, sort them, and then process the data. Or you can... lose data since you're relying on them being sorted and they aren't.

Here's a bit of code to reproduce the issue.

```go
package main

import (
	"fmt"
	"log"
	"sort"
	"time"
)

func main() {
	if err := run(); err != nil {
		log.Fatal(err)
	}
}

func run() error {
	n := 100

	times := make([]string, 0, n)
	// now := time.Now().UTC()
	now, err := time.Parse("2006-01-02", "2020-07-21")
	if err != nil {
		return err
	}

	for j := 0; j < n; j++ {
		now = now.Add(time.Microsecond)
		newTime := now.Format(time.RFC3339Nano)
		times = append(times, newTime)
	}
	sort.Strings(times)

	for j := 1; j < len(times); j++ {
		t1, t2 := times[j-1], times[j]
		t1p, err := time.Parse(time.RFC3339, t1)
		if err != nil {
			return err
		}
		t2p, err := time.Parse(time.RFC3339, t2)
		if err != nil {
			return err
		}

		if t1p.UnixNano() > t2p.UnixNano() {
			fmt.Printf("apparently, %s precedes %s\n", t1, t2)
		}

	}

	return nil
}
```

This yields

```
apparently, 2020-07-21T00:00:00.000019Z precedes 2020-07-21T00:00:00.00001Z
apparently, 2020-07-21T00:00:00.000029Z precedes 2020-07-21T00:00:00.00002Z
apparently, 2020-07-21T00:00:00.000039Z precedes 2020-07-21T00:00:00.00003Z
apparently, 2020-07-21T00:00:00.000049Z precedes 2020-07-21T00:00:00.00004Z
apparently, 2020-07-21T00:00:00.000059Z precedes 2020-07-21T00:00:00.00005Z
apparently, 2020-07-21T00:00:00.000069Z precedes 2020-07-21T00:00:00.00006Z
apparently, 2020-07-21T00:00:00.000079Z precedes 2020-07-21T00:00:00.00007Z
apparently, 2020-07-21T00:00:00.000089Z precedes 2020-07-21T00:00:00.00008Z
apparently, 2020-07-21T00:00:00.000099Z precedes 2020-07-21T00:00:00.00009Z
```

If you print the affected portion of the sorted strings slice, you get this series

```
2020-07-21T00:00:00.000015Z
2020-07-21T00:00:00.000016Z
2020-07-21T00:00:00.000017Z
2020-07-21T00:00:00.000018Z
2020-07-21T00:00:00.000019Z
2020-07-21T00:00:00.00001Z    <- this precedes all the timestamps above
2020-07-21T00:00:00.000021Z
2020-07-21T00:00:00.000022Z
2020-07-21T00:00:00.000023Z
2020-07-21T00:00:00.000024Z
```

At this point you can't rely on these formatted strings being sorted, that is unless you use your own formatting string. `RFC3339Nano` in Go cannot be relied on, not in terms of its ordering guarantees.

This is [a known issue](https://github.com/golang/go/issues?q=is%3Aissue+RFC3339+is%3Aclosed), among many other ones.
