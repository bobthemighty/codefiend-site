+++
title = "A Fair Pay Review Process"
slug = "a-fair-pay-review-process"
date = "2024-05-06"
category = "management"

[extra]
author = "Bob Gregory"

[taxonomies]
tags = ["management"]
+++

<style>
td { text-align: center }
</style>

# How to run a fair pay review process

So you're an engineering manager for the first time, and you've got to figure out how to run a pay review process. This is easy to screw up and, if you do, you will make lots of people unhappy. I want to show you the simplest, most defensible, way to allocate pay rises to staff. I need a cool name, so I'm going to call it The STEVE Protocol.

<!-- more -->

I'd like to claim there's some very clever backronym here, but actually it was introduced to me by a guy named Steve. It's almost certainly not original, because it's actually common sense.

# TL;DR

{% paper(type="note") %}

1. Figure out the disparity between the target salaries and current salaries of your staff.
2. Get a budget for pay rises.
3. Assuming the budget isn't enough to cover the total disparity, allocate proportionally.

{% end %}

Simples.

# The philosophy

To apply this approach, you need a few things in place. The first thing you need is a philosophy of compensation. Here it is: *Your salary is compensation for your time.* Your employer pays you to do work for them instead of growing mung beans, or playing Pokemon Go, or reading Baudelaire, or whatever else you kids get up to. That's why you get paid every month, it's a transaction where the employer _buys_ your time and attention.

*Your salary should be dictated by the market rate for your time*. In a perfect world, your employer shouldn't pay you more than your time is worth, and you shouldn't accept _less_ than your time is worth. Now this is up for some debate, because you might choose to work for less than market rate at a start up, or a registered charity or what have you.

*Pay rises are to correct disparities between market rate and current compensation.* Over time, you hire people, you promote people, and so disparities will inevitably creep in. Some people will be over paid, others will be underpaid. The purpose of pay rises is to _correct_ these imbalances as best you can. Pay rises are *not* a reward, because your salary is not a reward: it is your fair price for your time.

With all of that said, let's talk about the pre-requisites, and set up a scenario.

# Pre-requisites

There are two-and-a-half other things you need to have understood before you get started.

## A Compensation Strategy

Your compensation strategy is the decisions you make with respect to equity and market rate. Here are some viable options:

* We pay market rate. No more, no less.
* We pay the lower end of market rate. We make up for it with equity.
* We pay below market rate. We make up for it with a sense of mission, and a fantastic learning culture.
* We pay over market rate, because the work here is boring.

None of these are wrong. I work for an early-stage startup, and our pay scales are market rate, but we're working on fixing climate change, and everybody is really friendly. Also, we have stock options, and a very well provisioned fridge. Those factors are enough to keep people happy, for the most part.

I know people who work in finance, and earn double what I do, but they're shuffling around money on behalf of hedge funds.

It doesn't matter what your strategy _is_, so long as you choose one, and you apply it consistently, and you are honest with your employees about the trade-offs you're making. This is unlikely to be a decision that an engineering manager has authority to take. You'll need to pester your boss and your boss's boss to get a simple message in writing.

## A Set of Pay Bands

