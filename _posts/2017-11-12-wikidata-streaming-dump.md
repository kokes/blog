---
title: "Wikidata: Reading large streams with a tiny memory footprint"
date: 2017-11-12T13:03:53+01:00
draft: false
---

Wikipedia has grown to be the biggest encyclopedia on the web, so one would think that its contents could be used in various analyses. Turns out there is a lesser known dataset that is more suitable for this sort of work, yet it's still closely linked to Wikipedia - enter [Wikidata](https://www.wikidata.org).

Wikidata hosts an enormous database of *structured* data. Think of it as Wikipedia minus the article contents plus links between pages (entities). So when you have, say, [David Cameron](https://www.wikidata.org/wiki/Q192), you know he's human (despite his robotic narrative at times), male, has a sibling, a spouste, is a politician etc. All these links can be further followed, so you can query Wikidata to give you all politicians born in March.

What's great about Wikidata is that they offer all their data in frequent dumps, [available to us all](https://dumps.wikimedia.org/wikidatawiki/entities/). I have long wanted to craw these datasets and extract a small subset of them for further use. **In case you want just a tiny subset, say a list of French presidents, I'd recommend Wikidata's [query tool][1] instead.** It will save you a lot of bandwidth, time, and worries.

With that out of the way, let's get cracking.

[1]: https://query.wikidata.org/#SELECT%20%3Finstance_of%20%3Finstance_ofLabel%20WHERE%20%7B%0A%20%20SERVICE%20wikibase%3Alabel%20%7B%20bd%3AserviceParam%20wikibase%3Alanguage%20%22%5BAUTO_LANGUAGE%5D%2Cen%22.%20%7D%0A%20%20%3Finstance_of%20wdt%3AP39%20wd%3AQ191954.%0A%7D%0ALIMIT%20100

If we look at the aforementioned data dump, we quickly find out it currently (November 2017) boasts 22 gigabytes *compressed*. With these sorts of file sizes, you have to get a bit creative. Luckily, Python's line reading extends beyond `open`. We can utilise the `gzip` package for that, the only difference is that we need to tell it to read it as text (by default, it reads byte streams).


```bash
$ curl -I https://dumps.wikimedia.org/wikidatawiki/entities/latest-all.json.gz
```


```python
import gzip
with gzip.open('data/latest-all.json.gz', 'rt') as gf:
    for ln in gf:
        pass # process_line(ln)
```

This is all well and good... but funnily enough, I don't have enough space to even download this file (sad, I know). I would normally download this file and scan it, uncompressing it on the fly, to save space, just like above. But at this point I can't even do that. So let's try something fun... let's scan it without saving it locally.

But before we do that, let's look at the file format. Usually these large dumps are [JSONL](http://jsonlines.org/) files. Given that JSON objects can be represented as a single line, so you can have a dump that looks like this:

```json
{"foo": 2, "bar": "baz"}
{"foo": 4, "bar": "baz"}
{"foo": 12, "bar": "baz"}
{"foo": -1, "bar": "baz", "bak": false}
```

This allows for extremely efficient line scans that consume very little memory, like the `gzip` example above. And given that each line is independent, it's not syntactically linked to any other line, we don't have to keep anything else in memory. Sadly, the Wikidata dump is one large JSON list, so the example above looks like this:

```json
[
{"foo": 2, "bar": "baz"},
{"foo": 4, "bar": "baz"},
{"foo": 12, "bar": "baz"},
{"foo": -1, "bar": "baz", "bak": false}
]
```

That won't be a giant deal, we can still scan this line by line, we just have to be a tad more careful.

In order to test this while developing it, I will download a megabyte of that large dump and save it locally. It will be a corrupted file, so you might experience an error or two. The reason is that we're cutting the stream at an arbitrary boundary and given that it's a JSON list, it won't be closed. Moreover, the JSON object on the last line will likely be incomplete as well. We don't particularly mind either of those errors, as we shall see later on.


```bash
$ curl -r 0-1048576 -o data/wikidata-1m.json.gz https://dumps.wikimedia.org/wikidatawiki/entities/latest-all.json.gz
```

## Reading compressed data
We'll now launch a dummy HTTP server in our directory, `python -m http.server` (I'm assuming you're running Python 3, because... you should). This will listen on `localhost:8000`, so we can now ping `localhost:8000/data/wikidata-1m.json.gz` and pretend it's that dump on Wikidata's servers. It will be our little playground.


```python
# we can replace this later
# If you named your files differently, just head over to `http://localhost:8000`
# in your web browser and find the file you're looking for (and copy its address).
url = 'http://localhost:8000/data/wikidata-1m.json.gz'
```


```python
from urllib.request import urlopen

with urlopen(url) as r:
    print(r.read(20))
```

```text
b'\x1f\x8b\x08\x00}\xcf\x01Z\x00\x03\x8b\xe6\x02\x00~\x7fC\xf8\x02\x00'
```


Right, the content is gzipped, we already know that, but how do we read it line by line. (Notice we're using `with`, so that the connection doesn't end up dangling when we don't read the file in full.)

I've never read straight from the web, at least not compressed data. I thought I'd use `gzip`, but that doesn't read streams. After a lot of fiddling with `zlib`, I randomly tried something... and it turns out the solution is really simple. The trick is to realise that the object returned by `urlopen` has a `read` method, so we can pass it to whatever that accepts a `fileobj`. And `gzip.GzipFile` does.


```python
import gzip

with urlopen(url) as r:
    with gzip.GzipFile(fileobj=r) as gf:
        print(gf.read(50))
```

```text
b'[\n{"type":"item","id":"Q26","labels":{"en-gb":{"la'
```

Excellent stuff. Now that we're getting text, we can iterate like we're used to.


```python
with urlopen(url) as r:
    with gzip.GzipFile(fileobj=r) as gf:
        for j, ln in enumerate(gf):
            print(ln[:35])
            if j == 5: break
```

    b'[\n'
    b'{"type":"item","id":"Q26","labels":'
    b'{"type":"item","id":"Q27","labels":'
    b'{"type":"item","id":"Q29","labels":'
    b'{"type":"item","id":"Q3","labels":{'
    b'{"type":"item","id":"Q23","labels":'


## Using `requests`

If you browse the documentation for `urllib`, Python's standard stack for all things HTTP, it actually [recommends you use a third party library](https://docs.python.org/3/library/urllib.request.html), `requests`. It's a fantastic piece of code, the API is really simple and joy to use.

Let's look at how this example would look with `requests`. Make sure your `requests` package is up-to-date, support for this `with` syntax has been added only recently (mid 2017).


```python
import requests

with requests.get(url, stream=True) as r:
    with gzip.GzipFile(fileobj=r.raw) as gf:
        print(gf.read(50))
```

```
b'[\n{"type":"item","id":"Q26","labels":{"en-gb":{"la'
```


You can see it's almost identical. It's just that you need to access the `raw` property to get the underlying stream of data. Otherwise it's all the same.

## Dealing with JSON lists

As noted above, we're not getting JSONL, but a plain JSON list. If you look at the syntax, it won't be much of an issue. (Don't mind the error, that's just our incomplete file.)


```python
import json
with urlopen(url) as r:
    with gzip.GzipFile(fileobj=r) as gf:
        for ln in gf:
            if ln == b'[\n' or ln == b']\n':
                continue
            if ln.endswith(b',\n'): # all but the last element
                obj = json.loads(ln[:-2])
            else:
                obj = json.loads(ln)
            # process `obj`
            print(obj['id'], end=' ')
```

    Q26 Q27 Q29 Q3 Q23 Q42 Q36 Q39 Q62 Q64 Q71 Q72 Q82 Q83 Q89 Q96 Q99 Q123 Q125 Q136 Q140 Q143 Q145 Q146 Q147 Q148 Q163 Q168 Q173 Q184 Q188 Q189 Q194 Q195 Q207 Q212 Q221 Q227 Q228 Q232 Q244 Q248 Q255 Q263 Q268 Q277 Q278 Q281 Q284 Q297 Q305 Q319 Q323 Q346 Q349 Q353 Q360 Q362 Q368 Q394 Q395 Q396 Q398 Q401 Q412 Q419 Q433 



    EOFError: Compressed file ended before the end-of-stream marker was reached


And with that, we're basically done.

## Polishing the code with iterators
We could make this neater by leveraging iterators. We'll first get something that will feed us lines of a gzipped remote file.


```python
def stream_gzipped_lines(url):
    with urlopen(url) as r:
        with gzip.GzipFile(fileobj=r) as gf:
            for ln in gf:
                yield ln
```

And then something that will take these lines and extract JSON objects.


```python
def stream_gzipped_json_list(url):
    for ln in stream_gzipped_lines(url):
        if ln == b'[\n' or ln == b']\n':
            continue
        if ln.endswith(b',\n'): # all but the last element
            yield json.loads(ln[:-2])
        else:
            yield json.loads(ln)
```

Let's test it!


```python
for obj in stream_gzipped_json_list(url):
    print(obj['id'], end=' ')
```

    Q26 Q27 Q29 Q3 Q23 Q42 Q36 Q39 Q62 Q64 Q71 Q72 Q82 Q83 Q89 Q96 Q99 Q123 Q125 Q136 Q140 Q143 Q145 Q146 Q147 Q148 Q163 Q168 Q173 Q184 Q188 Q189 Q194 Q195 Q207 Q212 Q221 Q227 Q228 Q232 Q244 Q248 Q255 Q263 Q268 Q277 Q278 Q281 Q284 Q297 Q305 Q319 Q323 Q346 Q349 Q353 Q360 Q362 Q368 Q394 Q395 Q396 Q398 Q401 Q412 Q419 Q433 



    EOFError: Compressed file ended before the end-of-stream marker was reached


Good... or is it?

## Closing the connection

We're not quite done. We now have a choice to make. Our connection within our iterator `stream_gzipped_lines` remains open beyond our loop - unless we fully read the file (which we won't in many cases). So we can do one of at least two things.

We can keep things simple and just iterate and process the data, breaking out of the loop when necessary. The two `with` statements will close all the handlers that we opened.


```python
with urlopen(url) as r:
    with gzip.GzipFile(fileobj=r) as gf:
        for ln in gf:
            break # process the line here
```

A more elaborate solution is to implement `with` for our iterators.


```python
class read_remote_gzip(object):
    def __init__(self, url):
        self.url = url

    def __enter__(self):
        self.conn = urlopen(self.url)
        self.fh = gzip.GzipFile(fileobj=self.conn)
        return self

    def __exit__(self, *exc_info):
        print('closing connections')
        self.fh.close()
        self.conn.close()

    def __iter__(self):
        return self

    def __next__(self):
        return self.fh.readline()

with read_remote_gzip(url) as r:
    for j, ln in enumerate(r):
        print(ln[:30])
        if j == 5: break
```

```text
b'[\n'
b'{"type":"item","id":"Q26","lab'
b'{"type":"item","id":"Q27","lab'
b'{"type":"item","id":"Q29","lab'
b'{"type":"item","id":"Q3","labe'
b'{"type":"item","id":"Q23","lab'
closing connections
```

## Reading straight from the dump
Let's read from the latest dump instead of our dummy local server. Let's include all the code here. We won't be using the class defined above, just to keep things simple. It's up to you, what level of abstraction you require. Notice that we're using stock Python 3, no external libraries whatsoever.


```python
import gzip, json
from urllib.request import urlopen

url = 'https://dumps.wikimedia.org/wikidatawiki/entities/latest-all.json.gz'

with urlopen(url) as r:
    with gzip.GzipFile(fileobj=r) as f:
        for j, ln in enumerate(f):
            if ln == b'[\n' or ln == b']\n':
                continue
            if ln.endswith(b',\n'): # all but the last element
                obj = json.loads(ln[:-2])
            else:
                obj = json.loads(ln)
                
            print(obj['id'], end=' ')
            if j == 5: break
```

```text
Q26 Q27 Q29 Q3 Q23 
```

And that's it. With fewer than 20 lines of code, we can read straight from a remote gzipped dump of Wikidata.

---

With that, thanks for reading!

## PS: about that memory
We can add two lines to get the process ID (PID) of our code and check in a tool of our choice, what the actually memory consumption is. Just make sure to increase the number of elements you're getting from the stream, so that the process doesn't exit before you check its memory consumption.


```python
import gzip, json
from urllib.request import urlopen
import os

print(os.getpid())

url = 'https://dumps.wikimedia.org/wikidatawiki/entities/latest-all.json.gz'

with urlopen(url) as r:
    with gzip.GzipFile(fileobj=r) as f:
        for j, ln in enumerate(f):
            if ln == b'[\n' or ln == b']\n':
                continue
            if ln.endswith(b',\n'): # all but the last element
                obj = json.loads(ln[:-2])
            else:
                obj = json.loads(ln)
                
            print(obj['id'], end=' ')
            if j == 5: break
```

    52094
    Q26 Q27 Q29 Q3 Q23 

On my system, this whole thing consumes around 17 megs of memory. Without JSON parsing, it was as low as 13 MB.

## PPS: What about unix?

We're using some simple rules here (removing commas etc.), but if all we got a stream of JSONL, we could get away with not using code at all. Unix tools and pipes to the rescue, one can do something like `curl foo.bar | zgrep "my keyword" | jq (...) > output.baz`. Make sure to check out the excellent `zcat` and `zgrep`, which give you efficient line reading of `cat` and `grep` to compressed streams.

