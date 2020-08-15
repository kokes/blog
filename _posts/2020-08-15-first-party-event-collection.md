---
title: "Painless first party event collection"
date: 2020-08-15T07:03:53+02:00
---

Whenever users interact with your sites, you almost automatically find out certain information. What pages they loaded, what assets they requested, what page they got to your site from etc. But sometimes you need more granular information, some that cannot be read from your server logs. This is usually what Google Analytics will give you. While Google gives you some aggregated statistics, you may want to dig a little deeper and perhaps roll your own solution.

Before we go any further, I want to clarify that this is not about _tracking_, the solution offered here doesn’t really take into account any user identification and does not track anyone across sites. This is to find out if your users actually read your articles, if they open modals, how long it takes them to discover certain features, if they click on things that are not clickable, for debugging purposes etc.

We used to run an off the shelf commercial product, think Heap or Mixpanel. We eventually decided to migrate for the following reasons:

1. Warehouse duplication - we were paying for data being stored in an alternative data warehouse together with its own visualisation. But we had our own warehouse and BI tools, so it didn’t make sense to run this, do ETL, validation, fix pipelines whenever the format changed etc.
2. Blocking - the solution used a vendor’s piece of JavaScript, so it was blocked by a non-trivial number of our customers. We ran into this issue even within the company, when people complained they couldn’t their own actions in the system.
3. Non-linear scaling - the solution wasn’t that expensive, but if we wanted to triple our event count, we suddenly got a 10x bill. So we had to work around this by deleting a lot of events, which made the whole thing counterproductive.
4. Ownership of data and schema - this wasn't a terribly sensitive dataset, but it did make sense to own it, not just for its value, but also for the stability of the schema and data model.

## Rolling your own solution

There are just a few parts to this:

1. Frontend data collection
2. Backend data persistence
3. Visualisation and analytics

We won’t cover the third part, that’s a whole can of worms and companies usually have some solution that works for them (e.g. Tableau, Looker).

First let's build a simple frontend site. This won't really do anything fancy, it will just report any click on any button on the site. You can attach these events pretty much to anything, including mouse moves.

```html
<!DOCTYPE html>
<html>
    <head><title>My Site</title></head>

    <body>
        <button>Click me!</button>
        <button>Click me as well!</button>


        <script type='text/javascript'>
            // common functionality across all event logs
            async function sendEvent(eventName, properties) {
                const event = {
                    event: eventName,
                    url: window.location.href,
                    properties: properties,
                }
                return fetch('/events', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify(event),
                })

            }
            // now let's register all event calls
            const buttons = document.getElementsByTagName('button');
            for (let button of buttons) {
                button.addEventListener('click', (e) => {
                    const properties = {
                        'button_text': e.target.textContent,
                    }
                    sendEvent('clicked_event', properties);
                })
            }
        </script>
    </body>
</html>
```

What this will do is that it will send a POST request to your backend, so let's deal with that. Here's the poorest man's Node.js server that handles these incoming events.

```javascript
const http = require('http');
const fs = require('fs');

const listener = function (req, res) {
    // serve the frontend
    if (req.url === '/' && req.method === 'GET') {
        fs.readFile('index.html', function (err, data) {
            if (err) {
                res.writeHead(404);
                return;
            }
            res.writeHead(200, {'Content-Type': 'text/html'});
            res.end(data, 'utf-8')
        });
        return;
    }
    // fail on anything but the frontend and event handling
    if (!(req.url === '/events' && req.method === 'POST')) {
        res.writeHead(400);
        res.end();
        return;
    }
    // accumulate data and parse it once it all comes in
    let data = []
    req.on('data', chunk => {
        data.push(chunk)
    })
    req.on('end', () => {
        const event = JSON.parse(data);
        event.server_time = Date.now();
        console.log(event);
        // send the event to your backend
    })

    res.writeHead(202);
    res.end();
}

const server = http.createServer(listener);
server.listen(8080);
```

As you can see, the server just accepts the JSON and adds a timestamp. This is to ensure that we have a credible source of _event time_. This is a term in stream processing and while we don't strictly adhere to the definition - the event happened in the browser, not on the backend - we cannot get a reliable timestamp from the frontend, so this will have to do. Google "event time vs. processing time" to find out more about this.

The only thing we didn't implement in this server was the actual event handling. This really depends on what existing infrastructure you already have and also what kind of traffic you expect.

## Handling event processing

For simple sites with not much traffic, I'd just insert this event to a relational database system (Postgres, MySQL etc.). Modern RDBMS can handle JSON data just fine, you can then have some simple ETL process to extract some of the bits in the events into relational objects, but all in all, the pipeline will remain quite simple.

If you anticipate more traffic and/or you want to decouple your application from your database, there are a few alternatives. My favourite, and the one we used in production, was that you simply send the event to a message queue - not Kafka or Pulsar, plain AMQP like RabbitMQ or ActiveMQ. These can handle most workloads and they are simple, battle tested and, more importantly, already common within infrastructures.

Now it's completely up to you to handle this stream. We had three independent processes:

1. Collect the data and save it to S3 (as a `.json.gz`) once we reach `n` events or `m` seconds, whicever comes first. This way we don't depend on the stream to be the durable part of the architecture, data are safely stored in our blob storage.
2. From time to time, go and compact those S3 blobs into larger files. It's not the best idea to have a bunch of small files on S3, so we combined them into larger (hourly) chunks.
3. Load some of the S3 data into a database, so that we can readily query it.

This was a nicely working pipeline - we leveraged existing technologies across the whole stack, there was miniscule utilisation of resources (< 1%), we could easily scale this 100x without worrying about a single part of the pipeline.

The collection endpoint could be easily load balanced, the message queue had enough capacity, S3 scales beyond what we'd ever need, and the database could be swapped for Presto, Drill, Athena, BigQuery or anything of this sort - we already have the data in blob storage, so it's readily available for these big data processing tools.

## Conclusion

This is not the only solution, obviously. Common alternatives are that you collect your data in the fronend using "your" library (it's bundled within your JavaScript, but it's really your vendor's code), send it to your backend and then log it to your vendors APIs from there, bypassing the frontend blockers.

This only solves one of the problems I outlined, that's why we went another route. But your mileage may vary! Also, I omitted a number of things - error handling, uuid generation, validation, ... - this only serves to illustrate the principles of our solution.

There are a number of things I like about this solution:

1. It's super simple in terms of architecture.
2. It's composed, not inherited, so parts can easily be swapped, added, removed.
3. It scales way beyond what we need.

I hope you find value in this.
