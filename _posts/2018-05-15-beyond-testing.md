---
title: "Beyond Testing"
date: 2018-05-15T17:03:53+02:00
draft: false
---

There is a certain level of thrill when you refactor code, run your tests and it all goes through just fine. Sure, tests are no guarantee that you didn’t break anything, but it’s a good thing to have in place. Especially when they are not costly to create.

Enter distributed systems.

My familiarity with distributed systems has always been along the lines of “there are multiple computers doing what a single one would usually do”. Sure, there’s CAP, there is sharding, consensus, but those are mostly resolved issues… well, it turns out there’s much more.

There was one real eye-opener for me, Martin Kleppmann’s book [Designing Data-Intensive Applications](https://dataintensive.net/). This book is a spectacular overview of all things (distributed) data systems. Whatever can and will go wrong, is in there.

While the author does describes all the problems thoroughly and with examples, I wanted a bit more hands on approach. Then I stumbled upon [jepsen.io](https://jepsen.io). This is Kyle Kingbury’s site, he specialises in breaking databases. This involves simulating (temporary) network partitions, trying to change system clock in quick succession and all sorts of other nasties.

I have now seen probably all his [talks on YouTube](https://www.youtube.com/watch?v=tRc0O9VgzB0), it’s very detailed and entertaining. Just a word of caution - it’s fast paced and assumes you know the basic terminology of distributed systems (which is something Mr. Kleppmann’s book can help with).

The last bit of material is a talk from Strange Loop from a few years ago. In [Deterministic Simulation - Testing of Distributed Systems](https://www.youtube.com/watch?v=4fFDFbi3toc), Will Wilson talks about how they, FoundationDB, torture their database in order to understand and improve its durability. FoundationDB was recently open sourced by Apple and has gained a lot of praise by the community for its guarantees. If you watch the video, you’ll gain a bit more insight into why that’s the case. It’s incredibly interesting how they manage to deterministically simulate the environment a distributed system is in. This includes deterministic random number generators or avoiding parallel processes.

There is a lot more content out there, this is just a small sample of what I really enjoyed recently. With all this content in mind, I will always think twice before trying that “how new database”.