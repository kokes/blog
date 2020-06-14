---
title: "High Performance Data Loss"
date: 2020-06-12T07:03:53+02:00
---


**This article reflects software as it was stable at the time of writing (June 2020). There are some ongoing changes, pending releases etc. that will fix some of these issues. I reported many of these and will edit the article if they get resolved.**

When talking about data loss, we usually mean some traditional ways of losing data, many of which are related to databases:

- `TRUNCATE` table, which deletes all the data in a table - and you may have just accidentally truncated the wrong one
- `DELETE` without a `WHERE` clause - you meant to delete only some rows, but ended up deleting all of them
- running against production instead of dev/staging - you think you're on your dev machine, you drop things left and right and then realise you messed up your production data
- crashing without backups - this is pretty common, you encounter a system failure, but there are no backups to restore from


These are fairly common and well understood and I don't mean to talk about them here. Instead, I want to talk about a class of data integrity issues much less visible than a dropped table. _We can actually lose data without noticing it at first. We may still have it, it's just incorrect due to a fault in our data pipeline._

How is that possible. Well, let's see what a program can do when it encounters some invalid input. There are basically three things it can do:

1. Fail (return an error, throw an exception etc.)
2. Return some NULL-like value (None, NULL, nil or whatever a given system uses)
3. Parse it incorrectly

I ordered these roughly in the order I'd expect a parser to behave. I subscribe to the notion that we should fail early and often, some systems don't, let's see about that.

## Lesson 1: parse all the formats

