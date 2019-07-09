---
title: "How to lose data in Apache Spark"
date: 2019-07-09T07:03:53+02:00
draft: false
---

I've been using Spark for some time now, it has not always been smooth sailing. I can understand a tool's limitations as long as I'm told so, explicitly. The trouble with Apache Spark has been its insistence on having the wrong defaults. How wrong? So wrong they lose your data in unexpected ways.

Let's look at a few examples. I'm running all of these in PySpark 2.4.0 on macOS.

```python
import pyspark.sql.types as t
import pyspark.sql.functions as f
```

### Missing overflows

I had some integer data, which, over time, grew more and more and it became a set of longs. Let's say it looked like this:

```python
with open('data.csv', 'w') as fw:
    fw.write('id\n123123\n123123123\n123123123123')
```

If we load it in Spark, we find it there, it's all there.

```python
spark.read.option('header', True).csv('data.csv').show()
```

    +------------+
    |          id|
    +------------+
    |      123123|
    |   123123123|
    |123123123123|
    +------------+
    


The reason Spark has not lost any data at this point is the lack of schema. Let's tell Spark these numbers are integers.

```python
schema = t.StructType([
    t.StructField('id', t.IntegerType(), False)
])

spark.read.option('header', True).schema(schema).csv('data.csv').show()
```

    +---------+
    |       id|
    +---------+
    |   123123|
    |123123123|
    |     null|
    +---------+
    


The last datapoint is missing now. No errors, no warnings.

Let's now say the numbers are longs. It all works out just fine.

```python
schemal = t.StructType([
    t.StructField('id', t.LongType(), False)
])

spark.read.option('header', True).schema(schemal).csv('data.csv').show()
```

    +------------+
    |          id|
    +------------+
    |      123123|
    |   123123123|
    |123123123123|
    +------------+
    


What if we use schema inference? That works, because Spark does an extra pass over all your data. But schema inference is super brittle, you never know what sort of data is coming your way. A change in a single row of your inputs can destroy your whole application.

```python
spark.read.option('header', True).option('inferSchema', True).csv('data.csv').show()
```

    +------------+
    |          id|
    +------------+
    |      123123|
    |   123123123|
    |123123123123|
    +------------+
    


### Working with non-existent dates

Let's say your systems are not working quite correctly and they contain some dates that don't make any sense. Like the third one and later here:

```python
with open('dates.csv', 'w') as fw:
    fw.write('''date
2018-03-30
1990-01-24
2019-01-32
2019-02-29
2019-12-37''')
```

What if we read these as dates? And explicitly extract the month number.

```python
schemad = t.StructType([
    t.StructField('date', t.DateType(), False),
])

df = spark.read.option('header', True).schema(schemad).csv('dates.csv')
df.withColumn('month', f.month('date')).show()
```

    +----------+-----+
    |      date|month|
    +----------+-----+
    |2018-03-30|    3|
    |1990-01-24|    1|
    |2019-02-01|    2|
    |2019-03-01|    3|
    |2020-01-06|    1|
    +----------+-----+
    


In a bizarre data manipulation, Spark actually overflowed these incorrect dates instead of emitting an error or filling in NULLs. Literally anything other than this would be better.

### CSVs don't have a standard per se, but let's be reasonable...

CSVs are tricky, they don't have an explicit standard, the closest we have is [RFC 4180](https://tools.ietf.org/html/rfc4180) and data processing tools tend to follow it. Not Spark. Not Spark with its default settings. There are two simple rules that get violated:

1. Quoted fields can contain newlines.
2. Quotation marks within a field's content are escaped with another quotation mark (not a backslash).

Let's write some standards (sic) compliant data.

```python
import csv

with open('data.csv', 'w') as fw:
    cw = csv.writer(fw)
    cw.writerow(["message", "upvotes"])
    cw.writerow(["hello\nworld", 12])
    cw.writerow(["also hello", 9])
```

These are two rows of data, but Spark thinks there are three rows.

```python
df = spark.read.option('header', True).csv('data.csv')
df.count()
```




    3



The first one has an empty column...

```python
df.withColumn('nan', f.isnull(f.col('upvotes'))).select('nan').show()
```

    +-----+
    |  nan|
    +-----+
    | true|
    |false|
    |false|
    +-----+
    


That's all because Spark, by default, reads CSV row by row, even though column values can easily span any number of rows you choose.

```python
df.show()
```

    +----------+-------+
    |   message|upvotes|
    +----------+-------+
    |     hello|   null|
    |    world"|     12|
    |also hello|      9|
    +----------+-------+
    


