---
title: "Busting APIs to win Lego Hulkbusters"
date: 2019-05-31T07:03:53+02:00
draft: false
---

Just a few weeks ago, I attended [PyData Amsterdam](https://pydata.org/amsterdam2019/), a real fun conference, and apart from a ton of interesting talks, I also got a [Lego Hulkbuster](https://shop.lego.com/en-CZ/product/The-Hulkbuster-Ultron-Edition-76105). It was thanks to a reaaally narrow win in a programming contest of sorts. And since a few conference attendees were curious as to how we solved it, here's how it went down.

### The problem

There's an endpoint, where you send a string - a combination of all ASCII lowercase letters - and it gives you back a number between 0 and 1, your goal is to get as close to 1 as possible, you don't know how the score is calculated, you just get the number. And on top of that, there's a little bit of noise, so you don't get the exact same value out for a given input.

### My first go - hill climbing

Since I knew that brute force wasn't the best thing - the search space is big and the API throughput is low - I had to come up with something a tiny bit more sophisticated. I didn't want to spend much time on the problem, so I just implemented a dead simple solution that was bound to be better than brute force - [hill climbing](https://en.wikipedia.org/wiki/Hill_climbing).

The idea is real simple - if you have a smooth and continuous function (more on that later), you can climb it like a hill in a fog - you don't necessarily see the top, but if you see a way up, you go that way. There are two variants of hill climbing, either you survey all your options at any given point and go the one with the best gain (derivatives work well for this, if you know the functions definition), or you just go the first direction that goes up. I went with the latter.

(There are a few complications if the function is not convex, smooth, or continuous, but let's assume those away for now.)

How does that work in code?

```python
import random
import time
import difflib

def request(pattern):
	'I did not know how this worked, I just called an API'
	desired = 'awvlpfgkdoiuzejthsxrqycnbm'
	
	return difflib.SequenceMatcher(None, pattern, desired).ratio()

current_pattern = string.ascii_lowercase
best_score = -1
positions = list(range(len(current_pattern)))

while True:
	new = list(current_pattern)
	i, j = random.sample(positions, 2)
	new[i], new[j] = new[j], new[i]

	new_pattern = ''.join(new)

	new_score = request(new_pattern)
	if new_score > best_score:
		best_score = new_score
		print(best_score)
		current_pattern = new_pattern
```

There were a few more bells and whistles, I logged a few things here and there (`a+` for the win) and I actually did more than just a single swap to speed things up - I decreased the number of swaps as my best score got higher and higher, to limit the size of the step, which had to be smaller in later stages.

A quick side note on the logging: I'm a fan of minimal and portable architectures. Here, I didn't use a single database of any sort, I didn't even use any third party libraries, so when I needed to run my code remotely, I just SSH'ed into a random VPS I had, `scp`ed my code there and ran it in tmux. When I wanted to collect my results, I just `rsync`ed my datafile, which was just a plain text file. As for monitoring my progress, I could just open up a new terminal and run `watch -n 1 sort -rn data.txt` and I'd get an auto updating list of my best scores, courtesy of [coreutils](https://www.gnu.org/software/coreutils/).


So why did all this fail in the end? Well, the issue was that the API wasn't as deterministic as my simulation above. There was a tiny bit of *noise* in the result, which was fine in the early stages, because you could leapfrog that. But once I got to the score of 0.85, a step in a direction towards 1 could be masked by noise and I would end up just running in circles.

Time for something a little more sophisticated.


### My second go - noise estimation

Since I didn't know what the *true* score was, just the score and some noise, I thought - easy peasy, I'll just ask for the score for a given string a thousand times, plot its histogram and boom, I've got a nice gaussian distribution where the mean is my true underlying value.

Two problems:
1. This would essentially increase my number of queries by a factor of a thousand (or at least a few hundred)
2. It didn't work at all, since the noise was a bit more random than I anticipated. Sneaky Vincent.

I abandoned this after about half an hour. It's important to know when to abandon dead end approaches.


### My third go - incorrect regression

Since I was logging all my data, I thought - let's run some models on that thing. Since I'm not really a modeller and all I remember are regression models, I thought - hey, that should be good enough.

I ran a regression on letter positions, it looked roughly like this:

```
score = intercept + a_pos + b_pos + c_pos + ... + y_pos + z_pos + error
```

The idea works like this: I assume the model does something like a [Levenshtein distance calculation](https://en.wikipedia.org/wiki/Levenshtein_distance), so if `abcd` is the true string, `abdc` is better than `badc`, because the former requires only one swap. From this insight, I should be able to model individual letter positions - if `a` is supposed to be at the very end, the model will score those strings were `a` is towards the end of the string, other things being equal (which is the regression proposition). Then, the estimated parameters will tell me where the individual letters should be positioned in order to get maximum score.

I tried simple OLS, bits of logistic regression... to no avail. Well, I had some info from the model, but not nearly enough to go on. We'll get to my failing of this approach later on.

### My fourth go - correct regression

Next, I tried running a regression on bigrams - combinations of two letters. This had one major advantage over the approach above: Even a lower quality model would give me hints as to what the *true string* looks like. Imagine getting a signal that `ab` and `bc` are similarly important in the string, but it's hard to say which one is where. Since my bigrams preserve order (I tried sorting them and it didn't work, luckily), I know that `ab` has to precede `bc`, otherwise I would have two `b`s in my string, which is a violation of the problem conditions.

Now, there was a small hiccup - I had a really uneven sample. Why is that? Since I did all that hill climbing, My sample contained the "winning" bigrams already, so for a large sample of these bigrams, I had little to no data and thus didn't have much to go on. Luckily, I realised this fairly quickly and started downloading *random* data from the API. Essentially doing something like

```python
import random

pattern = list(string.ascii_lowercase)
while True:
	random.shuffle(pattern)
	score = request(''.join(pattern))
	log(pattern, score)
```

It was a dead simple data collection that meant to give me an unbiased data sample. And boy did I get one. The regression worked wonderfully, it did have a couple hundred parameters, but since these linear models have closed form solutions, estimation was a breeze.

In no time I got the right string by combining the bigrams to get the winning *true string* (or at least I thought I had it). But now what? I was at 0.96 and not getting any higher.

Another side note: it wasn't as wonderful and rosy as I make it sound. Some bigrams were ambiguous, so I wrote a short script that would pick those that were unambiguous and filter out the rest based on the letters I had already used, thus reducing the search space quite a bit.

### My fourth go - brute force

Now the final piece in the puzzle - just blast the API to play with its noise generator. This was a simple proposition with a simple solution. Since the noise was random, given enough API calls, I was bound to get closer and closer to one... every now and then. So I wrote some some parallel code that just called the API with the same input and saved the results. Over and over. I think I had 256 threads running for an hour or so. I quickly got to three, four, five nines.

And that's basically it. I let it run for a while and ended up winning with 0.9999999845972404 - the second place having a 7 instead of my 8 in there. So it was pure luck. But since there were only two of us in the running, it was essentially a coin flip. I like those odds.

### Addendum: It could have been a bit easier

When discussing our solutions, I found out the other team used a simple regression on character positions, just like I tried and failed. My failing in getting an unbiased dataset led to my creation of bigram models... but I never got back to the simple character-level models, which, I'm told, worked beautifully. You just sort the parameters in descending order and there you go, you have your string.


### Thanks

So there it is. I used fairly simple techniques, learned a thing or two, and just had plain fun during a conference. And I won a [Hulkbuster](https://shop.lego.com/en-CZ/product/The-Hulkbuster-Ultron-Edition-76105), courtesy of [GoDataDriven](https://godatadriven.com/).
