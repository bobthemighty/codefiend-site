+++
title = "How I prioritise work"
slug = "prioritisation-for-engineering-managers"
date = "2024-04-19"
category = "process"

[extra]
author = "Bob Gregory"

[taxonomies]
tags = ["process", "management"]
+++

I had a conversation with a friend today who is fairly new to engineering management, and struggling with the usual set of problems. One thing he mentioned in particular was "it's hard for me to make space in our delivery cycle for us to deliver technical improvements". 

I'm not an expert here by any means, but I think we do this pretty well at CarbonRe.

There are three sub-problems here: who gets to decide what work is delivered, how do we define technical work in the first place, and how do we effectively prioritise when there's too much stuff to do.

<!-- more -->

# Roles and responsibilities

One of the first things I did at CarbonRe is talk to the product team about roles and responsibilities. There's a lot of teams out there who have some kind of Scrum or Scrumban process where Product Owners have become micro-managers.

This is a lethal anti-pattern because it sets up a them-and-us culture.

> We need to solve some technical debt, but the product owner won't let us.

If I had a nickel for every time I'd heard this statement, I'd have a load of nickels that I can't spend because I'm British and they'd just sit around on my desk as a shiny, angry reminder of the ways in which we've collectively failed the agile dream.

## Accountability requires responsibility

I'm the head of engineering. If the system breaks, then I am personally accountable for the breakage. It doesn't matter if it broke because we didn't test the code, or because of an architectural oversight, or a process failure. Ultimately that all lands on my desk, and I'm the one at fault.

The product owner does not get paged at 3am if the system breaks, and so the product owner *does not get to decide* what work the team should do.

There are a few common objections to this.

> But that's just how it works, the product owners decide what goes into the sprint, they won't let me choose the priority.

Friend, you're a manager. Convincing people is your job. If you can't have a frank conversation with your peer and reach a satisfactory compromise, you're not going to make it. If, somehow, you're an engineering manager who reports to the product owner, then things have gone very sideways. Talk to your mutual boss, and fix it. You can't run a team if you're not able to make decisions.

You can only be accountable if you have responsibility. Make that clear.

> But if the product owner isn't deciding the priority of work, what's the point of them?

The role of a product team in a healthy org is to:

* assess the _problems_ that customers have
* articulate the _value_ of solving those problems
* and lead the process of designing _solutions_ to those problems.

Once the product team have articulated the relative _value_ of solving different problems, the engineering team can describe the _cost_ of solving those problems. That gives the product team a way to prioritise: how do we generate the most value at the least cost?

Given that prioritisation, it's the role of engineering leadership to figure out how to get their team to deliver against those priorities.

> But if the engineering team decide what to work on, they'll just fritter away their time on refactoring.

As the engineering manager, I am personally accountable for the output of my team. If my team is spending all their time on technically exciting, but ultimately worthless, activity then I'm the one who gets fired.

Again, I can only be accountable if I have responsibility. If you don't have a framework for roles and responsibilities that allows you to make decisions for the technical health of your system, go fix it.

# How do we define technical work?

My old boss used to say "I have never said no to a meaningful proposal to improve our code or processes",  and yet, engineers were forever complaining that they didn't get to fix things. What gives?

If you want to make technical improvements to your system, you need to propose that like any other project. We use a simple notion template that asks three questions:

{% paper(type="note") %}
### What?

What do you want to build. Describe the outcome of the project. What will we be able to do when the project is complete? This needs to be detailed enough that people can actually define work, with an agreed end point.

### Why?

What's the justification for doing this work? If you can't link your proposed work to a tangible business need, you can't justify spending time and money on it.

### When?

Every piece of work has some kind of priority. Does this need to happen *now*? Does this need to happen *this quarter*? Is it just an idle thought for one blissful day when the team has run out of things to do?
{% end %}

As an example, I want to do some work to fix our build pipeline so that we can deploy faster. Here's how that might work.

{% paper(type="note") %}
### What?

Fix the deployment system so that we can build and deploy individual components of our monorepo. We should be able to make a change to one part of our stack without needing to build, test, and deploy the whole system. We should target a time of < 5 minutes from a commit to production. This can be done in roughly a sprint.

### Why?

During our last outage, 
* we received alerts within 3 minutes of the system breaking
* it took us 5 minutes to identify the cause, write a fix and push a commit
* it then took *40* minutes for the build pipeline to reach production. 

We can respond to failure more quickly if we can deploy code more quickly. This reduces the likelihood that we lose customers over minor incidents.

### When?

We should do this work before we launch $PROJECT_X, since that will increase the number of customers on the platform, and increase the likelihood that an incident has direct customer impact.
{% end %}

Simples. This is _much_ more likely to get buy in than "we want to make github actions faster by improving our [pants](https://www.pantsbuild.org/) usage." 

Your peers do not care about your monorepo, or your pants. They care about customer retention, and they want to know how long the work will take.

If your engineers want to make technical improvements, insist that they take the time to propose them properly. If they don't know how, it's your job to show them.

# How do we decide what's most important?

I use a really simple priority list, and I talk about it at every single planning meeting so that all the stakeholders know the score.

1. Production incidents. 
   If the system is broken, then we interrupt whatever else is going on to get it fixed.
2. Hard customer commitments. 
   If we have a contract with a customer to deliver X product by Y date, then we get that done, even if that means skimping on technical improvements.
3. Incident actions. 
When something goes wrong in production, we take steps to make sure that doesn't happen again. We identify these in a post-mortem meeting, and they take priority over everything except customer commitments and live incidents.
4. Work in flight. 
If you've got 3 projects in progress, do not start another one until all the work in flight is completed. Teams are happier and more productive when they get to finish stuff. Half-finished work does not deliver value.
5. New work. 
This is a combination of technical work that needs to be done to improve the health of the system, and new projects to satisfy the product team.

With that mental list, it's easy to look at the stuff in the backlog and understand what needs to be done: fix prod, deliver on time, stop prod breaking again, finish the incompleted tasks, then pick up new work.

How do you prioritise when you have multiple options for new work? Go back to the _value_ that you're delivering. Is it more valuable to the company to release a new feature, or to make future work quicker and easier? If you've fixed the roles and responsibilities, then it's *up to you* to justify the decision.

# Use the "Architecture Tax"

Technical projects are often difficult to prioritise because the value can be hard to define. If we have two projects on the table "deliver new customer widget" and "refactor old customer widget", it can be difficult to justify the cost of the latter.

One handy technique is to _tax_ customer-facing projects by mixing technical improvements into the deliverables. This makes sense, because - often - improvements only matter when you're actively changing the code.

For example, I'm about to start a project to extract a "recommendations" api. We've scheduled that work to happen as a prerequisite to adding a new customer to our platform. 

When we added the last customer, there was a hard commitment, and we'd have found it hard to deliver on time if we'd done this refactoring. We chose to deliver to the customer on the old code that makes us sad.

For our new customer, we have time, so we're starting the project early, and fixing the architecture before we start to deliver.

This is a much easier sell: "in order for us to deliver customer X, we need to fix the way that recommendations are stored and served. That'll add 2 weeks onto the delivery time but will make future customers faster to onboard."

# Summary

So there you have it. If you're a new-ish engineering manager, and you're struggling to prioritise technical work:

* Define roles and responsibilities
* Ensure that proposed technical work has a business justification
* Use a simple priority hierarchy, and make that well understood
* Link technical improvements to customer-facing work
