---
author: Matt Oswalt
comments: true
date: 2017-07-20 00:00:00+00:00
layout: post
slug: stackstorm-architecture-components
title: 'StackStorm: High-Level Architecture and Components'
categories:
- Blog
tags:
- automation
- stackstorm
---

# Intro

Often, when learning a new technology, it makes sense to break it up into component parts and look at them individually. Then, when you conceptually "re-assemble" them, you know how each part works, and can appreciate the greater system.

StackStorm is very conducive to this mindset because it's already broken up into several subcomponents, that more or less operate independently. If you've ever been involved with OpenStack, you know that OpenStack itself isn't really a tangible thing (there is no "OpenStack Server"), but rather a collection of smaller services that specialize in their area of focus, like networking or storage. Similarly, StackStorm is a collection of smaller components that work together to make event-driven automation possible, and just as important, scalable.

This will be a relatively long post, as there's a lot to talk about. However, I'll try to keep things chunked up nicely so we can reason about each component without getting overwhelmed. That said, it is aimed primarily at developers or sysadmins that **really** want to get into the guts of StackStorm.

> If you haven't read my [previous post on StackStorm concepts(https://keepingitclassless.net/2016/12/introduction-to-stackstorm/), I highly recommend you read that now, or this post won't make a lot of sense. The components we'll discuss here exist to serve those concepts.