### Read your own writes, please

Let's now read whatever we write to disk. These are two columns, one row, all strings.

```python
spark.createDataFrame([['foo', 'bar\nbaz']]).write.mode('overwrite').csv('export')

spark.read.csv('export').show()
```

    +----+----+
    | _c0| _c1|
    +----+----+
    | foo| bar|
    |baz"|null|
    +----+----+
    


Completely broken, not only is it two rows now, the values have now shifted, there's an extra quotation mark, a null value, it's a mess.

Let's now write something and read it elsewhere.

```python
spark.createDataFrame([['I have a 2.5" drive']]).coalesce(1).write.mode('overwrite').csv('export')

spark.read.csv('export').show()
```

    +-------------------+
    |                _c0|
    +-------------------+
    |I have a 2.5" drive|
    +-------------------+
    


So far so good.

```python
import csv
from glob import glob
```

```python
fn = glob('export/*.csv')[0]

with open(fn) as fr:
    cr = csv.reader(fr)
    print(next(cr))
```

    ['I have a 2.5\\ drive"']


We've now lost data, because Spark doesn't write RFC 4180 CSVs.

Luckily, there's a way out - [I've written about](https://kokes.github.io/blog/2018/05/19/spark-sane-csv-processing.html) all the relevant `spark.read.csv` options that make your life easier.

### Casting is risky

We sometimes load data in some format, but want to change them into another. Let's look at how we can lose data in this process.

Let's start with the file we defined in the very first example.

```python
with open('data.csv', 'w') as fw:
    fw.write('id\n123123\n123123123\n123123123123')

df = spark.read.option('header', True).schema(schemal).csv('data.csv')
df.show()
```

    +------------+
    |          id|
    +------------+
    |      123123|
    |   123123123|
    |123123123123|
    +------------+
    


What happens when we cast the (long) id here into an integer?

```python
df.withColumn('id', df.id.cast('int')).show()
```

    +-----------+
    |         id|
    +-----------+
    |     123123|
    |  123123123|
    |-1430928461|
    +-----------+
    


It overflows. (Remember it was a NULL above.)

What if we load those non-existent dates as strings and then change our minds and cast them to dates?

```python
df = spark.read.option('header', True).csv('dates.csv')
df.show()
```

    +----------+
    |      date|
    +----------+
    |2018-03-30|
    |1990-01-24|
    |2019-01-32|
    |2019-02-29|
    |2019-12-37|
    +----------+
    


```python
df.withColumn('date', df.date.cast('date')).show()
```

    +----------+
    |      date|
    +----------+
    |2018-03-30|
    |1990-01-24|
    |      null|
    |      null|
    |      null|
    +----------+
    


So when reading it with a schema, Spark mutated the datapoints into non-sense values... now it's throwing them away altogether.

### Untyped partitions are untyped

I would sometimes export data in partitions, the partition keys being strings. But these strings could be converted into integers, if one wished. And that turns out to be a bad thing.

Let's write a dataframe into three partitions based on its id column, this is either `001`, `002` or `003`.

```python
import os

schemap = t.StructType([
    t.StructField('id', t.StringType(), True),
    t.StructField('value', t.StringType(), True),
])

df = spark.createDataFrame([['001', 'foo'], ['002', 'bar'], ['003', 'bak']], schema=schemap)

df.coalesce(1).write.mode('overwrite').partitionBy('id').json('partitioned')
```

We get a directory per partition, all is good.

```python
os.listdir('partitioned')
```




    ['id=002', '._SUCCESS.crc', 'id=003', '_SUCCESS', 'id=001']



Let's read it back in. Now we should have one row per partition. (Note that this only works, because Spark coerced both sides to the same data type.)

```python
df = spark.read.json('partitioned')

df.filter(f.col('id') == '001').count()
```




    1



Equality works just fine, but if we try an `.isin` operation to check against potentially a larger list of values, the coercion mechanism within Spark leaves us with no data. Luckily, we didn't lose any data, we just haven't retrieved them. Which, in some cases, can be the same thing.

```python
df.filter(f.col('id').isin(['001'])).count()
```




    0



### Cherish integrity, not just with mission critical data

There is an inherent appeal in having a super performant and full featured technology like Apache Spark. But when it starts losing my data with default settings, it makes it something I can't honestly recommend to anyone.

When your job description is to manage data pipelines, you can't be losing data while performing mundane operations. Your tool should always err when faced with a problematic input, anything other than that is dangerous.