When it comes to stringified dates, I have [a favourite format](https://xkcd.com/1179/), that is [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601), because it is widely used, cannot be easily misinterpreted, and, as a bonus, is sortable.

Sadly, not all input data come in this format, so you need to accommodate that. You can usually specify the format you expect the string to be in and pass it to [strptime](https://docs.python.org/3/library/datetime.html#datetime.datetime.strptime) or its equivalent.

This all works if your data is homogenous or if you know how to construct such a date format string. But if either of those preconditions fails, it's no longer trivial. Luckily, there's [dateutil](https://pypi.org/project/python-dateutil/). This library has as a lot of _abstractions_ for a slightly friendlier work with dates and times. The functionality I use the most is its parser, it works like this


```python
from dateutil import parser

formats = ['2020-02-28', '2020/02/28', '28/2/2020', '2/28/2020', '28/2/20', '2/28/20']

[parser.parse(j).date().isoformat() for j in formats]
```




    ['2020-02-28',
     '2020-02-28',
     '2020-02-28',
     '2020-02-28',
     '2020-02-28',
     '2020-02-28']



No matter you throw at it, it will usually figure things out.

My use case was very similar. I was parsing dozens of datasets from our labour ministry and it had all sorts of date formats. There was the Czech variant of `23. 4. 2020`, but also `23. 4. 2020 12:34:33` and `23. 4. 2020 12` and a few others.

The issue was that the dateutil parser assumes m/d/y as the default (if the input is ambiguous), so fourth of July using the non-american notation (day first) would get detected as seventh of April.


```python
parser.parse('4. 7. 2020').date().isoformat()
```




    '2020-04-07'



Luckily dateutil has a setting, where you can hint that your dates tend to have days first.


```python
parser.parse('4. 7. 2020', dayfirst=True).date().isoformat()
```




    '2020-07-04'



... and it works as expected.

This worked just fine for a while, but then I noticed some of my dates were in the future, but they denoted things from the past. I dug into the code and found out what happened.

The data source switched from non-standard `4. 7. 2020` to ISO-8601, so `2020-07-04` - excellent! - but since dateutil was told `dayfirst`, it misinterpreted a completely valid, standard date, just because I changed some dateutil defaults.


```python
parser.parse('2020-07-04', dayfirst=True).date().isoformat()
```




    '2020-04-07'



We ended up with a mangled ISO date and were taught two lessons:

- sometimes explicit is better than implicit - if we know the possible source formats, we can map their conversions explicitly and have more control over them (hint: use [strptime](https://docs.python.org/3/library/datetime.html#datetime.datetime.strptime)/[fromisoformat](https://docs.python.org/3/library/datetime.html#datetime.time.fromisoformat))
- you need to err when there's an unexpected input - it's much safer to crash than trying to estimate what was probably meant (you will be wrong at some point)

This is a [known (and unresolved) issue](https://github.com/dateutil/dateutil/issues/268).

## Lesson 2: overflows are not just for integers

In computer science, if you have a container that can hold values of -128 to 127, computing 100\*2 will _overflow_ (~wrap around) and you'll get a negative number. I always thought that this was a numbers thing, but I recently found out it can happen for dates and time as well.

There is a number of occasions where you can get invalid dates - be it a confusion about their formats (you swap days and months in ISO-8601, for instance), it could be a bug (adding a year is just adding 1 to the year, not checking if that day exists, hint: [leap years](https://en.wikipedia.org/wiki/Leap_year_problem)), it could be data corruption, it can simply be many things.

As we learned in lesson 1, a sensible way to resolve this is... not to resolve it and just err.

For this lesson, we'll need [Apache Spark](http://spark.apache.org/), a very popular data processing engine with Python bindings.


```python
import os
import pyspark.sql.functions as f
import pyspark.sql.types as t

import pandas
import numpy
```


```python
os.makedirs('data', exist_ok=True)
```

Let's write some non-existent dates. We'll let Spark infer what the types are, we won't tell it. Since these dates are syntactically alright, but non-existent in reality, we can expect one of three things:

1. Error
2. NULLs instead of dates (and possibly a warning)
3. Dates will get recognised as strings as they don't conform to date formats


```python
with open('data/dates.csv', 'w') as fw:
    fw.write('''name,date
john,2019-01-01
jane,2019-02-30
doe,2019-02-29
lily,2019-04-31''')

df = spark.read.option('inferSchema', True).option('header', True).csv('data/dates.csv')
df.printSchema()
df.show()
```

    root
     |-- name: string (nullable = true)
     |-- date: timestamp (nullable = true)
    
    +----+-------------------+
    |name|               date|
    +----+-------------------+
    |john|2019-01-01 00:00:00|
    |jane|2019-03-02 00:00:00|
    | doe|2019-03-01 00:00:00|
    |lily|2019-05-01 00:00:00|
    +----+-------------------+
    


Turns out we don't get either of those three sensible results, dates overflow into the following month, which, to me, is pretty insane.

Let's try and hint that we have dates in our data. At this point only the first two options apply: error or null.


```python
schema = t.StructType([
    t.StructField('name', t.StringType(), True),
    t.StructField('date', t.DateType(), True),
])

df = spark.read.schema(schema).option('header', True).csv('data/dates.csv')
df.printSchema()
df.show()
```

    root
     |-- name: string (nullable = true)
     |-- date: date (nullable = true)
    
    +----+----------+
    |name|      date|
    +----+----------+
    |john|2019-01-01|
    |jane|2019-03-02|
    | doe|2019-03-01|
    |lily|2019-05-01|
    +----+----------+
    


And yet... nothing, still overflows.

Let's see what pandas does (type inference is not shown here, but pandas just assumes they are plain strings in that case). It errs as expected. This way we don't lose anything and we're forced to deal with the issue.


```python
df = pandas.read_csv('data/dates.csv')
try:
    pandas.to_datetime(df['date'])
except ValueError as e:
    print(e) # just for output brevity
```

    day is out of range for month: 2019-02-30


As I noted above - if you swap days and months, you can easily move years into the future. If you place garbage in any of the fields, you can move a century.


```python
with open('data/dates_swap.csv', 'w') as fw:
    fw.write('''name,date
john,2019-31-12
jane,2019-999-02
doe,2019-02-22''')
    
# luckily, inferSchema will work fine (will think it's a string)
schema = t.StructType([t.StructField('name', t.StringType(), True), t.StructField('date', t.DateType(), True)])
df = spark.read.schema(schema).option('header', True).csv('data/dates_swap.csv')
df.printSchema()
df.show()
```

    root
     |-- name: string (nullable = true)
     |-- date: date (nullable = true)
    
    +----+----------+
    |name|      date|
    +----+----------+
    |john|2021-07-12|
    |jane|2102-03-02|
    | doe|2019-02-22|
    +----+----------+
    


Digging into Spark's source, we can trace this back to Java's standard library. The API that gets used is not generally recommended:

```
scala> import java.sql.Date
import java.sql.Date

scala> Date.valueOf("2019-02-29")
val res0: java.sql.Date = 2019-03-01
```

This issue is being addressed within Apache Spark, the forthcoming major version (3) [will use a different Java API](https://issues.apache.org/jira/browse/SPARK-26178) and will change the behaviour described above. When letting Spark infer types automatically, it will not assume dates and will choose strings instead (no data loss). When supplying a schema, it will trow away all the invalid dates, i.e. we're losing data without any warnings.

Back in our current Spark 2.4.5, we're left with not just `java.sql.Date`, but also other implementations, because e.g. using Spark SQL directly won't trigger this issue. Go figure.


```python
spark.sql('select timestamp("2019-02-29 12:59:34") as bad_timestamp').show()
```

    +-------------+
    |bad_timestamp|
    +-------------+
    |         null|
    +-------------+
    


When researching these issues, I stumbled [upon a comment](https://issues.apache.org/jira/browse/SPARK-12045) by Michael Armbrust (a hugely influential person in the Spark community and a great speaker)

> Our general policy for exceptions is that we return null for data dependent problems (i.e. a date string that doesn't parse) and we throw an AnalysisException for static problems with the query (like an invalid format string).

Which just about sums up the issues raised earlier - it's more common to return a NULL instead of dealing with an issue - you'll see that yourself when running Spark at scale - it will not fail too often, but it will lose your data instead.

The major issue with this is that not only is it wrong... the result is also a perfectly valid date, so the usual mechanisms for detecting issues - looking for nulls, infinities, warnings etc. - none of that works, because the output looks just fine.

### Fun tidbit - Spark overflows time as well

In case you were wondering, you can place garbage into time as well, it will overflow like dates. (And just like the date issue, it will get addressed in Spark 3.0.)


```python
with open('data/times.csv', 'w') as fw:
    fw.write('''name,date
john,2019-01-01 12:34:22
jane,2019-02-14 25:03:65
doe,2019-05-30 23:59:00''')

df = spark.read.option('inferSchema', True).option('header', True).csv('data/times.csv')
df.printSchema()
df.show()
```

    root
     |-- name: string (nullable = true)
     |-- date: timestamp (nullable = true)
    
    +----+-------------------+
    |name|               date|
    +----+-------------------+
    |john|2019-01-01 12:34:22|
    |jane|2019-02-15 01:04:05|
    | doe|2019-05-30 23:59:00|
    +----+-------------------+
    



```python
with open('data/times99.csv', 'w') as fw:
    fw.write('''name,date
john,2019-01-01 12:34:22
jane,2019-02-14 13:303:65
doe,2019-05-30 984:76:44''')

df = spark.read.option('inferSchema', True).option('header', True).csv('data/times99.csv')
df.printSchema()
df.show()
```

    root
     |-- name: string (nullable = true)
     |-- date: timestamp (nullable = true)
    
    +----+-------------------+
    |name|               date|
    +----+-------------------+
    |john|2019-01-01 12:34:22|
    |jane|2019-02-14 18:04:05|
    | doe|2019-07-10 01:16:44|
    +----+-------------------+
    


### Funner tidbit - casting is different

While we did get date overflows when reading data straight into dates, what if we read in strings and only later decide to cast to dates. Should do the same thing, right? (I guess you see where I'm going with this.)


```python
fn = 'data/dates.csv'

df = spark.read.option('header', True).csv(fn)
df.show()
```

    +----+----------+
    |name|      date|
    +----+----------+
    |john|2019-01-01|
    |jane|2019-02-30|
    | doe|2019-02-29|
    |lily|2019-04-31|
    +----+----------+
    



```python
df.select(f.col('date').cast('date')).show()
```

    +----------+
    |      date|
    +----------+
    |2019-01-01|
    |      null|
    |      null|
    |      null|
    +----------+
    


### Integer overflows

I already touched on overflows earlier, let's put them to a test. Let's say we have a few numbers and one of them is too large for a 32-bit integer. Let's tell Spark it's an integer anyway and see what happens.


```python
fn = 'data/integers.csv'

with open(fn, 'w') as fw:
    fw.write('id\n123123\n123123123\n123123123123')
```


```python
schema = t.StructType([
    t.StructField('id', t.IntegerType(), False)
])

spark.read.option('header', True).schema(schema).csv(fn).show()
```

    +---------+
    |       id|
    +---------+
    |   123123|
    |123123123|
    |     null|
    +---------+
    


Okay, we get a null - fair(-ish). This is the same reason as noted above in the Armbrust quote - Spark prefers nulls over exceptions. (The number would load just fine if we specified longs instead of integers.)

Let's digest what this means - if you have a column of integers - say an incrementing ID - and your source gets so popular you keep ingesting more and more data that it no longer fits into int32, Spark will start throwing data out without telling you so. You have to ensure you have data quality mechanisms in place to catch this.

Note that if you do supply a schema upon data loading, pandas will fail the same way, at least if you use the venerable numpy type offering.


```python
print(pandas.read_csv(fn, dtype={'id': numpy.int32})) # is this better? not really
```

               id
    0      123123
    1   123123123
    2 -1430928461


You have to resort to pandas 1.x extension arrays type annotation to avoid issues.


```python
try:
    pandas.read_csv(fn, dtype={'id': 'Int32'})
except Exception as e:
    print(e)
```

    cannot safely cast non-equivalent int64 to int32


(Type inference "solves" the issue for both Spark and pandas.)

## Lesson 3: local problems can get global

You might be thinking: hey, problems in some random column don't concern me - I'm all about `amount_paid` and `client_id`, both of which are just fine. Well, not so fast.

1. If you filter on these mangled columns (e.g. "give me all the invoices where `amount_paid` is so and so") - you will lose data if this column is not parsed properly.
2. This mad thing you're about to read below.

Let's have a dataset of three integer columns. Or at least these have always been integers, but somehow you got a float in there (never mind the value is the same* as the integer). We'll illustrate how this can easily happen in real life.


```python
schema = t.StructType([
    t.StructField('a', t.IntegerType(), True),
    t.StructField('b', t.IntegerType(), True),
    t.StructField('c', t.IntegerType(), True),
])

with open('data/null_row.csv', 'w') as fw:
    fw.write('''a,b,c
10,11,12
13,14,15.0''')

spark.read.schema(schema).option('header', True).csv('data/null_row.csv').show()
```

    +----+----+----+
    |   a|   b|   c|
    +----+----+----+
    |  10|  11|  12|
    |null|null|null|
    +----+----+----+
    


The crazy thing that just happened is that just because `c`'s second value is a float instead of an integer, Spark decided to _throw away the whole row_ - so if you're filtering/aggregating on other columns, your analysis is already wrong now (this did bite us _hard_ at work).

Luckily, this gets fixed in the upcoming major version of Spark (3.0, couldn't find the ticket just now, but it does get resolved) and only that one single value is NULLed.

Also, like with some (sadly, not all) of the issues described here, you can catch these nasty bugs by turning the permissiveness down (mode=FAILFAST in the case of file I/O).


```python
try:
    spark.read.schema(schema).option('header', True).option('mode', 'FAILFAST').csv('data/null_row.csv').show()
except Exception as e:
    print(str(e)[:500])
```

    An error occurred while calling o114.showString.
    : org.apache.spark.SparkException: Job aborted due to stage failure: Task 0 in stage 17.0 failed 1 times, most recent failure: Lost task 0.0 in stage 17.0 (TID 17, localhost, executor driver): org.apache.spark.SparkException: Malformed records are detected in record parsing. Parse Mode: FAILFAST.
    	at org.apache.spark.sql.execution.datasources.FailureSafeParser.parse(FailureSafeParser.scala:70)
    	at org.apache.spark.sql.execution.datasources.csv.Uni


## Lesson 4: CSVs are fun until they aren't

CSVs look like such a simple format. A bunch of values separated by commas and newlines, seems hardly complicated. But that's only half of the story, there are two format aspects that don't get that much attention. 

1. If you want to use quotes in fields, they need to be escaped _by another quote_, not by a backslash like in most other places.
2. You can have newlines in fields. As many as you want in fact. You only need to quote your field first.

There isn't a standard per se, but [RFC 4180](https://tools.ietf.org/html/rfc4180) is the closest we got. Let's now leverage the fact that these rules are not always adhered to.


```python
import csv
```


```python
fn = 'data/quote.csv'

with open(fn, 'w') as fw:
    cw = csv.writer(fw)
    cw.writerow(['name', 'age'])
    cw.writerow(['Joe "analyst, among other things" Sullivan', '56'])
    cw.writerow(['Jane Doe', '44'])
```


```python
df = pandas.read_csv(fn)
print(df)
```

                                             name  age
    0  Joe "analyst, among other things" Sullivan   56
    1                                    Jane Doe   44


So here we have a CSV with some people and their ages, let's ask pandas what their average age is.


```python
df.age.mean()
```




    50.0



Now let's ask Spark the very same question.


```python
df = spark.read.option('header', True).option('inferSchema', True).csv(fn)
df.select(f.mean('age')).show()
```

    +--------+
    |avg(age)|
    +--------+
    |    44.0|
    +--------+
    


The reason is that Spark, _by default_, doesn't escape quotes properly, so instead of parsing the first non-header row as two fields, it spilled the name field into the age field and thus threw away the age information.


```python
df.show()
```

    +--------------+--------------------+
    |          name|                 age|
    +--------------+--------------------+
    |"Joe ""analyst| among other thin...|
    |      Jane Doe|                  44|
    +--------------+--------------------+
    


Not only did it completely destroy the file, it also didn't complain during the sum!

And this continues on if we finish the pipeline - if we actually write the data some place, it gets further destroyed.


```python
df.write.mode('overwrite').option('header', True).csv('data/write1')
```


```python
rfn = [j for j in os.listdir('data/write1') if j.endswith('.csv')][0]
```


```python
print(pandas.read_csv(os.path.join('data/write1/', rfn)))
```

                    name                                age
    0  \Joe \"\"analyst"  among other things\\" Sullivan\""
    1           Jane Doe                                 44


At this point we used Spark to load a CSV and write it back out and in the process, we lost data.

### FAILFAST to the not rescue

I noted earlier that the `FAILFAST` mode is quite helpful, though since it's not the default, it doesn't get used as much as I'd like. Let's see if it helps here.


```python
df = spark.read.option('header', True).option('mode', 'FAILFAST').option('inferSchema', True).csv(fn)
df.select(f.mean('age')).show()
```

    +--------+
    |avg(age)|
    +--------+
    |    44.0|
    +--------+
    


You need to adjust quoting in the Spark CSV reader to get this right.


```python
df = spark.read.option('header', True).option('escape', '"').option('inferSchema', True).csv(fn)
df.show()
```

    +--------------------+---+
    |                name|age|
    +--------------------+---+
    |Joe "analyst, amo...| 56|
    |            Jane Doe| 44|
    +--------------------+---+
    


## Lesson 5: CSVs are really not trivial

It's easy to pick on one technology, so let's look at one other implementation of CSV I/O - `encoding/csv` in Go, that's their standard library implementation. The issue presents itself when you have a single column of data - this happens all the time. Oh and you have missing values. Let's take this example:

```
john
jane

jack
```

It's a simple dataset of four values, the third one is missing. If we write this dataset using `encoding/csv` and read it back in, we won't get the same thing. This is what's called a roundtrip test. Here's my [longer explanation](https://github.com/golang/go/issues/39119) together with a snippet of code.

Now this fails, because `encoding/csv` skips all empty rows - but in this case an empty row is a datapoint. The implementation departs from RFC 4180 in a very subtle way, which, in our case, leads to loss of data.

I reported the issue, submitted a PR, but was told it probably won't (ever) be fixed.

## Lesson 6: all bets are off

So far I've shown you actual cases of data loss that I unfortunately experienced first hand. Now let's look at a hypothetical - what if someone knew about this problem and wanted to exploit it. Exploit a CSV parser? Do tell!

Let's look at a few pizza orders.


```python
df = pandas.read_csv(fn)
df['note'] = df['note'].str[:20]

print(df)
```

        name        date  pizzas                  note
    0   john  2019-09-30       3          cash payment
    1   jane  2019-09-13       5     2nd floor, apt 2C
    2  wendy  2019-08-30       1  no olives, moar chee


How many pizzas did people order?


```python
df.pizzas.sum()
```




    9



Correct answer is... well, it depends who you ask 😈


```python
sdf = spark.read.option('header', True).option('mode', 'FAILFAST').option('inferSchema', True).csv(fn)
sdf.agg(f.sum('pizzas')).show()
```

    +-----------+
    |sum(pizzas)|
    +-----------+
    |     127453|
    +-----------+
    


So now instead of nine pizzas, we need to fulfil 127 _thousand_ orders, that's a tall order (sorry not sorry).

Wonder what happened? It's not much clearer when we look at the dataset.


```python
sdf.show()
```

    +--------+-------------------+------+--------------------+
    |    name|               date|pizzas|                note|
    +--------+-------------------+------+--------------------+
    |    john|2019-09-30 00:00:00|     3|        cash payment|
    |    jane|2019-09-13 00:00:00|     5|   2nd floor, apt 2C|
    |   wendy|2019-08-30 00:00:00|     1|no olives, moar c...|
    |    rick|               null|   123|                null|
    |   suzie|               null|112233|                null|
    |     jim|               null| 13593|                null|
    |samantha|               null|    29|                null|
    |   james|               null|  1000|                null|
    |  roland|               null|   135|                null|
    |   ellie|               null|   331|                   "|
    +--------+-------------------+------+--------------------+
    


It's a touch clearer if we look at the file itself. In the third order, I decided to play with the note value. I decided to span multiple lines (a completely valid CSV value) and leveraged the fact that Spark, by default, reads CSVs line by line and will interpret all the subsequent lines (of our note!) as new records. I can fabricate a lot of data this way.


```python
with open('data/pizzas.csv') as fr:
    print(fr.read())
```

    name,date,pizzas,note
    john,2019-09-30,3,cash payment
    jane,2019-09-13,5,"2nd floor, apt 2C"
    wendy,2019-08-30,1,"no olives, moar cheese
    rick,,123,
    suzie,,112233,
    jim,,13593,
    samantha,,29,
    james,,1000,
    roland,,135,
    ellie,,331,"


We managed to silently inject data into an analysis by leveraging _the most common data format_ and _one of the most used data engineering tools_. That's pretty dangerous.

If we want to fix it in Spark, we need to force it to take into consideration multiline values.


```python
sdf = spark.read.option('header', True).option('multiLine', True).option('inferSchema', True).csv(fn)
sdf.show()
```

    +-----+-------------------+------+--------------------+
    | name|               date|pizzas|                note|
    +-----+-------------------+------+--------------------+
    | john|2019-09-30 00:00:00|     3|        cash payment|
    | jane|2019-09-13 00:00:00|     5|   2nd floor, apt 2C|
    |wendy|2019-08-30 00:00:00|     1|no olives, moar c...|
    +-----+-------------------+------+--------------------+
    


(Note that this breaks Unix tools like grep or cut, because these don't really operate on CSVs, they are line readers.)

## Bonus: a magic trick

With all the standard examples out of the way, let's now try a fun trick I learned of by accident, when I wanted to demonstrate something completely different.

Let's look at a small dataset, it's a list of children, what's their average age?


```python
fn = 'data/trick.csv'

with open(fn, 'w') as fw:
    fw.write('''name,age
john,1
jane,2
joe,
jill,3
jim,4''')
```

Let's see, we'd guess 2.5, right? Maybe 2 since there's a missing value. But engines tend to skip missing values for aggregations.


```python
schema = t.StructType([t.StructField('name', t.StringType(), True), t.StructField('age', t.IntegerType(), True)])
df = spark.read.option('header', True).schema(schema).csv(fn)
df.agg(f.mean('age')).show()
```

    +--------+
    |avg(age)|
    +--------+
    |     2.5|
    +--------+
    


Now let's narrow the dataset down to only 1000 rows and ask Spark again. Just to be sure, let's do it twice, the same thing!


```python
for _ in range(2):
    pandas.read_csv('data/trick.csv').head(1000).to_csv('data/trick.csv') # lossless, right?

    # this is the exact same code as above
    df = spark.read.option('header', True).schema(schema).csv(fn)
    df.agg(f.mean('age')).show()
```

    +--------+
    |avg(age)|
    +--------+
    |    null|
    +--------+
    
    +--------+
    |avg(age)|
    +--------+
    |     2.0|
    +--------+
    


We managed to get three different answers from a seemingly identical dataset. _How?_

## Tidbits from other systems

Now we only covered a handful of tools, there are tons more to dissect. Here is just a quick list of notes from using other systems:

- Be careful when saving the abbreaviation of Norway in YAML, it might get turned to `false` (see [the Norway problem](https://hitchdev.com/strictyaml/why/implicit-typing-removed/) for more details).
- Some scikit-learn defaults [have been questioned](https://ryxcommar.com/2019/08/30/scikit-learns-defaults-are-wrong/) - do we know what defaults our _[complex piece of software]_ uses?
- When you go distributed, a whole host of consistency issues arise. If you like to sleep at night, I don't recommend Martin Kleppmann's [Designing Data-Intensive Applications](http://dataintensive.net/) or Kyle Kingbury's [Jepsen suite](http://jepsen.io/).
 - Some reconciliation systems are _designed_ to lose data, see [Last Writer Wins (LWW)](https://dzone.com/articles/conflict-resolution-using-last-write-wins-vs-crdts)
 - Building distributed data stores is really hard and some popular ones can lose a bunch of data in some cases. See Jepsen's reports on MongoDB or Elasticsearch (and keep in mind the version these are valid for).
- Parsing JSON is much more difficult than parsing CSVs, the dozens of mainstream parsers don't seem to agree on what should be done with various inputs (see [Parsing JSON is a minefield](http://seriot.ch/parsing_json.php))

## The future

I'd love to say that all of these issues will get resolved, but I'm not that certain. Some of these technologies had a chance to use major version upgrades to introduce breaking improvements, but they chose not to do so.

- pandas 1.0 still has nullable int columns by default, despite offering superior functionality as an opt-in (we didn't explicitly cover this issue, but it was a culprit in one of my examples)
- Apache Spark has improved its date handling, but the CSV situation is still dire, the/my issue has been out there for years now and v3.0 was the perfect candidate for a breaking improvement, but no dice.
- Go's encoding/csv can still lose data and it seems my battle to improve this has been lost already, so the only chance is to fork the package or use a different one.

## Conclusion

While I described many very specific issues, the morale of this article is more about software abstractions. Do you _really_ know what's happening, when you do things like `pandas.read_csv`? Do you understand what things get executed during a Spark query? Frankly, you should keep asking these things as you use complicated software.

There are tons of abstractions these days and it's one of the reasons Python has become so popular, but it's also a very convenient footgun. So many abstractions, so many ways to lose data.

While we did talk about the various ways to mess things up, we didn't quite cover how to get out of it, that's perhaps a topic for another day. But there are a few general tips to give you:

- Correctness should always trump performance. Make sure you only pursue performance once you have ensured your process works correctly.
- Explicit is often better than implicit. Sounds like a cliche, but I mean it - abstractions will save you a lot of trouble, until you reach a point where the implicit mechanisms within a giant package are actually working against you. Sometimes it makes sense to get your hands dirty and code it up from scratch.
- Fail early, fail often. Don't catch all the exceptions, don't log them instead of handling them, don't throw away invalid data. Crash, fail, err, make sure errors get exposed, so that you can react to them.
- Have sensible defaults. It's not enough your software _can_ work correctly, it's important it _does_ work correctly from the get go.
- The job of (not just) data engineers is to move data *reliably* - the end user doesn¨t care if it’s Spark or Cobol, they want their numbers to line up. Always put your data integrity above your technology on your list of priorities.
- Know your tools. Make sure you understand at least a part of what's happening under the hood, so that you can better diagnose it when it goes wrong. And it will.


Last but not least - this article doesn't mean to deter you from using abstractions. Please do, but be careful, get to know them, understand how they work, and good luck!
