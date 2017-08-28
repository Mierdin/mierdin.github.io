---
author: Matt Oswalt
comments: true
date: 2017-08-25 00:00:00+00:00
layout: post
slug: stackstorm-architecture-core-services
title: 'StackStorm Architecture Part I - StackStorm Core Services'
categories:
- Blog
tags:
- automation
- stackstorm
---

A while ago, I wrote about [basic concepts in StackStorm](https://keepingitclassless.net/2016/12/introduction-to-stackstorm/). Since then I've been knee-deep in the code, fixing bugs and creating new features, and I've learned a lot about how StackStorm is put together.

Now, I'd like to spend some time exploring the StackStorm architecture. What subcomponents make up StackStorm? How do they interact? How can we scale StackStorm? These are all questions that come up from time to time in our community, and there are a lot of little details that I even forget from time-to-time. I'll be doing this in a series of posts, so we can explore a particular topic in detail without getting overwhelmed.

Also, it's worth noting that this isn't intended to be an exhaustive reference for StackStorm's architecture. The best place for that is still the [StackStorm documentation](https://docs.stackstorm.com/). My goal in this series is merely to give a little bit of additional insight into StackStorm's inner workings, and hopefully get those curiosity juices flowing.

Here are some useful links to follow along - this post mainly focuses on the content there, and elaborates:

- [High-Level Overview](https://docs.stackstorm.com/install/overview.html)
- [StackStorm High-Availability Deployment Guide](https://docs.stackstorm.com/reference/ha.html)
- [Code Structure for Various Components in "st2" repo](https://docs.stackstorm.com/development/code_structure.html)

## StackStorm High-Level Architecture

Before diving into the individual StackStorm services, it's important to start at the top; what does StackStorm look like when you initially lift the hood?

The best place to start for this is the [StackStorm Overview](https://docs.stackstorm.com/overview.html), where StackStorm concepts and a very high-level walkthrough of how the components interact is shown. In addition, the [High-Availability Deployment Guide](https://docs.stackstorm.com/reference/ha.html) (which you should absolutely read if you're serious about deploying StackStorm) contains a much more detailed diagram, showing the actual, individual process that make up a running StackStorm instance:

<div style="text-align:center;"><a href="{{ site.url }}assets/2017/04/services.png"><img src="{{ site.url }}assets/2017/04/services.png" width="500" ></a></div>

> It would be a good idea to keep this diagram open in another tab while you read on, to understand where each service fits in the cohesive whole that is StackStorm

As you can see, there's not really a "StackStorm server". StackStorm is actually comprised of multiple microservices, each of which has a very specific job to do. Many of these services communicate with each other over RabbitMQ, for instance, to let each other know when they need to perform some task. Some services also write to a database of some kind for persistence or auditing purposes. The specifics involved with these usages will become more obvious as we explore each service in detail.

## StackStorm Services

Now, we'll dive in to each service individually. Again, the purpose of this post is to explore each service individually to better understand them, but remember that they must all work together to make StackStorm work. It may be useful to keep the diagram(s) above open in a separate tab, to keep the big picture in mind.

We'll be looking at things from a systems perspective as well as a bit of the code, where it makes sense. My primary motivation for this post is to document the "gist" of how each service is implemented, to give you a head start on understanding them if you wish to either know how they work, or contribute to them. Selfishly, I'd love it if such a reference existed for my own benefit, so I'm writing it.

### st2actionrunner

We start off by looking at [`st2actionrunner`](https://docs.stackstorm.com/reference/ha.html#st2actionrunner) because, like the Actions that run inside them, it's probably the most relatable component for those that have automation experience, but are new to StackStorm or event-driven automation in general.

`st2actionrunner` is responsible for receiving execution (an instance of a running action) instructions, scheduling and executing those executions. If you dig into the `st2actionrunner` code a bit, you can see that it's powered by two subcomponents: a [scheduler](https://github.com/StackStorm/st2/blob/master/st2actions/st2actions/scheduler.py), and a [dispatcher](https://github.com/StackStorm/st2/blob/master/st2actions/st2actions/worker.py). The scheduler receives requests for new executions off of the message queue, and works out the details of when and how this action should be run. For instance, there might be a policy in place that is preventing the action from running until a few other executions finish up. Once an execution is scheduled, it is passed to the dispatcher, which actually runs the action with the provided parameters, and retrieves the resulting output.

> You may have also heard the term "runners" in reference to StackStorm actions. In short, you can think of these kind of like "base classes" for Actions. For instance I might have an action that executes a Python script; this action will use the `run-python` runner, because that runner contains all of the repetitive infrastructure needed by all Python-based Actions. Please do not confuse this term with the `st2actionrunner` service; `st2actionrunner` is a running process for running all Actions, and a "runner" is a Python base class to declare some common foundation for an Action to use. In fact, `st2actionrunner` is indeed [responsible for handing off execution details to the runner](https://github.com/StackStorm/st2/blob/master/st2actions/st2actions/container/base.py), whether it's a Python runner, a shell script runner, etc.

As shown in the component diagram, `st2actionrunner` communicates with both RabbitMQ, as well as the database (which, at this time is MongoDB). RabbitMQ is used to deliver incoming execution requests to the scheduler, and also so the scheduler can forward scheduled executions to the dispatcher. Both of these subcomponents update the database with execution history and status.

### st2api

If you've worked with StackStorm at all (and since you're still reading, I'll assume you have), you know that StackStorm has an API. External components, such as the CLI client, the Web UI, as well as third-party systems all use this API to interact with StackStorm.

`st2api`, like the aforementioned `st2actionrunner` runs as it's own separate process. To put it very loosely, `st2api` "translates" API calls into RabbitMQ messages and database interactions. What's meant by this is that incoming API requests are usually aimed at either retrieving data, pushing new data, or executing some kind of action with StackStorm. All of these things are done on other running processes; for instance, `st2actionrunner` is responsible for actually executing a running action, and it receives those requests over RabbitMQ. So, `st2api` must initially receive such instructions via it's API, and forward that request along via RabbitMQ. Let's discuss how that actually works.

> The 2.3 release changed a lot of the underlying infrastructure for the StackStorm API. The API itself isn't changing (still at v1) for this release, but the way that the API is described within `st2api`, and how incoming requests are routed to function calls has changed a bit. Everything we'll discuss in this section will reflect these changes. Pleaes review [this issue](https://github.com/StackStorm/st2/issues/2686) and [this PR](https://github.com/StackStorm/st2/pull/2727) for a bit of insight into the history of this change.

The way the API itself actually works requires its own blog post for a proper exploration. For now, suffice it to say that StackStorm's API is defined with the [OpenAPI specification](https://github.com/StackStorm/st2/blob/master/st2common/st2common/openapi.yaml). Using this definition, each endpoint is linked to its own "implementation function" behind the scenes. These functions may write to a database, they may send a message over the message queue, or they may do both. Whatever's needed in order to implement the functionality offered by that API endpoint is performed within that function.

For the purposes of this post however, let's talk briefly about how this API is actually served from a systems perspective. Obviously, regardless of how the API is implemented, it will have to be served by some kind of HTTP server.

> Note that in a production-quality deployment of StackStorm, the API is front-ended by nginx. We'll be talking about the nginx configuration in another section, so we'll not be discussing it here. But it's important to keep this in mind.

We can use this handy command, filtered through `grep` to see exactly what command was used to instantiate the `st2api` process.

```
~$ ps --sort -rss -eo command | head | grep st2api

/opt/stackstorm/st2/bin/python /opt/stackstorm/st2/bin/gunicorn st2api.wsgi:application -k eventlet -b 127.0.0.1:9101 --workers 1 --threads 1 --graceful-timeout 10 --timeout 30
```

As you can see, it's running on Python, like most StackStorm components. Note that this is the distribution of Python in the StackStorm virtualenv, so anything run with this Python binary will already have all of its pypi dependencies satisfied - these are installed with the rest of StackStorm.

The second argument - `/opt/stackstorm/st2/bin/gunicorn` - shows that [Gunicorn](http://gunicorn.org/) is running our API application. Gunicorn is a WSGI HTTP server. it's used to serve StackStorm's API as well as a few other components we'll explore later. You'll notice that for `st2api`, the third positional argument is actually a reference to a Python variable (remember that this is running from StackStorm's Python virtualenv, so this works). Looking at [the code](https://github.com/StackStorm/st2/blob/master/st2api/st2api/wsgi.py) we can see that this variable is the result of a call out to the setup task for the [primary API application](https://github.com/StackStorm/st2/blob/master/st2api/st2api/app.py), which is where the aforementioned OpenAPI spec is loaded and rendered into actionable HTTP endpoints.

Obviously there's a lot more to talk about regarding how the API is actually implemented, but that's for another post. For now, this is how `st2api` works at a systems level.

TODO what about webhooks?

### st2rulesengine

While `st2rulesengine` might be considered one of the simpler services in StackStorm, its job is the most crucial. It is here that the entire premise of event-driven automation is made manifest.

For an engaging primer on rules engines in general, I'd advise listening to [Sofware Engineering Radio Episode 299](http://www.se-radio.net/2017/08/se-radio-episode-299-edson-tirelli-on-rules-engines/). I had already been working with StackStorm for a while when I first listened to that so I was generally familiar with the concept, but it was nice to get a generic perspective that explored some of the theory behind rules engines.

Remember my earlier post on [StackStorm concepts](https://keepingitclassless.net/2016/12/introduction-to-stackstorm/)? In it, I briefly touched on Triggers - these are definitions of an "event" that may by actionable. For instance, when someone posts a tweet that matches a search we've configured, the Twitter sensor may use the `twitter.matched_tweet` trigger to notify us of that event. A specific instance of that trigger being raised is known creatively as a "trigger instance".

In short, StackStorm's rules engine looks for incoming trigger instances, and decides if an Action needs to be executed. It makes this decision based on the rules that are currently installed and enabled from the various packs on the system.

As is common with most other StackStorm services, the logic of this service is contained within a ["worker"](https://github.com/StackStorm/st2/blob/master/st2reactor/st2reactor/rules/worker.py), using a handy Python base class which centralizes the receipt of messages from the message queue, and allows the rules engine to focus on just dealing with incoming trigger instances.

The [engine itself](https://github.com/StackStorm/st2/blob/master/st2reactor/st2reactor/rules/engine.py) is actually quite straightforward:

1. Receive trigger instance from message queue
2. Determine which rule(s) match the incoming trigger instance
3. Enforce the consequences from the rule definition (usually, executing an Action)

> The [rules matcher](https://github.com/StackStorm/st2/blob/master/st2reactor/st2reactor/rules/matcher.py) and [enforcer](https://github.com/StackStorm/st2/blob/master/st2reactor/st2reactor/rules/enforcer.py) are useful bits of code for understanding how these tasks are performed in StackStorm. Again, while the work of the rules engine in StackStorm is crucial, the code involved is fairly easy to understand.

`st2rulesengine` needs access to RabbitMQ to receive trigger instances and send a request to execute an Action. It also needs access to MongoDB to retrieve the rules that are currently installed.






Timers? Is that why TriggerWatcher is used? Do Sensors have anythingt to do with this or is this contained in the rules engine?
hhttps://github.com/StackStorm/st2/blob/master/st2reactor/st2reactor/timer/base.py

How does a rule send registrations of triggers to the sensorcontainer? Is this even a thing? What does the communicatio between the rules engine and the sensor container look like?

Do trigger-instances show up even if there are no rules? I think they do - confirm this: that the only indication that a rule evaluated to true was that it resulted in an execution (and logs of course). Maybe traces help here?

"triggered by" in GUI execution history shows how executions were kicked off (like by rules). Is this available in CLI?

### st2sensorcontainer

The job of the `st2sensorcontainer` service is to execute and manage the Sensors that have been installed and enabled within StackStorm. The name of the game here is to simply provide underlying infrastructure for running these Sensors, as much of the logic for how the Sensor itself works is done within that code. This includes dispatching Trigger Instances when a meaningful event has occurred. `st2sensorcontainer` just maintains awareness of what Sensors are installed and enabled, and does its best to keep them running.

The [sensor manager](https://github.com/StackStorm/st2/blob/master/st2reactor/st2reactor/container/manager.py) is responsible for kicking off all the logic of managing various sensors within `st2sensorcontainer`. To do this, it leverages two subcomponents:

- [process container](https://github.com/StackStorm/st2/blob/master/st2reactor/st2reactor/container/process_container.py): Manages the processes actually executing Sensor code
- [sensor watcher](https://github.com/StackStorm/st2/blob/master/st2common/st2common/services/sensor_watcher.py): Watches for Sensor Create/Update/Delete events 

#### Sensors - Process Container

The process container is responsible for running and managing the processes that execute Sensor code. If you look at the [process container](https://github.com/StackStorm/st2/blob/master/st2reactor/st2reactor/container/process_container.py) code, you'll see a `_spawn_sensor_process` actually kicks off a `subprocess.Popen` call to execute a ["wrapper" script](https://github.com/StackStorm/st2/blob/master/st2reactor/st2reactor/container/sensor_wrapper.py):

```
~$ st2 sensor list
+-----------------------+-------+-------------------------------------------+---------+
| ref                   | pack  | description                               | enabled |
+-----------------------+-------+-------------------------------------------+---------+
| linux.FileWatchSensor | linux | Sensor which monitors files for new lines | True    |
+-----------------------+-------+-------------------------------------------+---------+

~$ ps --sort -rss -eo command | grep sensor_wrapper

/opt/stackstorm/st2/bin/python /opt/stackstorm/st2/local/lib/python2.7/site-packages/st2reactor/container/sensor_wrapper.py --pack=linux --file-path=/opt/stackstorm/packs/linux/sensors/file_watch_sensor.py --class-name=FileWatchSensor --trigger-type-refs=linux.file_watch.line --parent-args=["--config-file", "/etc/st2/st2.conf"]
```

This means that each individual sensor runs as its own separate process. The usage of the wrapper script enables this, and it also provides a lot of the "behind the scenes" work that Sensors rely on, such as dispatching trigger instances, or retrieving pack configuration information. So, the process container's job is to spawn instances of this wrapper script, with arguments set to the values they need to be in order to run specific Sensor code in packs.

#### Sensors - Watcher

We also mentioned another subcomponent for `st2sensorcontainer` and that is the "sensor watcher". This subcomponent watches for Sensors to be installed, changed, or removed from StackStorm, and updates the process container accordingly. For instance, if we install the [`slack`](https://github.com/StackStorm-Exchange/stackstorm-slack) pack, the [`SlackSensor`](https://github.com/StackStorm-Exchange/stackstorm-slack/blob/master/sensors/slack_sensor.yaml) will need to be run automatically, since it's enabled by default.

The sensor watcher subscribes to the message queue and listens for incoming messages that indicate such a change has taken place. In the [watcher code](https://github.com/StackStorm/st2/blob/master/st2common/st2common/services/sensor_watcher.py), a handler function is referenced for each event (create/update/delete). So, the watcher listens for incoming messages, and calls the relevant function based on the message type. By the way, those functions are defined back in our [sensor manager](https://github.com/StackStorm/st2/blob/master/st2reactor/st2reactor/container/manager.py), where it has has access to instruct the process container to make the relevant changes.

That explains how CUD events are handled, but where do these events originate? When we install the `slack` pack, or run the `st2ctl reload` command, some [bootstrapping code](https://github.com/StackStorm/st2/blob/master/st2common/st2common/bootstrap/sensorsregistrar.py) is executed, which is responsible for updating the database, as well as publishing messages to the message queue, to which the sensor watcher is subscribed.

### st2auth

Take a look at StackStorm's [API definition](https://github.com/StackStorm/st2/blob/master/st2common/st2common/openapi.yaml) and search for "st2auth" and you can see that the authentication endpoints are defined alongside the rest of the API. Only difference is, gunicorn is executed with different parameters?

```
~$ ps --sort -rss -eo command | head | grep st2api

/opt/stackstorm/st2/bin/python /opt/stackstorm/st2/bin/gunicorn st2api.wsgi:application -k eventlet -b 127.0.0.1:9101 --workers 1 --threads 1 --graceful-timeout 10 --timeout 30

~$ ps --sort -rss -eo command | head | grep st2auth

/home/vagrant/st2/virtualenv/bin/python ./virtualenv/bin/gunicorn st2auth.wsgi:application -k eventlet -b 0.0.0.0:9100 --workers 1

```



How do other components authenticate? Look at logs, the beginning of sensorcontainer shows a token being allocated

Also gunicorn

### st2resultstracker

Should I call out the callback vs resultstracker question here?

### st2notifier

### st2garbagecollector




## Conclusion

PUT EACH section below in its own post. Just briefly hint at them in the conclusion.




### Mistral

[Mistral](https://docs.stackstorm.com/mistral.html) is very important; it's fair to say this is the most robust option for Workflows in StackStorm today. In case you're unfamiliar with Mistral, [it is an OpenStack project](https://wiki.openstack.org/wiki/Mistral) for managing workflows. Since Mistral provides a lot of useful workflow primitives, StackStorm integrates with Mistral, and provides plugins to allow Mistral to execute StackStorm actions.

Mistral is a complex subject on its own, and requires its own blog post (coming soon) for a proper deep-dive. For now, suffice it to say that it behaves much like other StackStorm components, in that it operates as a standalone process, sending and receiving instructions from StackStorm in the execution of workflows.

TODO - Add explanation of how results are tracked after you've explored `st2resultstracker`

## Third-Party Components

- Need to talk about RMQ, nginx, mongo, and postgres