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

### Seeing Commands for StackStorm Components

`ps --sort -rss -eo rss,pid,command | head` is useful for outputting the processes using the most memory on the system, and the command that was run to start them. On a typical StackStorm system, these will be mostly StackStorm processes - so you can quickly see at a glance the commands that were run to spin up the various StackStorm components.

```
vagrant@st2vagrant:~$ ps --sort -rss -eo rss,pid,command | head
  RSS   PID COMMAND
83708  8573 /usr/bin/mongod --config /etc/mongod.conf
82180 14511 /opt/stackstorm/mistral/bin/python /opt/stackstorm/mistral/bin/gunicorn --log-file /var/log/mistral/mistral-api.log -b 127.0.0.1:8989 -w 2 mistral.api.wsgi --graceful-timeout 10
82148 14514 /opt/stackstorm/mistral/bin/python /opt/stackstorm/mistral/bin/gunicorn --log-file /var/log/mistral/mistral-api.log -b 127.0.0.1:8989 -w 2 mistral.api.wsgi --graceful-timeout 10
76940  7595 /usr/lib/erlang/erts-5.10.4/bin/beam.smp -W w -K true -A30 -P 1048576 -- -root /usr/lib/erlang -progname erl -- -home /var/lib/rabbitmq -- -pa /usr/lib/rabbitmq/lib/rabbitmq_server-3.2.4/sbin/../ebin -noshell -noinput -s rabbit boot -sname rabbit@st2vagrant -boot start_sasl -kernel inet_default_connect_options [{nodelay,true}] -rabbit tcp_listeners [{"127.0.0.1",5672}] -sasl errlog_type error -sasl sasl_error_logger false -rabbit error_logger {file,"/var/log/rabbitmq/rabbit@st2vagrant.log"} -rabbit sasl_error_logger {file,"/var/log/rabbitmq/rabbit@st2vagrant-sasl.log"} -rabbit enabled_plugins_file "/etc/rabbitmq/enabled_plugins" -rabbit plugins_dir "/usr/lib/rabbitmq/lib/rabbitmq_server-3.2.4/sbin/../plugins" -rabbit plugins_expand_dir "/var/lib/rabbitmq/mnesia/rabbit@st2vagrant-plugins-expand" -os_mon start_cpu_sup false -os_mon start_disksup false -os_mon start_memsup false -mnesia dir "/var/lib/rabbitmq/mnesia/rabbit@st2vagrant"
75196 14462 /opt/stackstorm/mistral/bin/python /opt/stackstorm/mistral/bin/mistral-server --server engine,executor --config-file /etc/mistral/mistral.conf --log-file /var/log/mistral/mistral-server.log
64968 12443 /opt/stackstorm/st2/bin/python /opt/stackstorm/st2/bin/gunicorn st2api.wsgi:application -k eventlet -b 127.0.0.1:9101 --workers 1 --threads 1 --graceful-timeout 10 --timeout 30
63144 11618 /opt/stackstorm/st2/bin/python /opt/stackstorm/st2/bin/gunicorn st2auth.wsgi:application -k eventlet -b 127.0.0.1:9100 --workers 1 --threads 1 --graceful-timeout 10 --timeout 30
62204 12384 /opt/stackstorm/st2/bin/python /opt/stackstorm/st2/bin/gunicorn st2stream.wsgi:application -k eventlet -b 127.0.0.1:9102 --workers 1 --threads 10 --graceful-timeout 10 --timeout 30
53836 11541 /opt/stackstorm/st2/bin/python /opt/stackstorm/st2/bin/st2actionrunner --config-file /etc/st2/st2.conf
```

Tweak the `head` command to see more/less.

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

Before we get into how the the `st2api` component works, it's important to understand the API itself. I wrote a post (Insert reference to separate post on the API itself) that outlines the way that the API is defined as of StackStorm 2.3, and how to figure out exactly what's happening behind the scenes when you run something like `st2 run` from the command-line. Especially if you're interested in contributing to StackStorm, this is a useful post to read.

In short, the API is constructed with Swagger description of all the API endpoints. Using this description each endpoint is linked to its own "implementation function" behind the scenes. These functions may write to a database, they may send a message over the message queue, or they may do both. Whatever's needed in order to implement the functionality offered by that API endpoint.

However, the API still has to be served. There still has to be an HTTP server that will accept incoming requests to the API.

> Note that in a production-quality deployment of StackStorm, the API is front-ended by nginx. We'll be talking about the nginx configuration in another section, so we'll not be discussing it here. But it's important to keep this in mind.

We can use the aforementioned handy `ps` command, filtered through `grep` to see exactly what command was used to instantiate the `st2api` process.

```
vagrant@st2vagrant:~$ ps --sort -rss -eo rss,pid,command | head | grep st2api
64968 12443 /opt/stackstorm/st2/bin/python /opt/stackstorm/st2/bin/gunicorn st2api.wsgi:application -k eventlet -b 127.0.0.1:9101 --workers 1 --threads 1 --graceful-timeout 10 --timeout 30
```