Here are some useful links to follow along - this post mainly focuses on the content there, and elaborates:
- [High-Level Overview](https://docs.stackstorm.com/install/overview.html)
- [StackStorm High-Availability Deployment Guide](https://docs.stackstorm.com/reference/ha.html)
- [Code Structure for Various Components in "st2" repo](https://docs.stackstorm.com/development/code_structure.html)


## StackStorm High-Level Architecture

Before diving into specific components, it's important to start at the top; what does StackStorm look like when you initially lift the hood?

The best place to start for this is the [StackStorm Overview](https://docs.stackstorm.com/overview.html), where StackStorm concepts and a very high-level walkthrough of how the components interact is shown.

<div style="text-align:center;"><a href="{{ site.url }}assets/2017/04/architecture_diagram.jpg"><img src="{{ site.url }}assets/2017/04/architecture_diagram.png" width="300" ></a></div>

In addition, the [High-Availability Deployment Guide](https://docs.stackstorm.com/reference/ha.html) (which you should absolutely read if you're serious about deploying StackStorm) contains a much more detailed diagram, showing the actual, individual process that make up a running StackStorm instance:

<div style="text-align:center;"><a href="{{ site.url }}assets/2017/04/components.png"><img src="{{ site.url }}assets/2017/04/components.png" width="300" ></a></div>

> It would be a good idea to keep this diagram open in another tab while you read on, to understand where each component fits in the cohesive whole that is StackStorm

Each component has a specific job to do - and each also needs to communicate with the other components. As you can see in the diagram, many of the components communicate over RabbitMQ. Some also write to a database. These relationships will become more obvious as we explore each component in detail.

## StackStorm Components

Now, we'll dive in to each component individually. Again, the purpose of this deep-dive is to understand what each component does, but keep in mind that all of these components must work together to make StackStorm work. It may be useful to keep the diagram(s) above open in a separate tab, to keep the big picture in mind.

### st2actionrunner

We start off by looking at [`st2actionrunner`](https://docs.stackstorm.com/reference/ha.html#st2actionrunner) because, like the Actions that run inside them, it's probably the most relatable component to those that have automation experience, but are new to StackStorm or event-driven automation in general.

`st2actionrunner` is responsible for receiving execution (an instance of a running action) instructions, scheduling and executing those executions. If you dig into the `st2actionrunner` code a bit, you can see that it's powered by two parts, a [scheduler](https://github.com/StackStorm/st2/blob/master/st2actions/st2actions/scheduler.py#L41), and a [dispatcher](https://github.com/StackStorm/st2/blob/master/st2actions/st2actions/worker.py#L46). The scheduler receives requests for new executions off of the message queue, and works out the details of when and how this action should be run. For instance, there might be a policy in place that is preventing the action from running until a few other executions finish up. Once an execution is scheduled, it is passed to the dispatcher, which actually runs the action with the provided parameters, and retrieves the resulting output.

> You may have also heard the simple term "runners" in StackStorm. In short, you can think of these kind of like "base classes" for Actions. For instance I might have an action that executes a Python script; this action will use the `run-python` runner, because that runner contains all of the repetitive infrastructure needed by all Python-based actions. Please do not confuse this term with the `st2actionrunner` container, as this container is a process, and a "runner" is a Python base class to declare some common foundation for an Action to use. In fact, `st2actionrunner` is indeed [responsible for handing off execution details to the runner](https://github.com/StackStorm/st2/blob/master/st2actions/st2actions/container/base.py#L59), whether it's a Python runner, a shell script runner, etc.

As shown in the component diagram, `st2actionrunner` communicates with both RabbitMQ, as well as the database (which, at this time is MongoDB). RabbitMQ is used to deliver incoming execution requests to the scheduler, and also so the scheduler can forward scheduled executions to the dispatcher. Both of these subcomponents update the database with execution history and status.

### st2api

If you've worked with StackStorm at all (and hopefully if you're this far through this post, you have), you know that StackStorm has an API. External components, such as the CLI client, the Web UI, and chatops all use this API to interact with StackStorm. This API is also where 3rd party software can integrate with StackStorm (with the exception of packs, which usually have a tighter integration at the Python level, and don't need to use the API).

`st2api`, like the aforementioned `st2actionrunner` runs as it's own separate process. To put it very loosely, `st2api` "translates" API calls into RabbitMQ messages or database calls. What's meant by this is that incoming API requests are usually aimed at either retrieving data, pushing new data, or executing some kind of action with StackStorm. All of these things are done on other running processes; for instance, `st2actionrunner` is responsible for actually executing a running action, and it receives those requests over RabbitMQ. So, `st2api` must initially receive such instructions via it's API, and forward that request along via RabbitMQ. Let's discuss how that actually works.

> The 2.3 release changed a lot of the underlying infrastructure for the StackStorm API. The API itself isn't changing (still at v1) for this release, but the way that the API is described within `st2api`, and how incoming requests are routed to function calls has changed a bit. Everything we'll discuss in this section will reflect these changes. Pleaes review [this issue](https://github.com/StackStorm/st2/issues/2686) and [this PR](https://github.com/StackStorm/st2/pull/2727) for a bit of insight into the history of this change.

Before we get into how the the `st2api` component works, it's important to understand the API itself. I wrote a post (Insert reference to separate post on the API itself) that outlines the way that the API is defined as of StackStorm 2.3, and how to figure out exactly what's happening behind the scenes when you run something like `st2 run` from the command-line. Especially if you're interested in contributing to StackStorm, this is a useful post to read.

In short, the API is constructed with Swagger description of all the API endpoints. Using this description each endpoint is linked to its own "implementation function" behind the scenes. These functions may write to a database, they may send a message over the message queue, or they may do both. Whatever's needed in order to implement the functionality offered by that API endpoint.

However, the API still has to be served. There still has to be an HTTP server that will accept incoming requests to the API.

> Note that in a production-quality deployment of StackStorm, the API is front-ended by nginx. We'll be talking about the nginx configuration in another section, so we'll not be discussing it here. But it's important to keep this in mind.

We can use this handy command, filtered through `grep` to see exactly what command was used to instantiate the `st2api` process.

```
vagrant@st2vagrant:~$ ps --sort -rss -eo rss,pid,command | head | grep st2api
64968 12443 /opt/stackstorm/st2/bin/python /opt/stackstorm/st2/bin/gunicorn st2api.wsgi:application -k eventlet -b 127.0.0.1:9101 --workers 1 --threads 1 --graceful-timeout 10 --timeout 30
```

As you can see, it's running on Python, as most StackStorm components run on. Note that this is the distribution of Python in the StackStorm virtualenv, so anything run with this Python binary will already have Python-based dependencies satisfied.

The specific application being run on Python is [Gunicorn](http://gunicorn.org/) (note the reference to `/opt/stackstorm/st2/bin/gunicorn`), which is a WSGI HTTP server. it's used to serve not only StackStorm's API, but also some additional components that we'll get into in future sections. You'll notice that for `st2api`, the 3rd positional argument is actually a reference to a Python variable (remember that this is running from StackStorm's Python virtualenv, so this works). Looking at [the code](https://github.com/StackStorm/st2/blob/master/st2api/st2api/wsgi.py) we can see that this variable is the result of a call out to the setup task for the [primary API application](https://github.com/StackStorm/st2/blob/master/st2api/st2api/app.py), which is where the aforementioned swagger spec is loaded and rendered into actionable HTTP endpoints.

That's about it! `st2api` is actually remarkably simple once you understand how the Swagger spec resolves to actual Python functions (again please read (TODO insert link to post here) for more info on that portion) and how the WSGI server is instantiated.

TODO Add paragraph on webhooks and how trigger/webhook registration is passed here

### st2rulesengine

> evaluates rules when it sees TriggerInstances and decides if an ActionExecution is to be requested. It needs access to MongoDB to locate rules and RabbitMQ to listen for TriggerInstances and request ActionExecutions. The auxiliary purpose of this process is to run all the defined timers.

How does a rule send registrations of triggers to the sensorcontainer? Is this even a thing? What does the communicatio between the rules engine and the sensor container look like?


### Mistral

[Mistral](https://docs.stackstorm.com/mistral.html) is very important because it is StackStorm's primary workflow engine. If you're unfamiliar with Mistral, [it is an OpenStack project](https://wiki.openstack.org/wiki/Mistral) for managing workflows. Since Mistral provides a lot of useful workflow primitives, StackStorm integrates with Mistral, and provides plugins to allow Mistral to execute StackStorm actions.

Mistral is a complex subject on its own, and requires its own blog post (coming soon) for a proper deep-dive. For now, suffice it to say that it behaves much like other StackStorm components, in that it operates as a standalone process, sending and receiving instructions from StackStorm in the execution of workflows.

TODO - Add explanation of how results are tracked after you've explored `st2resultstracker`



## Scaling Out Components

You may have also noticed in the high-level diagram, that for some components, there are multiple blocks overlaid on top of each other. This is intended to describe the horizontally scalable nature of many of StackStorm's components. Note the that we multiple copies of many of the components when we run `st2ctl status`:

```
vagrant@st2vagrant:~$ st2ctl status
##### st2 components status #####
st2actionrunner PID: 4467
st2actionrunner PID: 4470
st2api PID: 1386
st2api PID: 10663
st2stream PID: 1377
st2stream PID: 10678
st2auth PID: 1378
st2auth PID: 10658
st2garbagecollector PID: 4508
st2notifier PID: 4515
st2resultstracker PID: 4522
st2rulesengine PID: 4529
st2sensorcontainer PID: 4536
st2chatops PID: 4545
mistral-server PID: 4572
mistral-api PID: 4562
mistral-api PID: 11936
mistral-api PID: 11937
```

This is because several of the StackStorm components can (and often do) have multiple running instances of itself, for load balancing purposes. For instance, the more `st2actionrunner` instances running at a time, the more executions can be handled at once. It also provides a bit of high-availability, as these processes can/should run on separate physical or virtual machines.

When components wish to communicate with each other, they send messages using a specific type of RabbitMQ Exchange known as a `topic` exchange. This allows RabbitMQ to route incoming messages to the appropriate queue based on a routing key. RabbitMQ will know how many instances of a similar type are online and listening for messages, and will ensure incoming work is distributed evenly between them.

Scaling out the number of `st2actionrunner` instances is a very frequent question in the [StackStorm slack community](https://stackstorm.typeform.com/to/K76GRP), or [via email](support@stackstorm.com) - because the number of running `st2actionrunner` processes impacts how many actions StackStorm can process in a given time. This will vary by operating system, since this is done at the init system level. For instance, Ubuntu 14.04 uses "upstart", so we can look at `/etc/init/st2actionrunner.conf` to see how this done:

```
vagrant@st2vagrant:~$ cat /etc/init/st2actionrunner.conf
# StackStorm actionrunner main task. Starts up multiple workers.
#

...omitted for brevity...

pre-start script
  NAME=st2actionrunner
  WORKERS=$(/usr/bin/nproc 2>/dev/null)
  WORKERS=${WORKERS:-2}

  # Read configuration variable file if it is present
  [ -r /etc/default/$NAME ] && . /etc/default/$NAME
  for i in `seq 1 $WORKERS`; do
    start st2actionrunner-worker WORKERID=$i || :
  done
end script

...omitted for brevity...
```

Note the `WORKERS` variable. The two lines where this variable is defined is a bit of bash-foo that first tries to set WORKERS to the number of processors in the system (see `man nproc`), or if for some reason that doesn't return anything, it defaults to 2. So, if you instead set WORKERS to 3 on the next line, and restart StackStorm components with `st2ctl restart`, you'll end up with 3 instances of `st2actionrunner`.

Other StackStorm components also have their own init system configuration file and can be scaled out similarly.
