---
layout: post
title: "Jenkins Build History"
date: '2017-05-03'
author: Juri Timošin
tags:
- jenkins
---

[1]: https://github.com/influxdata/telegraf
[2]: https://github.com/influxdata/influxdb
[3]: https://grafana.com/

Having big Jenkins cluster requires monitoring many things. Lately we started saving information
about what build ran on which machine and when. Jenkins actually provides that feature, this is
called "Build history" and can be seen for the whole cluster or for some particular node.
Unfortunately, when cluster is quite big (ours has more than 300 executors serving more than 10000
builds per day) Jenkins is not able to show the graph.

<!--more-->

So we decided to build it ourselves. We are already using [Telegraf][1] for monitoring Jenkins,
[InfluxDB][2] for storing time series data and [Grafana][3] for displaying the graphs. Jenkins
provides the information we require through API, so all we had to do is to make a new kind of
request to Jenkins Master. The API url is:

{% highlight url %}
http://<jenkinsserver>/computer/api/json?tree=computer[displayName,oneOffExecutors[number,currentExecutable[fullDisplayName]],executors[number,currentExecutable[fullDisplayName]]]
{% endhighlight %}


Instead of json you can use xml too, but I am parsing the result using ruby and it's more
comfortable to parse json in ruby. We have to make a request not only for `executors` - those are
responsible for ordinary jobs, but also for `oneOffExecutors` - those show the pipeline jobs.
`Number` in the `executors` shows the executor number and `currentExecutable.fullDisplayName` shows
job name, build number (and configuration for matrix jobs). Unfortunately `number` in the
`oneOffExecutors` is always -1, but we actually do not care about the executor number, we just need
to know what build runs when and on what machine.

The parsed data gets written into the database so that node, job, build and config are used as tags
and the only metric data is the executor number:

{% highlight %}
jenkins.build,node=marine,job=my-test-job-1,build=132,config=ubuntu\,clean executor=0
jenkins.build,node=tank,job=my-test-job-2,build=12,config=ubuntu executor=-1
{% endhighlight %}

Then in grafana we have to make a query for node and group by all the other tags:

{% highlight sql %}
SELECT mean("executor") FROM "jenkins.build" WHERE "node" =~ /^$node$/ AND $timeFilter GROUP
BY time($interval), "job", "build", "config" fill(none)
{% endhighlight %}

Which gives us a beautiful picture:

![Jenkins Build Timeline in Grafana]({{ site.url }}/assets/grafana-jenkins-build-timeline.png)

where every straight line is 1 build of 1 job.
