---
author: Matt Oswalt
comments: true
date: 2017-04-17 00:00:00+00:00
layout: post
slug: stackstorm-architecture-components.markdown
title: 'StackStorm: Architecture and Components'
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

## StackStorm High-Level Architecture

Before diving into specific components, it's important to start at the top; what does StackStorm look like when you initially lift the hood?

The best place to start for this is the [StackStorm Overview](https://docs.stackstorm.com/overview.html), where StackStorm concepts and a very high-level walkthrough of how the components interact is shown.

<div style="text-align:center;"><a href="{{ site.url }}assets/2017/04/architecture_diagram.jpg"><img src="{{ site.url }}assets/2017/04/architecture_diagram.png" width="300" ></a></div>

In addition, the [High-Availability Deployment Guide](https://docs.stackstorm.com/reference/ha.html) (which you should absolutely read if you're serious about deploying StackStorm) contains a much more detailed diagram, showing the actual, individual process that make up a running StackStorm instance:

<div style="text-align:center;"><a href="{{ site.url }}assets/2017/04/components.png"><img src="{{ site.url }}assets/2017/04/components.png" width="300" ></a></div>

> It would be a good idea to keep this diagram open in another tab while you read on, to understand where each component fits in the cohesive whole that is StackStorm

Each component has a specific job to do - and each also needs to communicate with the other components. As you can see in the diagram, many of the components communicate over RabbitMQ. Some also write to a database. These relationships will become more obvious as we explore each component in detail.

## Common Explanations

There are a few things that apply to all or most of the components in StackStorm, and it's worth going into detail on these before diving into specific components.

### Scaling Out Components

**(Promote this header if this ends up being the only common thing)**

You may have also noticed in the diagram, that for some components, there are multiple blocks overlaid on top of each other. This is intended to describe the horizontally scalable nature of many of StackStorm's components. Note the output when we run `st2ctl status` to get a handle on the running StackStorm processes:

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

## StackStorm Components

Now, we'll dive in to each component individually. Again, the purpose of this deep-dive is to understand what each component does, but don't lose sight of the end goal, which is that all these components work together to make StackStorm work. Keep the diagram(s) above open in a separate tab if you want, so you can keep the big picture in mind.

### st2actionrunner

We start off by looking at [`st2actionrunner`](https://docs.stackstorm.com/reference/ha.html#st2actionrunner) because, like the Actions that run inside them, it's probably the most relatable component to those that have automation experience, but are new to StackStorm or event-driven automation in general.

`st2actionrunner` is responsible for receiving execution (an instance of a running action) instructions, scheduling and executing those executions. If you dig into the `st2actionrunner` code a bit, you can see that it's powered by two parts, a [scheduler](https://github.com/StackStorm/st2/blob/master/st2actions/st2actions/scheduler.py#L41), and a [dispatcher](https://github.com/StackStorm/st2/blob/master/st2actions/st2actions/worker.py#L46). The scheduler receives requests for new executions off of the message queue, and works out the details of when and how this action should be run. For instance, there might be a policy in place that is preventing the action from running until a few other executions finish up. Once an execution is scheduled, it is passed to the dispatcher, which actually runs the action with the provided parameters, and retrieves the resulting output.

> You may have also heard the simple term "runners" in StackStorm. In short, you can think of these kind of like "base classes" for Actions. For instance I might have an action that executes a Python script; this action will use the `run-python` runner, because that runner contains all of the repetitive infrastructure needed by all Python-based actions. Please do not confuse this term with the `st2actionrunner` container, as this container is a process, and a "runner" is a Python base class to declare some common foundation for an Action to use. In fact, `st2actionrunner` is indeed [responsible for handing off execution details to the runner](https://github.com/StackStorm/st2/blob/master/st2actions/st2actions/container/base.py#L59), whether it's a Python runner, a shell script runner, etc.

As shown in the component diagram, `st2actionrunner` communicates with both RabbitMQ, as well as the database (which, at this time is MongoDB). RabbitMQ is used to deliver incoming execution requests to the scheduler, and also so the scheduler can forward scheduled executions to the dispatcher. Both of these subcomponents update the database with execution history and status.

