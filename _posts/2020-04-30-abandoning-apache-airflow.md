---
title: "Abandoning Apache Airflow"
date: 2020-04-30T07:03:53+02:00
---

I was listening to [this podcast about Prefect](https://softwareengineeringdaily.com/2020/04/29/prefect-dataflow-scheduler-with-jeremiah-lowin/), a dataflow scheduler that tries to learn from the shortcomings in Apache Airflow. It occurred to me that I discussed our decisions to deploy and subsequently abandon Airflow within our company, I actually discussed this on a number of occasions, so I might as well write it up.

### Inception

We had a job scheduler that triggered various jobs in a system that shall remain nameless (to avoid painful memories). One day we realised we don't have to use its enterprise version, because all it brings on top of the community version, was a job scheduler. I could do that in a weekend (c)!

At that time (late 2017), [Apache Airflow](https://airflow.apache.org/) was the go-to solution, the competition - Luigi and Pinball - wasn't as mature or lacked key features. I was a bit worried at the start, because Apache projects lean heavily toward the JVM, but this was Python, which we employed for other tasks, so the initial deployment was super easy. Install a package, link it to (our existing) Postgres instance and off we went.

After tinkering with it for a few days, we set up a Jenkins job to deploy this installation and it lived in our infrastructure for some six months and successfully launched tens of thousands of jobs. And then one day, I deleted it all. Here are the reasons for it. **Tl; dr: it did way more than we needed and the complexity got in our way too often during development.**

### Reloading jobs

Our first hurdle was refreshing the state Airflow operated with. We had all our jobs in a git repo and when we pushed new code, we needed Airflow to reflect it and start scheduling new jobs (or stop scheduling existing ones). That wasn't a solved thing and we ended up ductaping some kill/launch sequence (yes, we turned it off and on again).

This was not only suboptimal from a cleanliness perspective, but it just didn't work right at times - there'd be cruft left in the database, some jobs would be left hanging, it just was a silly solution to a very obvious problem.

### Too much Airflow code in our ETL

One main advantage of Airflow-like systems is that it decouples the tasks, lets you run them, retry them if necessary, and facilitate communication between them (e.g. "I created an S3 object at s3://foo/bar"). Sounds cool, but this turned out to be rather gimmicky and lead to more and more integration of Airflow libraries into our internal ETL code. We wanted to keep things separate, but it was just way too integrated.

Another example is time of execution - instead of using `now()`, you'd be provided with a timestamp for job execution - this was super handy if your job failed and you needed to backfill it - run it with a "fake" timestamp from the past. Works great in theory, but this was such a confusing concept that wasn't quite understood by many (me not excluding) and lead to surprising job invocations and bugs (especially around midnight).

One last hurdle in this regard - and perhaps the biggest - was debugging in local use. I'm a big believer in programs that are buildable and debuggable on local machines. I don't want to spawn 12 Docker containers just to run a job, I want to see what's happening, debug it in a proper way (`print` everywhere), and in general be in control. This was not possible with Airflow, because it took over invocation of jobs and it was very cumbersome to clone and debug.

### Deployment and dependencies

The way we deployed Airflow (and it's probably not the way we'd do it now) was we had a Jenkins job that installed it on a long-running EC2 box and it ran the jobs on that machine as it was (Airflow offered workers separate from the scheduler, but we didn't need that as our jobs were quite lightweight).

Now this worked fine when things were running, but we ran into issues when updating things. Be it Python dependencies or Python itself. Or environment variables. Or the OS. It was this one big persistent thing that operated everything and it was quite difficult to isolate things and to have reproducible runs.

### Lack of integration into our infrastructure

We already had a whole host of technologies within the company for running things (Gitlab, Mesos, Graphite, Grafana, ELK) and our Airflow solution was not integrated with any of those in terms of monitoring, logging, or resource management. Airflow has since gained Kubernetes support, so it's getting better in this regard, but at the time, we were in the dark and had to build our own Slack integration and other reporting, just so that we'd understand what was going on.


### Moment of realisation

At one point we realised on thing - we're not using half of what Airflow offers us and the remaining features are getting in our way instead of helping us. So is there a better way to do this thing? We needed a way to schedule things, have them logged, monitored, dependencies handled etc.

We already had a cron-like solution within our infrastructure, so we just used that. There are no job-dependencies, no inter-process communication, just cron schedules and retries on failures, but it ended up working well, we just had to do a few things to make it work.

### Making all jobs independent

Before Airflow, we'd have to check "is job B running right now? If so, abort", because some jobs couldn't overlap. With Airflow, we'd just stitch them one after the other and it worked well. Since our new solution didn't have this, we had to work around this and it wasn't all that hard in the end. We just made sure that jobs could run concurrently. In the end, we realised that there shouldn't be a reason two jobs cannot run concurrently - if we make all jobs atomic and idempotent (more on that later), then a concurrent job will either see old or new data, but nothing in between, so it doesn't matter what jobs run when.

For this to work, we had to ensure two things:

1. Atomicity - every job would either write everything or nothing. There were no partial results. This is fairly trivial in databases, which are sort of built for this, we had to think about things a bit elsewhere. E.g. in S3, we'd just buffer the result locally or used some flags, or used metadata in a database that'd be written only after persisting data etc.
2. Idempotency - this is quite a key component of any job, regardless of a scheduler. All it says it that it doesn't matter how many times your run a job, it always behaves the same way. This saved us many times.

This also resulted in a lack of cleanups (before that, we'd always have to ensure we'd delete all the partial results that could be there because of a failure halfway through a job) and everything was more reliable.

### Chaining things

This was tricky and, to be honest, the least satisfactory thing in our solution. How do you compose parts of your jobs in a nice way (the whole DAG spiel)? We sort of did that by breaking up our jobs into smaller pieces, which could be run independently and when we needed things to run in a given order, we'd just import these jobs into a tiny orchestrator and run them there.

In order for this to work, we had to have a simple system - each job ultimately has a `main()` function, which gets run when the main file gets invoked. BUT, you can equally have a different file that imports these various main functions and invokes them instead. So we could compose a number of smaller jobs, run them in parallel if we wanted to, but we mostly didn't.

### Conclusion

What we ended up was a much better codebase, not just because we didn't have Airflow cruft in our code, but also because we had to think about building reliable jobs that could overlap or fail at any time. The resulting system was much easier to operate from a devops perspective, had much better reliability, monitoring, and logging - but these were only due to the fact we already had a cron-like system in place. If you don't, your mileage may vary. Also, we didn't have giant dags with dozens of complex relationships - for these use cases, dataflow systems like Airflow are still a good fit.

What I personally took most from this was the fact that this was super easy to debug locally. You just cloned a repo, installed a bunch of dependencies in a virtual environment and you were good to go. You could build a Docker image and you'd see _exactly_ what would happen in production. Most of all, you were just running simple Python scripts, you could introspect it in a million different ways (but, you know, print is the way to go).

I'm not saying people should go this route, but as I gain more experience, I cherish simplicity and debuggability (these two often go hand in hand). This taught me that while Airflow is great and offers a lot of cool features, it might just be in your way.

