---
title: "Assorted tech links #1"
date: 2019-05-08T07:03:53+02:00
draft: false
---

I do quite a bit of reading and watching of tech talks and since I often forget things, I decided to turn this into a sort of bookmarking service.

---

- A good talk [on the basics of open source contribution](https://www.youtube.com/watch?v=JhPC6_rO08s) - what to do, how to get that first pull request submitted etc. Paul Ganssle is the maintainer of [dateutil](https://dateutil.readthedocs.io/en/stable/) (its parser is superb) and [a contributor to CPython](https://mail.python.org/pipermail/python-committers/2019-February/006567.html).
- Service meshes (if that's the correct plural) are all the rage these days. And here's a [nice talk](https://www.youtube.com/watch?v=55yi4MMVBi4) by the author of [Envoy](https://www.envoyproxy.io/), which is quickly becoming the goto service mesh out there (Google announced a hosted version recently). If you like podcasts, check out [this episode](https://softwareengineeringdaily.com/2018/12/19/linkerd-service-mesh-with-william-morgan/) of Software Engineering Daily, it's about Linkerd.
- With all those cutting edge technologies, it's quite refreshing to listen to a podcast about [coding for the ZX Spectrum](https://hanselminutes.com/670/coding-for-the-zx-spectrum-and-netflixblack-mirrors-bandersnach-with-matt-westcott), specifically for an episode of Black Mirror (an excellent TV series, by the way).
- One of the most cited issues with pandas has been its inability to contain integer series with nulls. This lack of support is caused by the treatment of nulls in series - pandas uses sentinel values rather than a separate bitarray. Here is [a good overview of what's changed recently](https://www.youtube.com/watch?v=gxvTVxlvH9w) by Jeff Reback, pandas' long time maintainer.
- Postgres is my goto database and even after more than 20 years of development, there are new features introduced every year. The latest version, 11, is mostly focused around better partitioning and parallelisation. I liked two videos on it - Magnus Hagander [mostly focuses on features](https://www.youtube.com/watch?v=lgzXuFQ0Pbk), while Joe Conway [covers how development is done as well](https://www.youtube.com/watch?v=kPhs-wdrb58).
