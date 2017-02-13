---
layout: post
title: How to Run Dynamic Cloud Tests with 800 Tomcats, Amazon EC2, Jenkins and LiveRebel
date: '2012-09-10T11:46:00.001+03:00'
author: Juri Timošin
tags:
- Amazon EC2
- LiveRebel
- Knife
- Chef
- jenkins
- Continuous Integration
modified_time: '2012-09-10T11:56:30.986+03:00'
blogger_id: tag:blogger.com,1999:blog-360329120074358364.post-3278469963020911578
blogger_orig_url: http://dracoater.blogspot.com/2012/09/how-to-run-dynamic-cloud-tests-with-800.html
---

[1]: http://liverebel.com/
[2]: http://wiki.opscode.com/display/chef/Knife
[3]: http://www.opscode.com/chef/
[4]: http://jenkins-ci.org/
[5]: http://www.cloudbees.com/why-do-you-like-jenkins.cb
[6]: https://github.com/DracoAter/lr-cluster
[7]: https://github.com/opscode/knife-ec2/

I was brainstorming in the shower the other day, and I thought "Eureka!" - I need to bootstrap and
test my Java app on a dynamic cluster with 800 Tomcat servers right now! Then, breakfast.

<!--more-->

<img alt="" class="align-left" height="240" src="http://images.sodahead.com/polls/000513339/polls_onemillioncats1_5840_922377_poll_xlarge.jpeg" style="float: left; margin: 5px;" width="320" />

Obviously, every now and then you need to build a dynamic cluster of 800 Tomcat machines and then
run some tests. Oh, wait, you don’t? Well, lets say you do. Provisioning your machines on the cloud
for testing is a great way to "exercise" your app and work on:

- **Warming up**: Bootstrap a clean slate, install the software, run your tests
- **Checking your Processes**: Smoke testing for deploying the app to production
- **Ensuring success**: Checking load handling before launching the application to real clientelle
- **Leaving nothing behind**: After you've got all green lights, shut it all down and watch it disappear

At ZeroTurnaround, we need this for testing [LiveRebel][1] with larger deployments. LiveRebel is a
tool to manage JEE production and QA deployments and online updates. It is crucial that we support
large clusters of machines. Testing such environments is not an easy task but luckily in 2012 it is
not about buying 800 machines but only provisioning them in the cloud for some limited time. In
this article I will walk you through setting up a dynamic Tomcat cluster and running some simple
tests on them. (Quick note: When I started writing this article, we had only tested this out with
100 Tomcat machines, but since then we grew to be able to support 800 instances with LiveRebel and
the other tools).

## Technical Requirements

Let me define a bunch of non-functional requirements that I've thought up. The end result should
have 800 Tomcat nodes, each configured with LiveRebel. A load balancer should sit in front of the
nodes and provide a single URL for tests, and we'll use the LiveRebel Command Center to manage our
deployments.

Naturally, this is all easier said than done. The hurdles that we will need to overcome to achieve
this are:

- Provisioning all the nodes - starting/stopping Amazon EC2 instances
- Installing required software - Java, Tomcat, LiveRebel
- Configuring Tomcat cluster - (think jvmRoute param in `conf/server.xml`)
- Configuring a load balancer (Apache) with all the provisioned IP addresses
- Automation - one-click provision/start/stop/terminate on the cluster using Jenkins

## Tools

We chose Amazon AWS as our cloud provider, namely because we've become familiar with them over the
last couple years. For provisioning we use [Knife][2] and for configuration management we like
[Chef][3]. For automation, we went with [Jenkins][4] (I love [Jenkins][5]), and we have two jobs.
One to start the cluster and one to stop the cluster. Tests are not automated at the moment. Before
going further you have to have a Chef server running on some machine (it should not be necessarily
your own workstation) and Knife installed and configured on your Jenkins machine.

## Architecture

### Loadbalancer

First we have to create/launch a loadbalancer instance. Software to configure:

- Install Apache
- Install/enable Apache loadbalancer module
- Update Apache configuration
- Install LiveRebel Command Center and start it (could be a separate machine but we’ll use this
instance for 2 services)

