+++
title = "The Cake Shop Kata"
slug = "cake-shop-kata"
date = "2022-04-02"
category = "kata"

[extra]
author = "Bob Gregory"

[taxonomies]
tags = ["tdd", "kata", "modelling"]
+++

While spelunking through some code at work, I found a neat little programming problem that I've cleaned up a little and decided to publish as a TDD Kata.

In [the Cake Shop Kata](https://github.com/bobthemighty/cake-shop-kata) we're asked to write some code that can calculate the delivery date for different types of cakes, taking into account the skills and working patterns of the bakers. 


<!-- more -->

The original code I found was tough to understand and extend because there weren't any clear abstractions. It had grown from a simple set of requirements, but hadn't scaled well as the complexity grew. There were lots of special cases (if day is Sunday and cake is big...), and code that overwrote or reverse-engineered the work of previous steps (if it's Christmas, calculate how many weekend days we skipped, and subtract those from our delivery date).

Often when we're writing code in real-world systems, we're asked to implement relatively simple rules: an apple costs £1, baskets over £20 receive a 5% discount, three pieces of fruit cost £2, etc. The challenge comes in how these rules interact. Unless we can make our rules composable, the complexity grows super-linearly and we get stuck in a ball of mud.

The kata, with some test cases and a bootstrap for TypeScript (Python coming soon) is [up on Github](https://github.com/bobthemighty/cake-shop-kata/) but the requirements are below to save you a click.

{% paper(type="note") %}
Connascent Cakes is an artisan baker that sells custom-made cakes for delivery. Two friends, Sandro and Marco own the shop.
Marco does all the baking, while Sandro does decorations. 

Sandro and Marco want to start selling cakes online and need you to write code that can calculate the delivery date for their cakes.

The "lead time" is the number of days that it takes to make a cake. The delivery date is the date the cake was ordered, plus the lead time. For example, if a cake is ordered on the 1st of the month, and has a lead time of 2 days, the delivery date is the 3rd of the month.

* Marco works from Monday-Friday, and Sandro works from Tuesday-Saturday.
* Cakes are always delivered on the day they're finished. Nobody likes stale cake.
* Small cakes have a lead time of 2 days.
* Big cakes have a lead time of 3 days.
* If Marco receives a cake order in the morning (ie, before 12pm) he starts on the same day.
* Custom frosting adds 2 days extra lead time. You can only frost a baked cake.
* The shop can gift-wrap cakes in fancy boxes. Fancy boxes have a lead time of 3 days. Boxes can arrive while the friends are working on the cake,
* The shop can decorate cakes with nuts. Unfortunately, Sandro is allergic to nuts, so Marco does this job. Decorating a cake with nuts takes 1 extra day, and has to happen after any frosting has finished.
* The shop closes for Christmas from the 23rd of December and is open again on the 2nd of January. Cakes that would be complete in that period will be unable to start production until re-opening. Fancy boxes will continue to arrive throughout the festive period.

{% end%}

I'm planning to run a workshop at Cazoo to try this kata in an ensemble to practice our skills at modelling complex real-world problems.

If you try it, let me know on [Twitter](https://twitter.com/bob_the_mighty). I'm planning to write up my own attempt, warts and all, in a future post.
