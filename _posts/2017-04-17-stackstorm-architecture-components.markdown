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

### st2api

If you've worked with StackStorm at all (and hopefully if you're this far through this post, you have), you know that StackStorm has an API. External components, such as the CLI client, the Web UI, and chatops all use this API to interact with StackStorm. This API is also where 3rd party software can integrate with StackStorm (with the exception of packs, which usually have a tighter integration at the Python level, and don't need to use the API).

`st2api`, like the aforementioned `st2actionrunner` runs as it's own separate process. To put it very loosely, `st2api` "translates" API calls into RabbitMQ messages or database calls. What's meant by this is that incoming API requests are usually aimed at either retrieving data, pushing new data, or executing some kind of action with StackStorm. All of these things are done on other running processes; for instance, `st2actionrunner` is responsible for actually executing a running action, and it receives those requests over RabbitMQ. So, `st2api` must initially receive such instructions via it's API, and forward that request along via RabbitMQ. Let's discuss how that actually works.

> The upcoming 2.3 release changes a lot of the underlying infrastructure for the StackStorm API. The API itself isn't changing (still at v1) for this release, but the way that the API is described within `st2api`, and how incoming requests are routed to function calls has changed a bit. Everything we'll discuss in this section will reflect these changes.

First, we should talk about the API itself. The API has been defined using an API domain-specific language known as Swagger, which was created to separate the API definitions from the actual implementation in a more-or-less standardized way. This means that any API written in Swagger can take advantage of tooling that can produce generated code, as well as automatic documentation from this definition.

This Swagger definition is [located here](https://github.com/StackStorm/st2/blob/master/st2common/st2common/openapi.yaml). It plainly defines each API endpoint (like `/api/v1/actions`), what parameters are required by each, what they'll return in the form of a response, etc. While the actual implementation logic that works behind the scenes of each of these endpoints is not shown, it's easy to see at a high level what the API does.

However, just because the implementation is separated from the definition of the API, this does not mean we can't dig into the implementation when we need to. You may notice the `operationId` key in the Swagger definition. This is the link between the language-agnostic API definition and the actual implementation of that API endpoint. In the case of `st2api`, the key `st2api.controllers.v1.actions:actions_controller.post` literally points to the `ActionsController.post()` function in [`actions.py`](https://github.com/StackStorm/st2/blob/master/st2api/st2api/controllers/v1/actions.py). So, even though the Swagger description is designed to omit any implementation details, it's trivial to find where this implementation is taking place.

When you drill into the implementation, it becomes a little more obvious what's going on. Continuing to pick on the `post` implementation for `/api/v1/actions`, we can see that the primary goal of the function that sits behind this API endpoint is to receive an incoming action definition and write it to the database. Effectively, we're seeing what actually happens on the back-end when we add a new action with the StackStorm CLI:

```
vagrant@st2vagrant:/opt/stackstorm/packs/core/actions$ st2 --debug action create local.yaml
2017-04-13 22:40:42,012  DEBUG - Using cached token from file "/home/vagrant/.st2/token-st2admin"
# -------- begin 140451057640144 request ----------
curl -X POST -H  'Connection: keep-alive' -H  'Accept-Encoding: gzip, deflate' -H  'Accept: */*' -H  'User-Agent: python-requests/2.11.1' -H  'content-type: application/json' -H  'X-Auth-Token: fffffffffffffffff' -H  'Content-Length: 337' --data-binary '{"description": "Action that executes an arbitrary Linux command on the localhost.", "parameters": {"cmd": {"required": true, "type": "string", "description": "Arbitrary Linux command to be executed on the local host."}, "sudo": {"immutable": true}}, "enabled": true, "name": "local", "entry_point": "", "runner_type": "local-shell-cmd"}' http://127.0.0.1:9101/v1/actions
...
```

As a second example, we can use the same logic to find the function that actually schedules an execution when we run `st2 run`. Just run `st2 --debug core.local date` for a simple, quick example. The output contains several API calls, actually, but the interesting one is the POST to `/v1/executions`. Looking back at our Swagger definition, we see this references the key `st2api.controllers.v1.actionexecutions:action_executions_controller.post`. Follow that to the function `ActionExecutionsController.post()`, and this is where the execution is scheduled. This includes forwarding the request to RabbitMQ so that the `st2actionrunner` processes can receive and honor this request, as well as updating the database as necessary.

> One thing I found very useful and interesting exploring this part of the codebase is the use of models for any data that goes over the message queue, or to the database. All messages are standardized in a model, which is important for re-usability but it also makes it easy to repeatably produce wire formats like JSON when it comes time to actually send a message.

As it is often said, "the devil is in the details", and there's a lot of details to be explored here. There's a lot of code sitting behind statements like "the request is forwarded to the message queue" - often, if you try to actually walk the code down to the request actually being sent, you'll be going down a rabbit hole of parent classes after parent class. However, using the information in this section, you should be able to start with what you see on the CLI, and chase that all the way down to the implementation. The rest is up to you.