The load balancer should check in with chef-server and provide his own IP address. LiveRebel
Command Center should be running and accepting incoming requests on default port (9001).

### LiveRebel node(s)

As soon as the load balancer is ready we will create/launch nodes. Node instances need to:

- Install a Tomcat instance
- Figure out the IP of the load balancer
- Download lr-agent-installer.jar from the LiveRebel CC
- Run it (`java -jar lr-agent-installer.jar`)
- Start Tomcat

Again, when everything is done the node will check in with chef-server and provide its IP address.
After all nodes are ready we must update the Apache load balancer configuration and provide all the
IP addresses of the nodes. This is because of the architecture of the load balancer. It needs to
know the IP addresses of the machines it balances.

## Code (The Fun Part)

As we are using Chef, the natural way to act is to create several cookbooks and a couple of roles,
that will help us with configuration. There are four cookbooks in total, one for each application:
Apache, lrcc, Tomcat and Java. You can [get familiar with them on Github][6]. The code is provided
more just for information, because it will not run as is. There are some download links missing.
Another thing is that it was tested only on Ubuntu, so if you are using some other distribution,
you may need to tune it up.

We are going to use Knife command line tool to start and bootstrap our instances. Don’t forget to
[install and configure Knife-EC2][7] rubygem. First step is to create the load balancer. Provided
you have configured the Knife EC2 plugin and prepared the right AMI to launch (or use the default
provided by Ubuntu) it is relatively easy, just run (with right parameters):

{% highlight bash %}
knife ec2 server create --run-list role[lr-loadbalancer] --node-name ‘loadbalancer’
{% endhighlight bash %}

When the process finishes successfully you can go to https://your-server-address:9001/ and check if
the LiveRebel is running. It should be, but you will have to register or provide a license file. If
you already have a license file you can automate the registration step by copying the license into
LiveRebel home folder in your cookbook. Another thing to check is - if the load balancer has
registered with chef-server.

Next step - creating lr-nodes. Your 800 nodes can be created using a similar Knife EC2 command run in loop:

{% highlight bash %}
for i in {1..800} ; do
    knife ec2 server create --run-list role[lr-node] --node-name ‘lr-node-$i’
done
{% endhighlight bash %}

Everything is almost ready! All we need now is to create Jenkins jobs. The first one - we’ll name
it lr-cluster-create - should run these 2 commands and start the cluster. And the other one
lr-cluster-delete - stops it with these commands:

{% highlight bash %}
ids=`knife ec2 server list | grep "lr-node|loadbalancer" | awk {'print $1'} | tr 'n' ' '`
knife ec2 server delete $ids --yes
knife node bulk delete "lr-node-.*" --yes
knife client bulk delete "lr-node-.*" --yes
knife node delete "loadbalancer" --yes
knife client delete "loadbalancer" --yes
{% endhighlight bash %}

## Conclusions

At this point, you should be well on your way towards bootstrapping a clean environment, installing,
running your tests, checking load handling, and then you can shut it all down once you've seen
everything working to your satisfaction.

Your two Jenkins jobs are now able to spawn a dynamic Tomcat cluster. You can even parameterize your
job and supply a number of nodes that you are interested in for a really dynamic cluster.

One thing to note is that as in EC2, Amazon charges for EBS snapshots, so it is not very
cost-effective to just stop the cluster. Termination here will save you some money, especially if
you like bigger clusters.

Another thing is provisioning. Parallel provisioning for the 800 nodes takes roughly 30 minutes.
Starting a new instance from AMI takes some time, but most of it goes to bootstrapping the clean
environment with the Chef installation and downloading packages and archives.

Once you have the cluster started you still need to run tests. We test deploying, updating the
whole cluster with LiveRebel. You could be testing your own WEB application and see how it handles
the load.

The next steps for us is to automate the test suite and have these large scales tests executed
regularly. This will give us valuable feedback about releases in progress and their scalability.

I hope this article has helped you get started with dynamic Tomcat clusters and I’m more than happy
to go into more detail about any step here if you have questions - just contact me.

