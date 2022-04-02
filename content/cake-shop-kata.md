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

I'm planning to run a workshop at Cazoo to try this kata in an ensemble to practice our skills at modelling complex real-world problems.

If you try it, let me know on [Twitter](https://twitter.com/bob_the_mighty). I'm planning to write up my own attempt, warts and all, in a future post.
