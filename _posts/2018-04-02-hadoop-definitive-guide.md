---
title: "Reading about Hadoop (yes, get the Definitive Guide)"
date: 2018-04-02T13:03:53+01:00
draft: false
---

Right, so I didn't want to read [Hadoop: The Definitive Guide](http://shop.oreilly.com/product/0636920033448.do). I still have my JavaScript Definitive Guide, bought it back in 2005, because I was quite naive back then. Anyway, that book is a beast and I expected the Hadoop one to be just as beasty.

It started off when I was watching [Ted Malaska talk](https://www.youtube.com/results?search_query=ted+malaska) and got inspired. At this point, some focus on the high level technologies and often forget about the underlying technologies that allow for all the Sparks and HBases to exist. While I don't expect every analyst to understand how ZooKeeper works, I think it's quite handy to understand what the decisions behind these technologies were and why things are the way they are.

So I was googling around and found [Hadoop Application Architectures](http://shop.oreilly.com/product/0636920033196.do). I dug into it, but before the book even started, I read that I should have skimmed The Definite Guide beforehand. Sigh.

I was a bit worried at first, because the book is from 2015 and the technologies in this business move fast. But the fact is that Hadoop as the underlying architecture is moving a bit slower than the rest, providing a much needed stability to the ecosystem. But more importantly, the author doesn't just describe APIs and interfaces, he skillfully goes over the designs and reasons behind all the tricky parts of Hadoop.

These include differences between JBOD and RAID in terms of performance and reliability, how replication works and how it relates to data locality. What is bit rotting and how automatic checksums help with it, why you should get ECC memory. What the topologies in Hadoop are, where SPOF are, how fault tolerance is managed, especially with respect to the namenode. What the different scheduling mechanisms are and what their (dis)advantages are. How nested data is stored in Parquet. What the difference between authentication and authorisation is. The list goes on.

So if you ask whether you should read the book in 2018, I'd say definit(iv)ely (ahem). The way I approached is that I skipped quite a few parts. I (still) don't know how to write a MapReduce job in Java, but I don't expect to be asked to write one. I did follow the explanation of principles behind shuffling and other concepts that you still need to know for Spark and other more current technologies. Speaking of Spark, you'll still use many of the concepts described, be it data locality, hash partitioning, fair scheduling etc. Plus there's a whole chapter on Spark, be it 2015 Spark. Sure, it has evolved like crazy, but you'll still learn what RDDs are and why it's often faster than MapReduce.

Skip/skim Hadoop setup if you're not planning on being the DevOps guy for your Hadoop endeavours. Check what technologies beyond HDFS/YARN you'll need, odds are it won't be Pig or Crunch, so don't read too much into that either.

As for the rest of the book, it's a bit dated in places, but you should be able to spot it. It doesn't mention the brand new Erasure Coding strategy for replication in Hadoop 3, package installs in Python are through easy_install instead of pip, streaming focuses on Flume, but not Kafka, Flink, Storm (or any of the 37 other streaming platforms in the Apache ecosystem). SQL focuses on Hive, doesn't dwell too much on Impala, Presto, or Drill. The Spark bit doesn't cover DataFrames, the chapter on HBase doesn't cover secondary indexes.

Hey, you can get up to speed by reading relevant documentation or articles on these topics in a jiffy, but few people describe the underlying principles of modern big data processing as masterfully as Tom White. So go ahead and read Hadoop: Definitive Guide.
