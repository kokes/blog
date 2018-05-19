---
title: "Sane CSV processing in Apache Spark"
date: 2018-05-19T12:03:53+02:00
draft: false
---

Coming from Python's [pandas](https://pandas.pydata.org/), I got used to [Apache Spark](https://spark.apache.org/) fairly quickly. However, its incredibly rapid development has taken its toll. I'm looking at you, CSV parser.

In fact, Spark didn't have native CSV support until recently, but it does have one now and working with it is straightforward. Or at least the API would suggest so. But there is a number of gotchas that may result in a massive data loss.

If you use just `spark.read.csv('filename.csv')`, you will get burned sooner or later. Look at the [documentation](http://spark.apache.org/docs/2.3.0/api/python/pyspark.sql.html#pyspark.sql.DataFrameReader.csv) and check all the options you can specify. There are three of particular interest.

**By default, Spark's CSV DataFrame reader does not conform to [RFC 4180](https://tools.ietf.org/html/rfc4180).** There are two pain points in particular. This is Spark's *default* behaviour that we need to fix with settings:

- Double-quotes in fields must be escaped with another double-quote, just like the aforementioned RFC states. However, Spark, for some reason, uses backslashes. So not only do you not parse standard CSVs properly, *it outputs invalid CSV files*. (I have submitted [a JIRA](https://issues.apache.org/jira/browse/SPARK-22236) for this.)
- Spark considers each newline to be a row delimiter, but that's not how CSVs work. You are allowed newlines within fields, as long as the field is enclosed in double quotes. Spark does this, because reading files line by line is very fast and it also makes large CSVs splittable - five workers can work on a single file - that is rather difficult to do when you want to read it correctly.

However, things get worse. Because the default `mode` in the stock CSV reader is `PERMISSIVE`, all corrupt fields will be set to null. So you may have a completely valid CSV file, but Spark only sees nulls.

So, first things first, set the `mode` to `FAILFAST` to get all the gory tracebacks whenever Spark trips up. Then set `escape` to `'"'` and `multiLine` to `True` (here the syntax is for PySpark, but it's extremely similar in Scala). The call then ends up being `spark.read.options(mode='FAILFAST', multiLine=True, escape='"').csv('file.csv')`.

Two notes before I let you go - first, do use schemas if you can, the CSV reader in question accepts them and it will help you along the way. Second, if you're in control of the data generator and you know the file doesn't contain quotes (e.g. just numerical data) or newlines, you may be better off using the default settings. But in many cases, you won't have this luxury and, worst of all - Spark will fail very silently by just replacing everything with nulls.