Okay, so once you know how you pay people (we pay average market rate, but we're super cool, and fighting to save the world), you can draw up a set of of _pay bands_. These should be relatively wide, so that there is room for people to get increases, but shouldn't overlap. For example, in London right now, you can probably find yourself

* A graduate engineer for £35,000-£45,000
* A junior engineer for £50,000-£60,000
* A mid-level engineer for £65,000-£75,000
* A senior engineer for £80,000-£90,000
* A principal engineer for £95,000-£110,000

And so those are a good starting point for pay bands. In practice, the exact numbers will depend on the stack you're using, your geographical location, and a host of other factors. It's worth engaging a recruitment consultancy to do this for you. Tell them what skills you're hiring for, and they will go and do the market research and tell you how much it will cost.

Every engineer should know which band they're in, and whereabouts they fall. They should also know at least the lower-bound of the band above, so that they know what's waiting for them in the future. You might want to publish the whole pay structure, but be mindful that other parts of the organisation might pay significantly less, and it's not necessarily helpful.

## A growth framework

The last pre-requisite is a growth framework that helps you correctly level your engineers. There are a whole bunch of these available to crib from at [levels.fyi](https://levels.fyi). At [CarbonRe](https://carbonre.com), we stole a bunch of material from [Rent The Runway](https://dresscode.renttherunway.com/blog/ladder) because we liked the playful, D&D character sheet approach. 

Depending on the size of your organisation, your framework will be larger or smaller, don't fret. Right now, at CRe, we have four defined levels: junior engineer, engineer, senior engineer, and lead. That's more than enough.

Once you've written something up, you'll need to spend time with each engineer to work out where _they_ think they are on that ladder, and where _you_ think they are so that you can get buy-in.

# The procedure

Whew! That's a lot of stuff to do, and you should expect that to take you a few weeks: agree a compensation strategy with the higher-ups, get a benchmark for pay bands, and introduce a careers framework.

Now you're ready.

Meet our sample team. Our sample team are widget chuckers, and they are paid in W-bucks. We have five team members.

{% paper(type="note") %}

* Amara - Junior widget chucker. First hire in the org, capable, and learning fast.
* Björn - Senior widget chucker. Recent hire, negotiated hard in interview.
* Celeste - Mid-level widget chucker, quiet but chucks lots of widgets.
* Diego - Junior widget chucker, struggling to settle into the company.
* Ezgi - Senior widget chucker, performing well.

{% end %}

Their current pay is as follows

{% paper(type="note") %}

* Amara - W$70
* Björn - W$140
* Celeste - W$80
* Diego - W$80
* Ezgi - W$130

Total: *W$500*

{% end %}

After talking to the recruitment consultants, we draw up the following pay bands.

{% paper(type="note") %}

* Junior widget chucker: W$75-W$90
* Mid-level widget chucker: W$95-W$100
* Senior widget chucker: W$110-W$140

{% end %}

And then we assess each employee against the careers framework. We decide the following:

{% paper(type="note") %}

* Amara - Should be promoted to mid-level.
* Björn - Lower end of senior.
* Celeste - Solid mid-level.
* Diego - Lower-end of junior.
* Ezgi - Upper end of senior.

{% end %}

We use that to derive the _target salary_ for each employee. In other words, if we were starting again, how much _should_ we pay these people as fair compensation for their time?


<div class="notepaper">

| Employee | Current Salary | Target Salary | Suggested increase |
|----------|----------------|---------------|--------------------|
| Amara    |             70 |            95 |                 25 |
| Bjorn    |            140 |           120 |                  0 |
| Celeste  |             80 |            85 |                  5 |
| Diego    |             80 |            80 |                  0 |
| Ezgi     |            130 |           140 |                 10 |
| *Total*  |                |               |                 40 |

</div>

It turns out, to everybody's shock and horror, that we systematically underpay our female employees, and our earliest employees. Everybody is very embarassed.

Realistically, we're not going to cut Bjorn's pay, so we'll just have to live with him being overpaid and chalk it up to experience. If we want to correct all the other imbalances, it will cost us an extra 40 w-bucks. So you take that number, and you go talk to the CEO and say "please can I have 40 w-bucks extra budget to correct historical imbalances in our pay?" and he laughs at you.

"No," he says, "you can have 20 w-bucks extra for next year".

So what do we do now? We _divide the target by the available increase_. 

We want to hand out 40 w-bucks of pay rise, but we only have 20, so we simply halve the allocation.


<div class="notepaper">

| Employee | Current Salary | New Salary | Increase |
|----------|----------------|------------|----------|
| Amara    | 70             | 83         | 13       |
| Bjorn    | 140            | 140        | 0        |
| Celeste  | 80             | 82         | 2        |
| Diego    | 80             | 80         | 0        |
| Ezgi     | 130            | 135        | 5        |
| *Total*  | 500            | 520        | 20       |

</div>

# The consequences

So in our worked example, Amara got a huge uplift, while Celeste got a modest bump. Our two overpaid employees got nothing. In practice, you might need to _fudge_ a little to avoid people running away, but in theory what we've done is use the available funds to *close the biggest gaps*.

As you iterate this process, people's actual salary will converge toward their *fair compensation*.

As a nice side-effect, it's now easy to explain to any given engineer why they did, or did not, get a pay rise:

* You were earning W$80.
* Your target salary is W$85.
* I only have budget for 50% of the gap, ergo
* You now earn W$82

All this requires is a bit of set up, and the willingness to honestly evaluate your employees against the careers framework. 
