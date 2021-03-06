---
author: Matt Oswalt
comments: true
date: 2013-02-14 21:30:06+00:00
layout: post
slug: a-cloud-without-ipv6
title: A Cloud Without IPv6
wordpress_id: 2451
categories:
- IPv6
tags:
- data center
- ipv6
- sdn
---

As a Data Center junkie, I daily bear witness to the glorious transformations that are taking place all around me with respect to the "next-generation" of data center. Everyone who wants to move their DC to the next level are millions of dollars worth of DC networking gear that is EXTREMELY cutting edge, enabling virtualization and cloud to do things we only dreamed of being able to do mere years ago. We're buying enough blade servers to fill hundreds or thousands of racks, counting in the hundreds of petabytes worth of memory, and enough CPU cores to fill a small star.

We're talking about dreams, here people. We're throwing together concepts like Software Defined ______ - IaaS, SaaS, PaaS, and *aaS like it's been our job to do so for centuries, but we've done it in the past few years. Sure, some of it is (or should be) still whiteboard material, but there is NO shortage of good ideas coming forward right now. All of the massively powerful and cool infrastructure I just talked about will be used to do unbelievably amazing things, not only for the companies that create or run them, but for mankind. I used to wonder if the [view of the future in 2054 from Minority Report](http://io9.com/5920302/minority-report-really-did-predict-the-future) was a little bit of a stretch ([kind of like how Back to the Future was pretty much wrong](http://www.11points.com/Movies/11_Predictions_That_Back_to_the_Future_Part_II_Got_Wrong) - don't get mad, love the movie, but we've got 2 years left to get our flying cars, so you tell me) but with all this awesome new technology that's showing no signs of slowing, things are starting to look up. I mean we have the Kinect, that's kind of like John Anderton's little crime-board of future murder, right?

All silliness aside, we've got it going on right now. It is a true testament to the ingenuity and determination -

[![photo1]({{ site.url }}assets/2013/02/photo1.png)]({{ site.url }}assets/2013/02/photo1.png)

Wait - you're telling me we're still using a 32 bit address space? We want all of our phones, cars, pets, trash cans and power tools to be connected to the internet and orchestrated under all this computing power and software written by millions of little geniuses running around the globe coming up with an EXPONENTIALLY INCREASING number of cool ideas to make the world better, and we are using an address space whose capacity is literally equivalent to no more than half the world's population? Well, sorry 3.5 billion people, you don't get any form of internet connectivity, and the other 3.5 billion, you just get one IP, so your phone gets internet. Oh, by the way, the internet in this alternate reality sucks a lot, since there's no IPs left for any kind of server infrastructure, so you are basically now one big usenet newsgroup. Enjoy.

(Yes I am aware of NAT - I'm making a point.)

That logo up top is an artists depiction of our hopes and dreams getting stomped on, ladies and gentlemen. We're accelerating upwards and onwards, completely disregarding the fact that we've missed a crucial part of enabling these big dreams of ours. Don't get me wrong - the dreams are in the right place. It's a clear indication that we've advancing technologically to that next level. But what value do these dreams have if we haven't done EVERYTHING necessary to ensure their success?

Remember the OSI model? We've done a pretty good job of making sure Layers 1 and 2 are good. We have backplanes in the terabits per second - unheard of only a short time ago. We've got pretty wide adoption of 10GbE, and 40GbE is just starting to get out there. We have enough computing power to heat Pluto. We even have some really cool Layer 2 features, brought on largely by the presence of virtualization, such as Multi-chassis Etherchannel, enabling us to really scale out our datacenters, as well as between data centers with technology like OTV (like it or not). Truth be told, we are on the ball with these two layers.

Then we caught a glance at Layer 7 - ooh, you can do some cool stuff with Layer 7. Let's put a computer in everyone's pocket. Oh, it's the size of a file cabinet? Let's make it smaller then! Oh, and give it the ability to make phone calls without any cables. Then let's make the hardware manufacturers really up their game by creating applications that require more and more computing power because they do more and more cool stuff.

We've completely blasted past the fact that the OSI model exists for a reason - each layer depends crucially on the layer below it to work properly. Don't believe me? Try doing jack without layer 1.

It's long past time we started seriously rolling out IPv6 in our networks. I don't care if you just want to try [enabling it on your internet edge first](http://www.cisco.com/en/US/prod/collateral/iosswrel/ps6537/ps6553/ipv6_internet_edge_services_aag.pdf) and seeing what happens. That's honestly one of the best ways to do it, since [IPv6 is going to run on your network whether you want it to or not](http://packetpushers.net/ipv6-security-tips-whether-youre-deploying-it-or-not/). I'm not just talking about flipping a switch to enable it, I'm talking about ensuring that your Layer 3 network is going to play it's part in what will undoubtedly be the next evolution of the internet.

Now, there are some great things happening in the mean time, such as [Verizon's IPv6-mandatory LTE network](http://www.networkworld.com/news/2009/061009-verizon-lte-ipv6.html). Great step in the right direction. With a large IPv6-capable network like this that obviously has quite a few users on it, it's clear that IPv6 adoption is no longer just a whiteboard idea.

So why are we still adopting this thing so slowly? Here's one reason - a shockingly high number of engineers just don't know ANYTHING about IPv6. It's no surprise, take a look at most "current" networking curricula. Does "honorable mention" ring a bell? Following these curricula, everyone just learns the syntax to enable the IPv6 version of routing protocols, and that's about it. Frankly, I'd be surprised if the majority even do that - looking at the blueprint, if I was just getting into this, I'd give IPv6 maybe a day's worth of study to get by it on the exam, then forget about it. For the moment, I'm talking about Cisco certifications, but it could apply to others; Cisco certifications seem to treat IPv6 as it's own separate entity. (This is one area that I think [Juniper has done EXTREMELY well in](http://www.networkworld.com/community/blog/ipv6-certifications)) They don't educate the learner that this is the next generation of Layer 3, and you better learn to do both IPv4 and IPv6, because it's what you're going to be exposed to. There's really no education about IPv6 as a protocol, just the routing protocols that have been updated to carry these new IPv6 routes.

[The Internet Of Everything](http://blogs.cisco.com/news/the-internet-of-everything-has-begun/) is not just a marketing campaign, it is a reality.  Entry-level engineers in our field look to big companies like Cisco to set the bar for what they need to know to be successful. I've met many new engineers, either fresh out of college, or changing out of a previous career, and they all look to the CCIE as the mecca of networking education. [They're not completely wrong](http://blog.ioshints.info/2012/02/does-ccie-still-make-sense.html), [but there are several factors that are challenging this dynamic](http://packetpushers.net/network-interrupted/). I believe that the engineers that will be chief architects over massive IPv6-based networks are being trained today.

Spreading the word about IPv6 and answering the hard questions - it's up to the educators and the IPv6 community to get people talking. We will only solve these problems when we start talking about them. Sitting back and saying "IPv6 has too many problems" doesn't solve anything. No - it's not perfect, but nothing is, and it certainly won't improve until you jump in and help figure out the tough questions. [The alternatives are MUCH worse](http://www.networkworld.com/news/2010/060710-tech-argument-ipv6-nat.html). You want to talk about NAT? Sure! I'll talk about anything, as long as we're TALKING. Put the punching gloves away and lets solve this problem.