As you can see, it's running on Python, as most StackStorm components run on. Note that this is the distribution of Python in the StackStorm virtualenv, so anything run with this Python binary will already have Python-based dependencies satisfied.

The specific application being run on Python is [Gunicorn](http://gunicorn.org/) (note the reference to `/opt/stackstorm/st2/bin/gunicorn`), which is a WSGI HTTP server. it's used to serve not only StackStorm's API, but also some additional components that we'll get into in future sections. You'll notice that for `st2api`, the 3rd positional argument is actually a reference to a Python variable (remember that this is running from StackStorm's Python virtualenv, so this works). Looking at [the code](https://github.com/StackStorm/st2/blob/master/st2api/st2api/wsgi.py) we can see that this variable is the result of a call out to the setup task for the [primary API application](https://github.com/StackStorm/st2/blob/master/st2api/st2api/app.py), which is where the aforementioned swagger spec is loaded and rendered into actionable HTTP endpoints.

That's about it! `st2api` is actually remarkably simple once you understand how the Swagger spec resolves to actual Python functions (again please read (TODO insert link to post here) for more info on that portion) and how the WSGI server is instantiated.


### Mistral

[Mistral](https://docs.stackstorm.com/mistral.html) is very important because it is StackStorm's primary workflow engine. If you're unfamiliar with [Mistral](https://wiki.openstack.org/wiki/Mistral), it is an OpenStack project for managing workflows. Since Mistral provides a lot of useful workflow primitives, StackStorm integrates with Mistral, and provides plugins to allow Mistral to execute StackStorm actions.

Similar to StackStorm itself, we're not just talking about one component when we're talking about Mistral. Mistral itself is made up of two separate running processes:

- **`mistral-api`** allows 3rd-party systems to execute workflows, get execution status, etc.
- **`mistral-server`** schedules and runs workflows and tasks

Before looking at how Mistral and StackStorm interoperate, let's spend some time diving into these components, and understanding Mistral's architecture on it's own. [The Mistral Architectural Overview](https://docs.openstack.org/developer/mistral/architecture.html) page is a very useful place to start understanding this. Please review at least this page before continuing, as we'll be referring to certain components later.

If you're just getting started with Mistral and want to explore, you should start looking at the `mistral` tool. This is the primary mechanism for CLI interactions with Mistral, and it's one of the best ways to troubleshoot or just explore how Mistral works. As with many CLI tools, `mistral -h` shows the commands you can run and a description of each. If you're familiar with the StackStorm CLI client `st2`, much of the terminology will be familiar to you.

The focus of this section, however, is StackStorm integrates with Mistral, so let's dive into that. StackStorm treats mistral workflows in a way that is similar to how it treats Python script actions. There's a YAML metadata file, and a "script" file. Where Python actions have a Python script, `mistral` actions refer to a workflow definition, which also happens to be written in YAML.

Because of this, StackStorm ships with a [`mistral-v2` runner](https://github.com/StackStorm/st2/blob/master/contrib/runners/mistral_v2/mistral_v2.py), which is invoked whenever an action refers out to a mistral workflow. Looking at the code, we can see that the runner performs several tasks in order to run Mistral workflows, such as translating between StackStorm-specific resources and Mistral actions, as well as actually sending the workflow to Mistral for processing. This isn't a code-focused post, so we'll leave the rest as an exercise for the reader; however, we'll see some of the effects of these tasks shortly. For now, suffice it to say that this runner should be the first place you look to see how StackStorm and Mistral talk to each other.

> Note that the `mistral-v2` runner is only focused on executing a workflow. The runner returns immediately after successfully sending the workflow to Mistral. So, in order to get visibility into the progress of this workflow, StackStorm needs to constantly poll Mistral for updates on the workflow it invoked. This is done in `st2resultstracker`, and we'll explore this component in greater detail in a future section.

StackStorm maintains a fork of Mistral to ensure that the version in place is stable and integrates well with StackStorm releases. Another important component to understand is . This isn't it's own running process, but it's key to understanding how Mistral is packaged with StackStorm. First, the [`st2mistral` repository](https://github.com/StackStorm/st2mistral) is where we maintain the Mistral plugins necessary for integrating with StackStorm. In addition, the StackStorm fork of Mistral as well as all of the plugins necessary for StackStorm integration are all provided [in the `st2mistral` package](https://github.com/StackStorm/st2-packages/tree/master/packages/st2mistral) (i.e. `deb` or `rpm`).

Let's dive into this integration and explore exactly how Mistral and StackStorm talk to each other. As mentioned previously, the `mistral-v2` runner allows you to reference a Mistral workflow just like you'd reference a Python or shell script in an action metadatafile:

```
vagrant@st2vagrant:~$ st2 runner get mistral-v2
+-------------------+--------------------------------------------------------------+
| Property          | Value                                                        |
+-------------------+--------------------------------------------------------------+
| id                | 58f1283bc4da5f2df72359a3                                     |
| name              | mistral-v2                                                   |
| description       | A runner for executing mistral v2 workflow.                  |
| enabled           | True                                                         |
| query_module      | mistral_v2                                                   |
| runner_module     | mistral_v2                                                   |
...
```

If you dive into the runner's code, you'll notice it's performing a few preparatory tasks. One of these is the transformation of the actual Mistral workflow file. There's a very specific reason for this, which becomes a bit more evident when you look at the list of actions from Mistral's perspective, and notice that there's only one action for StackStorm:

```
vagrant@st2vagrant:~$ mistral action-get st2.action
+-------------+--------------------------------------------------------+
| Field       | Value                                                  |
+-------------+--------------------------------------------------------+
| ID          | 7bc082cf-41ee-4c60-af0e-1a7c6de1a5ab                   |
| Name        | st2.action                                             |
| Is system   | True                                                   |
| Input       | action_context, ref, parameters=null, st2_context=null |
| Description | None                                                   |
| Tags        | <none>                                                 |
| Created at  | 2017-04-14 19:52:28                                    |
| Updated at  | None                                                   |
+-------------+--------------------------------------------------------+
```

This action is one of the Mistral plugins provided in the [`st2mistral`](https://github.com/StackStorm/st2mistral/blob/master/st2mistral/actions/stackstorm.py) repo. It exists because the actions executed within a StackStorm-invoked workflow need to be run within `st2actionrunner`, not within a Mistral executor. So, this mistral action's job is simple: forward execution requests to StackStorm. So, if you write a workflow and run it with StackStorm, you might be using `core.local`, or your own Python action - it doesn't matter. From Mistral's perspective, they're all `st2.action` because the actual work is being done back in StackStorm:

```
vagrant@st2vagrant:~$ mistral action-execution-list
+--------------------------------------+------------+-----------------+-----------+
| ID                                   | Name       | Workflow name   | Task name |
+--------------------------------------+------------+-----------------+-----------+
| 6c00f837-963c-43a8-ab34-43aa50b80e3b | st2.action | testpack.wftest | my_task   |
| 75268e75-26e6-45e3-aba3-4b6dc0caa675 | st2.action | testpack.wftest | my_task   |
| 33f9084c-00c6-4569-8dcd-cad3cd979954 | st2.action | testpack.wftest | my_task   |
+--------------------------------------+------------+-----------------+-----------+
```

> For those curious about how this plugin is "made known" to Mistral, check out the `[entry_points]` section of setup.cfg in [`st2mistral`](https://github.com/StackStorm/st2mistral/blob/master/setup.cfg#L8). StackStorm's packaging process takes care of installing `st2mistral` within Mistral's virtualenv.

So how does this actually work? Well, first let's take a sample workflow - note that we're printing the workflow as it exists in the pack:


```
vagrant@st2vagrant:~$ cat /opt/stackstorm/packs/testpack/actions/workflows/wftest.yaml
---
  version: '2.0'
  testpack.wftest:
    type: direct
    tasks:
      my_task:
        action: core.local
        input:
          cmd: "echo 'HELLO'"
```

Pretty straightforward - workflow that runs a single task that uses `core.local` to print "HELLO" to stdout. However, we can use the `mistral workflow-get-definition` command to see the workflow definition that Mistral received from our `mistral-v2` runner upon execution:

```
vagrant@st2vagrant:~$ mistral workflow-get-definition testpack.wftest
testpack.wftest:
  tasks:
    my_task:
      action: st2.action
      input:
        parameters:
          cmd: echo 'HELLO'
        ref: core.local
  type: direct
version: '2.0'
```

Note that the action name `core.local` we previously used has been moved to the `ref` field, and is now `st2.action`. Again, this is because the only action that Mistral recognizes for StackStorm is `st2.action`. This translation is one of the preparatory tasks that the `mistral-v2` runner performs on the workflow before sending it to Mistral. If you look at the [`mistral-v2` runner](https://github.com/StackStorm/st2/blob/master/contrib/runners/mistral_v2/mistral_v2.py), and follow the code down, you can find [a function that performs this translation](https://github.com/StackStorm/st2/blob/master/st2common/st2common/util/workflow/mistral.py), which the runner uses when sending a workflow definition to Mistral.

If you look at the output of `mistral action-get st2.action` as shown above, you can see that this action accepts a parameter of `ref`, which is the StackStorm action that actually needs to be executed. As a result, Mistral is happy because the translated workflow references an action that it knows about, and StackStorm is happy becuase this Mistral action is sending the parameters it needs to execute the `core.local` action.

So, to summarize, here's the order of operations when you invoke a Mistral workflow with StackStorm:

1. StackStorm invokes the `mistral-v2` runner. This performs prep tasks on workflow, like translating StackStorm references to something Mistral knows about, then sends result to Mistral for execution. The runner returns immediately after confirming the workflow has been sent to Mistral.
2. Mistral receives new workflow execution, and starts executing tasks in workflow. Any tasks that run in StackStorm (probably most/all of them) will use `st2.action` action.
3. The `st2.action` action will use StackStorm's execution API to run actions referenced in workflow

In short, StackStorm passes the workflow to Mistral, which passes individual actions back to StackStorm for execution. In this way, Mistral really is just used for the workflow orchestration, and StackStorm retains control of actually executing the referenced actions.
