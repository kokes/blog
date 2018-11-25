---
title: "Merging Python iterables using sort merge join"
date: 2018-11-25T12:03:53+02:00
draft: false
---

When processing data in Python, I try to handle as much work as possible using lazy evaluation and iterators. This is very often feasible, but in some cases it's quite hard to achieve, e.g. when you have some state to handle as well. Lately, I have been faced with merging multiple streams of data on a simple condition. There are multiple algorithms for joins, but one stands out - [sort merge join](https://en.wikipedia.org/wiki/Sort-merge_join). If you can arrange for all inputs to be sorted, then it's rather simple to join these inputs in a memory and compute efficient manner.

This is so appealing that I tried to implement it in Python and eventually found out that you need nothing but the standard library. Let's see how it works.

First, let's get some data, we're gonna get a simple generator to spew random integers in a sorted fashion.

```python
import random
import itertools

def get_data():
    frac = random.random()
    j = 0
    while True:
        j += 1
        if random.random() < frac:
            continue
        yield j
```

Let's see what we got

```python
list(itertools.islice(get_data(), 10))
```




    [1, 2, 3, 6, 9, 11, 12, 13, 15, 18]



It's now trivial to use [heapq.merge](https://docs.python.org/3/library/heapq.html#heapq.merge), which takes iterables as an input (an arbitrary number of them) and returns an iterable of them, merged. It only assumes that the inputs are themselves sorted, all the advancing of individual iterables is handled by this function.

```python
import heapq
merged = heapq.merge(get_data(), get_data())
merged # it's lazy
```




    <generator object merge at 0x118947ca8>



```python
list(itertools.islice(merged, 10))
```




    [1, 1, 3, 4, 4, 5, 6, 7, 7, 8]



This is exactly what we wanted, you see that we clearly have data from both inputs as some values are repeated (and our generating function is strictly increasing).

Now for the grouping part. It's trivial again, we can use [itertools.groupby](https://docs.python.org/3/library/itertools.html#itertools.groupby), which, just like [uniq](https://www.gnu.org/software/coreutils/manual/html_node/The-uniq-command.html) in coreutils, groups sorted data. This is extremely memory efficient, because you only need to check if the identifier (in this case the value itself) of the next value is the same as the current one and if not, you "end" the current group and start another.

```python
grouped = itertools.groupby(merged)

list(itertools.islice(grouped, 10))
```




    [(9, <itertools._grouper at 0x1188f7f28>),
     (10, <itertools._grouper at 0x1188f7c88>),
     (11, <itertools._grouper at 0x1188f7f60>),
     (12, <itertools._grouper at 0x1188f7eb8>),
     (14, <itertools._grouper at 0x1188f7e10>),
     (15, <itertools._grouper at 0x1188f7f98>),
     (16, <itertools._grouper at 0x1188f7da0>),
     (17, <itertools._grouper at 0x1188f7ef0>),
     (19, <itertools._grouper at 0x1188f7d30>),
     (20, <itertools._grouper at 0x1188f7e48>)]



As you can see, `itertools.groupby` is itself lazy, because the group can be arbitrarily large, so it won't materialise the group, it will offer you a `_group`, which is just an iterable of that group's contents.

```python
group_id, group_contents = next(grouped)
list(group_contents) # materialise it
```




    [21]



To give you an end-to-end example, here are the two tools in action:

```python
s1 = get_data() # stream 1
s2 = get_data() # stream 2

for group, data in itertools.groupby(heapq.merge(s1, s2)):
    break # process `data` for a given `group`
```

That's it, it's that simple.

### Complex data

The first example was quite simple, because we only had streams of numbers, what if we had more complex data? It does get a tiny bit more involed, but it's along the same lines. Let's extend our data generating function first.

```python
import string
from uuid import uuid4

def get_dict_data():
    frac = random.random()
    j = 0
    while True:
        j += 1
        if random.random() < frac:
            continue
        yield {
            'id': j,
            'name': ''.join(random.choices(string.ascii_lowercase, k=10)),
            'project': uuid4(),
        }

next(get_dict_data())
```




    {'id': 7,
     'name': 'idpxvibroj',
     'project': UUID('cbb31e78-a897-4bc7-9166-1fb32d55cdee')}



We now have to tell `heapq.merge` what to join the streams on. Luckily there's a `key` argument, which is a function to extract the join key from merged elements. (If none is specified, like above, it compares the values directly.)

```python
merged = heapq.merge(get_dict_data(), get_dict_data(), key=lambda x: x['id'])
next(merged), next(merged), next(merged)
```




    ({'id': 1,
      'name': 'hgdiwutfbl',
      'project': UUID('52ca139c-c092-4331-9f4c-cd81fad1ac40')},
     {'id': 2,
      'name': 'oalbxvnfki',
      'project': UUID('3059944f-3b87-4d67-8a3d-695b3883cb60')},
     {'id': 2,
      'name': 'rwxprblplp',
      'project': UUID('bd2abc6f-6ffc-4bfc-b05c-9395df66e5fe')})



Cool, now onto the grouping part, the mechanics are the same here - `itertools.groupby` takes a `key` argument.

```python
grouped = itertools.groupby(merged, key=lambda x: x['id'])
next(grouped), next(grouped)
```




    ((3, <itertools._grouper at 0x118996198>),
     (4, <itertools._grouper at 0x1189c3e48>))



And again, this `itertools._grouper` lets you iterate on each group.

```python
group_id, group_contents = next(grouped)
```

```python
list(group_contents)
```




    [{'id': 5,
      'name': 'uitsncoxkw',
      'project': UUID('e10ecc13-6ef4-40bb-9f81-4807540ab770')},
     {'id': 5,
      'name': 'sjdlbxmnco',
      'project': UUID('c3ea8bcc-195c-4b2f-87c6-c356dff27daf')}]



And for a complete example:

```python
s1 = get_dict_data()
s2 = get_dict_data()
for group_id, group_contents in itertools.groupby(heapq.merge(s1, s2, key=lambda x: x['id']), key=lambda x: x['id']):
    break
```

### Identifying datasets

There is one last thing left to improve. You may need to know which stream/dataset each datapoint comes from. Because at this point, there is no way to tell, unless each emitter identifies itself in the payload. But it's fairly easy to add some information into our generators.

You probably know array and dictionary comprehensions. They are an easy way to create lists of things.

```python
[j**2 for j in range(10)]
```




    [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]



But did you know that you can create generators in the same way?

```python
(j**2 for j in range(10))
```




    <generator object <genexpr> at 0x1189be990>



We can leverage this to "decorate" our existing streams.

```python
s1 = (('stream1', j) for j in get_dict_data())
s2 = (('stream2', j) for j in get_dict_data())

for group_id, group_contents in itertools.groupby(heapq.merge(s1, s2, key=lambda x: x[1]['id']), key=lambda x: x[1]['id']):
    break
    
list(group_contents)
```




    [('stream1',
      {'id': 1,
       'name': 'xovmgnxwvg',
       'project': UUID('15e98ad8-5102-4a82-91de-84c23c7f404e')}),
     ('stream2',
      {'id': 1,
       'name': 'gztbcuesgq',
       'project': UUID('29601332-18d8-454e-8496-f31cfe1a8e87')})]



We did two things - we converted the generator from a generator of dicts into a generator of tuples - a tuple of a stream identifier and said dict. And they we slightly edited both lambdas to take this into consideration. Here we used `x[1]` instead `x` to access the dictionary, a [named tuple](https://docs.python.org/3/library/collections.html#collections.namedtuple) would probably serve us better here.

And that's it. This is all that I wanted to go over. Now you can easily and very efficiently merge sorted streams.