---
title: "Running Apache Spark (PySpark) jobs locally"
date: 2020-10-19T07:03:53+02:00
---

Apache Spark is not among the most lightweight of solutions, so it's only natural that there is a whole number of hosted solutions. AWS EMR, SageMaker, Glue, Databricks etc. While these services abstract out a lot of the moving parts, they introduce a rather clunky workflow with a slow feedback loop. In short, it's not quite like developing locally, so I want to talk about enabling that.

Now, as long as you have Java and Python installed, you're almost all set. At this point you can `python3 -m pip install pyspark`, be it in a virtual env or globally (the former is usually preferred, but it's up to your setup). Spark is neatly packaged with all the JARs included in the Python package, so there's nothing else to install.

When working in Spark, you get a global variable `spark` that enables you to read in data, create dataframes etc. This variable is usually conjured out of thin air in hosted solutions, so we'll have to create it ourselves.

Let's create a `pipeline.py` with just the following

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.getOrCreate()
print(spark.sql('select 1').collect())

```

The session alone would suffice, but the SQL statement is there just to test things out. Running `python3 pipeline.py` should launch a Spark session, run the simple query and exit (you could equally run `spark-submit`, but it's not necessary here). If you want to investigate the environment some more, you can run `python3 -i pipeline.py`, which is Python's way of saying “run this file and drop me in a Python interpreter with the state this run results in”. Super helpful for debugging.

## Taking things further

Since we have our Spark session, we can do our typical `df = spark.read.csv('our_data')`, but the data itself may be an issue. At this point, a lot of the data we interact with our within our shared infrastructure, be it HDFS or an object store like S3.

If we point our Spark instance at these sources, we will face a few issues.

1. We need libraries to access these filesystems, which makes our lean operation less lean.
2. There will be tremendous latency as well as high costs for out-of-network transfers (aka egress fees)
3. The source data are likely too big for local experimentation, so any quick development will be hampered by slow processing

We can solve all these issues by a simple sampling - just copy a few of your source files to your local disk (ideally as many files as your CPUs, to enable I/O parallelism). This will lead to quick local testing with no network involved (so you can even develop offline, if that's your thing) and just enough data for your laptop to easily handle.

Here's a more complete example:

```python
from pyspark.sql import SparkSession
import pyspark.sql.functions as f

if __name__ == '__main__':
    spark = SparkSession.builder.getOrCreate()
    df = spark.read.csv('sales_raw')
    df.filter(f.col('price') > 1000).groupby('region').count().show()
    df.write.partitionBy('region', 'brand').parquet('sales_processed')
```

A minimal Docker image for PySpark looks like this. This is just a generic example, make sure to pin both the base image and your dependencies.

```Dockerfile
FROM python:3.8-slim

# the mkdir call is there because of an issue in python's slim base images
# see https://github.com/debuerreotype/docker-debian-artifacts/issues/24
RUN mkdir -p /usr/share/man/man1 && apt-get -y update && apt-get -y --no-install-recommends install default-jre

# you'll likely use requirements.txt or some other dependency management
RUN python3 -m pip install pyspark
```

After building this (`docker build -t imgname .`), you can test things out, just make sure to override the entrypoint - the one inherited from the base image is a Python interpreter. So drop into the shell via `docker run --rm -it imgname bash` and off you go.

You're now ready to run a local, reproducible, quickly iterated workflow involving a distributed processing system. Isn't that nice?

Happy Sparking!

