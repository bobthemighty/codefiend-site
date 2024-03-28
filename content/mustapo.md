+++
title = "MUSTAPO: an architecture qualities checklist"
slug = "mustapo-architecture-qualities-list"
date = "2024-03-23"
category = "architecture"

[extra]
author = "Bob Gregory"

[taxonomies]
tags = ["architecture"]
+++

Whenever I'm designing systems, there are three things to consider: the _functional requirements_, ie. the stuff that the system does; the _constraints_, ie, the time, money, and knowledge factors; and the _architectural qualities_ - the non-functional requirements against which we test our design.

For the architecure qualities, or -ilities, I use a handy mnemonic: MUSTAPO.
<!-- more -->

I stole this, as with so many things, from Ian Cooper, though I did put an -O on the end, so that's my contribution to the annals of software architecture.

MUSTAPO stands for:

* Modifiability
* Usability
* Security
* Testability
* Availability
* Performance
* Observability

Each of these is a classic architecture quality, a characteristic that we want to have in our system. The way I use them is to think through the requirements for my system in each category. When I form a design for the system, I can test it against those qualities to see if it will satisfy the requirements, or I can use them to make trade-offs.

I often think in terms of risk. If I get this wrong, how would that make the system broken? Note that _nothing_ is perfectly secure, and _nothing_ is perfectly available. The goal here is to understand what the real requirements are for your current system.

I'm going to define these here, because people often ask me if it's written down anywhere, and I'll do a follow up post where we can do an architecture kata to see how they work in practice.

## Modifiability

A system is modifiable if you can change its behaviour. I have written code, small scripts or batch jobs, where I _never_ needed to change the code after it was written. In those cases, write whatever scruffy garbage will pass muster.

I have written code where non-technical users needed to configure the behaviour at run time with a rules engine, eg. to configure the rules for shipping parcels, or to configure the rules for detecting problems in IoT data. 

These are two extremes: dynamic behaviour based on config, and static behaviour. Usually our systems need to evolve over time, as our understanding of the domain deepens. This is why we apply fancy design patterns: they give us optionality for extending or replacing behaviour later, without making destructive changes.

A broken system is a system that can't keep pace with the changes demanded by its customers.

## Usability

A system is usable if it satisfies its functional requirements without pissing people off. It's helpful to expand your definition of "user" for this category. I've written systems that did nothing but read from a message queue, perform some logic, and send another message. These systems still have users: their users are the downstream consumers of the messages, and the poor engineers who have to deploy and troubleshoot the system during an incident.

Consider the devices that your users will work with. Sometimes your users will be using your system on a tablet, while wearing gloves, in a messy workshop. Sometimes your users will be on a locked-down browser in a cubicle. Sometimes your users will be blind, or elderly. How does this affect the way that people need to interact with your system?

A broken system is a system that isn't accessible by its users, or that requires too much manual intervention to be assistive.

## Security

A system is secure if it continues to function correctly in the presence of hostile actors. 

It's useful to ask "if I were evil, and I had access to this system, what could I achieve?". There is a significant difference in the security requirements for "https://cheese.com" and for your medical health records, or your banking app.

Most people's threat model is "script kiddy on the internet" or "sleepy software engineer". That's very different to "nation state" or "organised crime syndicate". If you're building a messaging app for North Korean dissidents, your threat model is radically different to the threat model for https://codefiend.co.uk.

Ask yourself what your hostile actors could achieve with access to each component, and then consider the mitigations. Could they cause financial loss? Could they make something blow up? Could they out gay kids to their peers and parents?

A broken system is a system that causes embarassment, financial loss, or death.

## Testability

A system is testable if I can put it into a chosen state, cause it act, and observe the result.

When we're building software systems, we want to be able to test them, so that we know they work. Most of our tests (hopefully) will be fast, in-memory, developer tests. Some of our tests might need to be slower end-to-end or integration tests that cross process boundaries.

Think about the use-cases of your system. How can you demonstrate that each use-case is working correctly? Often we can prove by construction: if component A works correctly, and component B works in the presence of a working A, then we know component B works correctly.

Think about the things that will make your system hard to test: third party integrations, non-determinism, async behaviour. How can you design the system so that you can exercise these components and assert their behaviour?

A broken system is one that is full of bugs, because it's too hard, or too expensive to test.

## Availability

A system is available if it responds to a request. It's unavailable if it's down for maintenance, or just kaput.

Think about the required up time for your system. Maybe you are building a system that has to run 24x7 and supports some horrendously expensive financial process. Maybe you're building a system that runs a batch job once a week. If it fails the first time, but succeeds on the 8th attempt, is that okay? Is it okay if it's not available at the weekend? How many times per day can a user see the "oops, we know there's a problem here."

If your system is _down_, does that matter? What will it cost you?

To increase availability, we can look at redundancy of critical components, like database servers, or we can construct fancy deployment systems where we roll out a new version to 1% of traffic at a time.

A broken system doesn't respond when it's needed.

## Performance

This one's a bit of a cheat, because it's really three architectural qualities in a trench-coat. To think about performance we need to think about *throughput*, *latency*, and *scalability*.

Throughput is the number of requests that we can satisfy in some time period. I'm working with an IOT system at the moment, where we receive a few thousand metrics every minute. 

Latency is the time it takes for us to satisfy a single request. In my case, a metric arrives at our ingest endpoint, and we push it through a series of lambda functions over the course of a few minutes.

Scalability is the ability to maintain some level of performance when we add capacity to the system. As we bring on new customers, they will each start sending us gajillions of metrics. To scale, we need to be run more lambda functions in parallel so that we can keep up with the increased volume. Because we can easily add more capacity for a linearly increasing price, we say that this system scales well.

You can go really fast by keeping everything in memory on a single computer, but that will hit a throughput limit pretty quickly, and you can't scale it.

If you add a database server, and a bunch of web servers, then you increase throughput, at the expense of latency. Web servers generally scale horizontally - you can just keep throwing more servers up, but databases do not. Instead we generally have to shard data, or make our database server arbitrarily huge and expensive.

Think about how many requests you need to satisfy, and how long you can reasonably take before your users get annoyed. Think about how your traffic will change in future, and how you can add capacity to handle that growth. Which components will hit a scaling bottleneck first, and is that ceiling low enough to be an issue?

A broken system is one that can't handle the load, and falls over, or is slow enough that users get bored and spend their money elsewhere.

## Observability

A system is observable if we can understand its internal state through its outputs. 

This is the dual of testability. In testability, we put the system into a known state and observe its output. With observability, we observe the output to understand the state. Many things that are hard to test (those third party integrations, for example) can be observed in production.

Observability is not the same as monitoring. It's not just about errors, or about performance, it's about our ability to ask questions of our production systems.

As with testability, I like to think about the use-cases of my system. For each use case, what signal do I need to understand whether it's working correctly? For example, in my current system, we ingest a butt load of IoT data, and process it so that we can run ML models on it. I want to observe:

* How big is the butt-load of data arriving every minute?
* How often does we fail to receive data?
* How many unique data points do we process at each step?
* How long does each step take?
* What's the end-to-end latency of the whole thing?
* What were the outputs of our ML models?

and so on. Recording all these into Honeycomb lets us view the behaviour of our system in near real-time and quickly diagnose problems when they inevitably occur.
