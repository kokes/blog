---
title: "Downloading Facebook API data in Python with no dependencies"
date: 2018-02-04T13:03:53+01:00
draft: false
---

**Note:** This post is a part of my *"hey, you don't need those stinking dependencies to do basic stuff"* series. I realise all the upsides and downsides of [NIH](https://en.wikipedia.org/wiki/Not_invented_here), I'll get to that at some point. For the time being, let's crack on.

When downloading data from Facebook in Python, I would usually use the `facebook-sdk` library, but then I ran into this issue. The Facebook API changes fairly rapidly, so in order to keep pace with forthcoming changes, I would usually use newer API versions that necessary, even when they offered less data.

So, recently, I switched to API version 2.11 and... `facebook-sdk` protested, it didn't know this API version. That's when I realised that the Python library was not the official one (there is none, as far as I could find). However, playing around in [Facebook API Explorer](https://developers.facebook.com/tools/explorer/), I discovered code snippets that replicate said API call in various languages.

Note that it offers `curl` syntax as well... and that the call is super simple, there's no OAuth or anything, it's just a simple token affair. That gave me hope that this could be trivially implemented in Python.

## First calls
You may be familiar with `requests`, but let's take a stab at this with `urllib`, Python's built-in library. Within this, we can use `urllib.request.urlopen` for opening HTTP requests and `urllib.parse.quote` to escape characters in URLs. Let's write just a few lines of code:

```python
from urllib.request import urlopen
from urllib.parse import quote

token = quote('...') # add yours here

obj = 'cnn'
url = 'https://graph.facebook.com/v2.11/{}?access_token={}'.format(obj, token)

r = urlopen(url)

r.read()
```

This should yield `b'{"name":"CNN","id":"5550296508"}'`. Excellent.

Now let's dig into things that will require some pagination handling, let's look at posts. We can extend our code above to be a bit more involved.

```python
from urllib.request import urlopen
from urllib.parse import quote
import json

token = quote('...')

def get_data(node, obj = '', fields = None):
    burl = 'https://graph.facebook.com/v2.11/{}/{}?access_token={}'.format(node, obj, token)
    url = burl if fields is None else '{}&fields={}'.format(burl, fields)
    r = urlopen(url)
    dt = json.load(r) # since this has a .read method (otherwise use json.loads)
    return dt
    
dt = get_data('cnn', 'posts', 'created_time,message,link,type')

dt['data'][:2]
```

This will yield CNN's first two posts, though note that at the end of our response object, we get something like

```
>>> dt['paging']['next']
'https://graph.facebook.com/v2.11/5550296508/posts?access_token=(...)&fields=created_time%2Cmessage%2Clink%2Ctype&limit=25&after=(...)'
```

## Pagination

We have at least two choices here. We can either recursively call our function... or we can simply iterate. Also, we won't materialise the data just yet, you'll see why.

```python
from urllib.request import urlopen
from urllib.parse import quote
import json

token = quote('...')

def get_data(node, obj = '', fields = None):
    burl = 'https://graph.facebook.com/v2.11/{}/{}?access_token={}'.format(node, obj, token)
    url = burl if fields is None else '{}&fields={}'.format(burl, fields)
    r = urlopen(url)
    dt = json.load(r)
    
    while True:
        for el in dt['data']:
            yield el

        if 'paging' in dt and 'next' in dt['paging']:
            dt = json.load(urlopen(dt['paging']['next']))
        else:
            break

dt = get_data('5550296508_10157911977211509', 'comments', 'created_time,message')
```

In this case, I wrote two things differently - I didn't return anything and I didn't ask for CNN's posts, rather for comments on one of its stories.

Turns out, we don't need to materialise everything. That's slow and wasteful, in case we don't actually need all the data. We can just return whatever is at hand and *only* when we run out, do we check what's next in the data from Facebook and retrieve that.

We're loading one post's comments, because that, unlike the list of all posts, is a fairly small dataset.

In case you're not familiar with generators - they don't return any data when first run, what they offer is a lazy mechanism that returns values only when asked. You can keep executing `next(dt)` to get new and new datapoints, or you may choose to materialise the whole thing using `list(dt)`. Or you simply loop it as `for el in dt`, there are many options at hand. And while you don't know generator's size upfront, they are indispensable for exactly that reason. They can be infinite.


## Putting it to some use

So how can we actually put this to some use? Well, let's download a bunch of posts and all the comments under them. Since our generator knows no bounds (or in this case it's all the posts ever posted on Facebook), we'll cut it off at fifteen posts, but that's just for demonstration.

```python
posts = get_data('cnn', 'posts', 'created_time,message,link,type')

with open('posts.json', 'w') as f:
    for j, post in enumerate(posts):
        if j > 15: break
        post['comments'] = list(get_data(post['id'], 'comments', 'created_time,message'))
        json.dump(post, f)
        f.write('\n')
        f.flush() # not necessary, just so that we can observe data coming in
```

This is all it takes. Again, no libraries, no nothing, just plain old Python. We managed to take 20 lines of code and use it in another 10 to get a fully functional Facebook data downloader.

## Objects and/or tabular

If you don't fancy JSON(L), you might want a set of CSVs instead. No problem, the code is almost just as simple. We will have a CSV of posts and a CSV for comments.

```python
import csv
hdp = ['created_time', 'id', 'link', 'message', 'type']
hdc = ['created_time', 'message']

with open('posts.csv', 'w') as fp, open('comments.csv', 'w') as fc:
    # write headers
    cwp = csv.writer(fp)
    cwc = csv.writer(fc)
    cwp.writerow(hdp)
    cwc.writerow(hdc)
    
    # write data
    for j, post in enumerate(posts):
        if j > 15: break
        post['comments'] = list(get_data(post['id'], 'comments', 'created_time,message'))
    
        cwp.writerow([post[k] for k in hdp])
        for c in post['comments']:
            cwc.writerow([c[k] for k in hdc])
```

I ran this and downloaded almost 10 thousand comments in no time, ready to upload them to any tool that knows CSVs.

### Wrapping up

There you go, just a couple dozen lines of code and we are ready to download data from Facebook. Zero dependencies, clear code, lazy loading, open formats.