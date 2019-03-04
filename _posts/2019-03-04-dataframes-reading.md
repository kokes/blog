---
title: "These DataFrames are made for reading"
date: 2019-03-04T07:03:53+02:00
draft: false
---

I don't really do any machine learning, so I only have one book on the topic, Sebastian Raschka's excellent [Python Machine Learning](https://sebastianraschka.com/books.html). I was browsing it the other day and I noticed this pattern that I've come across many times and has caused me much grievance.

In the sentiment analysis chapter, Sebastian guides us to untar a dataset with fifty thousand files and proceed to read them into a DataFrame. The suggested way is like this:


```python
import os
import pandas as pd

basepath = 'aclImdb'
df = pd.DataFrame()
for s in ('test', 'train'):
    for l in ('pos', 'neg'):
        path = os.path.join(basepath, s, l)
        for file in sorted(os.listdir(path)):
            with open(os.path.join(path, file), 'r', encoding='utf-8') as infile:
                 txt = infile.read()
            df = df.append([[txt, labels[l]]],
                            ignore_index=True)
```


The first edition suggests this takes up to 10 minutes (the printed console output shows around 700 seconds), the new edition cites about 3.5 minutes. Still, that's ages just to read files. What's the bottleneck here? Well, pandas is. Though, really, it's not pandas' fault, it's just that, by design, pandas DataFrames shouldn't be mutated fifty thousand times, not in an append fashion.

In many scenarios, this including, we are better off loading data into a "plain" container (like a list of tuples) and then creating a DataFrame from that. Here is a quick example of doing so. I'm using `os.walk`, though I'd usually use `glob` or `iglob`, it doesn't really matter much here. Also, while the book uses [pyprind](https://github.com/rasbt/pyprind) to signal progress, I suggest you use the excellent and more popular [tqdm](https://github.com/tqdm/tqdm).


```python
import os

import pandas as pd
from tqdm import tqdm

ds = []
for directory, _ , files in os.walk('aclImdb'):
    if directory not in ('aclImdb/test/pos', 'aclImdb/test/neg', 'aclImdb/train/pos', 'aclImdb/train/neg'): continue
    
    sent = 'pos' if directory.endswith('/pos') else 'neg'

    for fn in tqdm(files):
        with open(os.path.join(directory, fn), 'rt') as f:
            ds.append((f.read(), sent))
            
df = pd.DataFrame(ds)
```

This implementation is now wholly dependent on your drive's ability to read files, it does around 2800 iterations per second on my VPS, so the whole thing is done in 20 seconds. Not bad. 

Can we do better?

Oh why not, we can. Not in terms of CPU, but in terms of memory usage. There is no need to read the data into memory, we can offload them into a CSV directly.


```python
import csv
import os
from tqdm import tqdm

with open('output.csv', 'wt') as fw:
    cw = csv.writer(fw)
    cw.writerow(['text', 'sentiment'])
    for directory, _ , files in os.walk('aclImdb'):
        if directory not in ('aclImdb/test/pos', 'aclImdb/test/neg', 'aclImdb/train/pos', 'aclImdb/train/neg'): continue

        sent = 'pos' if directory.endswith('/pos') else 'neg'

        for fn in tqdm(files):
            with open(os.path.join(directory, fn), 'rt') as f:
                cw.writerow((f.read(), sent))
```


This way we get minimal memory overhead, you can run this with a few dozen megabytes of RAM. But there is still one thing missing. The book instructs us to shuffle the data in question - which is fairly difficult to do properly in a stream. But we can do something else - we can shuffle the list of files, it's only 50 thousand strings, and then read files from this shuffled list.


```python
from random import shuffle
from glob import glob
from itertools import chain

files = list(chain.from_iterable(glob(f'aclImdb/{ds}/{sent}/*.txt')
                                 for ds in ('test', 'train') for sent in ('pos', 'neg')))


shuffle(files)
```

I would normally be more lazy - in terms of listing (`iglob` instead of `glob`) and wouldn't use `list`, but here we need to materialise the list of files in order to shuffle it.

So here you go, reading files as fast your drive allows you to do so and with minimal overhead, both in terms of CPU and memory.
