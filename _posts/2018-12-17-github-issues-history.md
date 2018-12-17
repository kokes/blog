---
title: "Github Issues History"
date: 2018-12-17T12:03:53+02:00
draft: false
---

I owe quite a bit to the pandas Python package and even though I rarely use it these days, I feel like I should give back to the community. I started a while ago, just browsing [all those issues on Github](https://github.com/pandas-dev/pandas/issues). Where I could lend a hand, I did, where I couldn't, I just read through the problems the community was facing, to better understand the design and decision making within a large OSS project.

Just the other day, I was wondering what the issue dynamic was. Meaning there are now 2700+ open issues (at the time of writing), but it's hard to tell what the trend is. If you look at the [Pulse](https://github.com/pandas-dev/pandas/pulse/monthly), it shows statistics for the latest day/week/month, but that doesn't give you a long term trend. So while the past 30 days showed 250 closed issues and only 147 open issues, the reality is quite different.

First I needed to grab relevant data. You can use [BigQuery data dumps](https://medium.com/google-cloud/analyzing-github-issues-and-comments-with-bigquery-c41410d3308), but that comes with a few caveats, so I decided to download all the data from Github's API instead. I only needed to write this short script that downloads all issues and can even update the data as when run repeatedly. The code has no dependencies apart from Python 3. You only need to get an API key and save it into a relevant environment variable. Then you're good to go. I'm using sqlite3 (built into Python 3) to handle all the necessary upserts.

```
import json
import os
import sqlite3
from contextlib import closing
from urllib.request import urlopen

TOKEN = os.environ['GITHUB_PERSONAL_API_TOKEN']

ENDPOINT = 'https://api.github.com/repos/pandas-dev/pandas/issues?since={since}&access_token={token}&page={page}&sort=updated&direction=asc&state=all'

def get_issues(since):
    page = -1
    while True:
        page += 1
        r = urlopen(ENDPOINT.format(page=page, token=TOKEN, since=since))
        dt = json.load(r)
        if len(dt) == 0:
            return
        
        yield from (j for j in dt if 'pull_request' not in j)


if __name__ == '__main__':
    db = sqlite3.connect('github.db')

    with closing(db):
        db.execute('''create table if not exists pandas_issues(
         id int not null primary key, title text not null, state text not null, created_at text not null,
         updated_at text not null, closed_at text, payload text not null)''')

        insert_query = '''
        insert into pandas_issues(id, title, state, created_at, updated_at, closed_at, payload)
        values(:id, :title, :state, :created_at, :updated_at, :closed_at, :payload)
        ON CONFLICT(id) DO UPDATE SET title = EXCLUDED.title, state = EXCLUDED.state, created_at = EXCLUDED.created_at,
        updated_at = EXCLUDED.updated_at, closed_at = EXCLUDED.closed_at, payload = EXCLUDED.payload
        '''

        maxd = db.execute('select max(updated_at) from pandas_issues').fetchone()[0] or '1970-01-01T00:00:00Z'
        dd = ({'id': el['number'], 'title': el['title'], 'state': el['state'], 'created_at': el['created_at'],
               'updated_at': el['updated_at'], 'closed_at': el['closed_at'], 'payload': json.dumps(el, ensure_ascii=False)}
                      for el in get_issues(maxd))

        for j, el in enumerate(dd):
            db.execute(insert_query, el)
            if j%100 == 0:
                db.commit()

        db.commit()

```


Now that you have all the relevant issues, you can grep them to see what's what. It's quite easy to aggreagate data and see how the issue total evolved over time. For convenience, I migrated the dataset from SQLite to Postgres, just because I like having full outer joins, proper date columns and other nicities (SQLite only recently gained support for essential windows functions).

```
with closed as (
	SELECT date_trunc('month', closed_at)::date as "month", count(*) as "count_closed"
	FROM pandas_issues
	where closed_at is not null
	group by 1
), created as (
	SELECT date_trunc('month', created_at)::date as "month", count(*) as "count_created"
	FROM pandas_issues
	group by 1
)

select
month, coalesce(count_closed, 0) as count_closed, coalesce(count_created, 0) as count_created,
sum(coalesce(count_created, 0) - coalesce(count_closed, 0)) over(order by "month" asc)

from closed full outer join created using("month")
order by "month" desc;
```

Output of this query gives us a bleak overview of what's happening. Month over month the issues mount, only in July did we see a month where more issues got closed, thus bringing down the issue total.

| month        | count_closed | count_created | sum  | 
|--------------|--------------|---------------|------| 
| 2018-12-01 | 134          | 144           | 2709 | 
| 2018-11-01 | 244          | 284           | 2699 | 
| 2018-10-01 | 188          | 225           | 2659 | 
| 2018-09-01 | 116          | 192           | 2622 | 
| 2018-08-01 | 150          | 217           | 2546 | 
| 2018-07-01 | 273          | 227           | 2479 | 
| 2018-06-01 | 204          | 233           | 2525 | 
| 2018-05-01 | 141          | 224           | 2496 | 
| 2018-04-01 | 131          | 198           | 2413 | 
| 2018-03-01 | 148          | 214           | 2346 | 
| 2018-02-01 | 174          | 225           | 2280 | 
| 2018-01-01 | 179          | 244           | 2229 | 

There have actually only been 15 months where more issues got closed than created, with only two in the past two years.

```
with closed as (
	SELECT date_trunc('month', closed_at)::date as "month", count(*) as "count_closed"
	FROM pandas_issues
	where closed_at is not null
	group by 1
), created as (
	SELECT date_trunc('month', created_at)::date as "month", count(*) as "count_created"
	FROM pandas_issues
	group by 1
)

select
month, coalesce(count_closed, 0) as count_closed, coalesce(count_created, 0) as count_created

from closed full outer join created using("month")
where count_closed > coalesce(count_created, 0)
order by "month" desc;
```


| month        | count_closed | count_created | 
|--------------|--------------|---------------| 
| 2018-07-01 | 273          | 227           | 
| 2017-07-01 | 187          | 181           | 
| 2016-12-01 | 142          | 124           | 
| 2016-09-01 | 137          | 135           | 
| 2016-04-01 | 176          | 171           | 
| 2015-08-01 | 139          | 135           | 
| 2014-02-01 | 190          | 171           | 
| 2013-09-01 | 250          | 175           | 
| 2012-11-01 | 171          | 160           | 
| 2012-06-01 | 151          | 143           | 
| 2012-05-01 | 193          | 155           | 
| 2011-06-01 | 16           | 8             | 
| 2011-03-01 | 1            | 0             | 
| 2011-02-01 | 7            | 1             | 
| 2010-12-01 | 11           | 4             | 


You can do a lot more with this dataset, including looking at issue authors, cross referencing these number with labels, looking at numbers of issue untouched for a given period of time (I like to tackle issues that have not been updated for years, there is a notable chance they've already been resolved) and much more.

I hope this gives you at least some visibility into the issues submitted to the pandas project. Obviously this doesn't all boil down to a single number, or even a series of numbers. Some of this pertains to the forthcoming pandas 1.0 version, new API improvements (nullable int types!) and the dropping of Python 2 support. Last but not least, a lot of issues are feature requests.

But the most notable driving force behind the issue count (discounting really buggy software) is not a package's popularity, it is the API surface a given piece of software offers. E.g. the [requests](https://github.com/requests/requests) package is a lot more popular than pandas, but only has 119 open issues submitted at the time of writing this, but this stems from the fact that the requests API has been pretty stable over the years with no big new features introduced.

Originally, I thought we could organise more and more pandas sprints at PyData events to try and bring down pandas' issue count, but looking at this evidence, I think the primary goal at this point should be to at least stabilise the issue count.
