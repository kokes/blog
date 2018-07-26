---
title: "Streaming S3 objects in Python"
date: 2018-07-26T12:03:53+02:00
draft: false
---

Being quite fond of streaming data even if it's from a static file, I wanted to employ this on data I had on S3. I have previously streamed a lot of network-based data via Python, but S3 was a fairly new avenue for me. I thought I'd just get an object representation that would behave like a fileobj and I'd just loop it. Not quite. But not too bad either.

I googled around at first, but the various tips and tricks around the internets offered incomplete advice, but I managed to piece it together. First, I set up an S3 client and looked up an object.

```
import boto3
s3 = boto3.client('s3', aws_access_key_id='mykey', aws_secret_access_key='mysecret') # your authentication may vary
obj = s3.get_object(Bucket='my-bucket', Key='my/precious/object')
```

Now what? There's `obj['Body']` that implements the `StreamingBody` interface, but [the documentation](https://botocore.readthedocs.io/en/latest/reference/response.html) isn't terribly helpful here. You could iterate by chunks, but all we want is a buffered iterator that does this for us, right?

I tried a few tricks using the `io` package, but to no avail.

```
import io
body = obj['Body']
io.BufferedReader(body) # AttributeError: 'StreamingBody' object has no attribute 'readable'
io.TextIOWrapper(dt) # the same
```

Sad. But then, lo and behold, [codecs](https://docs.python.org/3/library/codecs.html) to the rescue.

```
import codecs
body = obj['Body']

for ln in codecs.getreader('utf-8')(body):
	process(ln)
```

I double checked via [memory_profiler](https://pypi.org/project/memory_profiler/), specifically using `mprof run` and `mprof plot` and all I got was 32 megs in memory. Sweet.

There were also a few gzipped files I needed to grep and it turned out to be a much simpler task to complete. `gzip.open` handles this sort of stuff with ease, no codecs business needed here.

```
import gzip

body = obj['Body']

with gzip.open(body, 'rt') as gf:
    for ln in gf:
    	process(ln)
```

Again, memory consumption never surpassed 35 megs, despite processing hundreds of megs of data. Cool, this seems to work just fine.

Happy streaming.