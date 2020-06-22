---
title: "Accessing S3 data with Apache Spark from stock PySpark"
date: 2020-06-22T07:03:53+02:00
---

For a while now, you've been able to run `pip install pyspark` on your machine and get all of Apache Spark, all the jars and such, without worrying about much else. While it's a great way to setup PySpark on your machine to troubleshoot things locally, it comes with a set of caveats - you're essentially running a distributed, hard to maintain system... via pip install.

While I'm used to Spark being managed (by either a team at work, Databricks or within Amazon's EMR), I like to run things locally. I've been using PySpark via pip for quite a while now, but I've always run into issues with S3. Let's now go through them and look at some resolutions to these problems.

For the purposes of this post, I created two S3 buckets - `ok-eu-central-1` in Frankfurt and `ok-us-west-2` in Oregon. We'll see why later on. Let's pip install PySpark and try a simple pipeline that just exports a single data point.

```python
from pyspark.sql import SparkSession


if __name__ == '__main__':
    spark = (SparkSession
                .builder
                .appName('my_export')
                .master("local[*]")
                .getOrCreate()
            )
    spark.sql('select 1').write.mode('overwrite').csv('s3a://ok-us-west-2/tmp')
```

But when we invoke `spark-submit pipeline.py`, we get the following:

```
py4j.protocol.Py4JJavaError: An error occurred while calling o34.csv.
: java.lang.RuntimeException: java.lang.ClassNotFoundException: Class org.apache.hadoop.fs.s3a.S3AFileSystem not found
```

This is because Spark doesn't ship with the relevant AWS libraries, so let's fix that. We can specify a dependency that will get downloaded upon first invocation (so be careful when bundling this in e.g. Docker, you'll want to ship it differently there):

```
$ spark-submit --packages org.apache.hadoop:hadoop-aws:2.7.4 pipeline.py 
```

Here 2.7.4 reflects the version of Hadoop libraries shipped with PySpark - at the time of writing (June 2020), there's (Py)Spark version 3.0.0 out there and it ships with 2.7.4. While Spark now supports Hadoop 3.x, this is what it ships with by default.

When we run the command above, it all works fine! So if it works for you now, you see what you have to do - you need to instruct Spark (somehow), to include this AWS dependency. I did this at runtime via a cli argument, but there are other (better) ways.

## Not all regions are equal

Now, this all works, but let's change the code above slightly. Let's write to the bucket in Frankfurt.

```python
# ...
    spark.sql('select 1').write.mode('overwrite').csv('s3a://ok-eu-central-1/tmp')
```

Now the invocation, even with the `--packages` argument won't work. It will fail with a 400 error:

```
py4j.protocol.Py4JJavaError: An error occurred while calling o37.csv.
: com.amazonaws.services.s3.model.AmazonS3Exception: Status Code: 400, AWS Service: Amazon S3, AWS Request ID: (...), AWS Error Code: null, AWS Error Message: Bad Request, S3 Extended Request ID: (...)=

```

A 400 is a bad request, so something we did was incorrect. If you go down the rabbit hole, you'll find out that there are multiple versions of the communication protocol and new regions only support a new protocol (v4). Sadly, the `hadoop-aws` library in version 2.7.4 defaults to v2, so it cannot handle Frankfurt and other regions without further settings. We can't just upgrade hadoop-aws, because there are other Hadoop-related libraries linked and their versions need to match.

There are two things you need to do, if you work with Spark on Hadoop 2 and new AWS regions. First, specify that you're using the v4 protocol. Here I'm submitting it as an argument (and for the driver only, since I'm using the standalone mode), you'll want to specify this elsewhere.

```
spark-submit --packages org.apache.hadoop:hadoop-aws:2.7.4 --conf spark.driver.extraJavaOptions='-Dcom.amazonaws.services.s3.enableV4' pipeline.py
```

Second, we need to setup the endpoint for our S3 bucket. Not to diverge too far, but S3 libraries tend to have an `endpoint` option, so that you can point them at other servers, even outside of AWS, as long as they support the S3 API. This is very useful for testing, among other things. Here we'll leverage this option, but only to point it within AWS.

```python
from pyspark.sql import SparkSession


if __name__ == '__main__':
    spark = (SparkSession
                .builder
                .appName('my_export')
                .master("local[*]")
                .config('fs.s3a.endpoint', 's3.eu-central-1.amazonaws.com') # this has been added
                .getOrCreate()
            )
    spark.sql('select 1').write.mode('overwrite').csv('s3a://ok-eu-central-1/tmp')
```

And that's it. Now that we've specified the endpoint, protocol version, and hadoop-aws, we can finally write to new S3 regions. Check out [the relevant AWS docs](https://docs.aws.amazon.com/general/latest/gr/s3.html) to get your region's endpoint.

I'm hoping that Spark switches to Hadoop 3.x for its PySpark distribution, that way we can avoid most of the shenanigans here (we'll only need `hadoop-aws` at that point).

Happy data processing!
