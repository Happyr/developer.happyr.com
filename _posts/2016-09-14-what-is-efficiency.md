---
status: publish
published: true
title: What is efficiency?
author: Tobias Nyholm
author_login: Tobias Nyholm
author_email: tobias@happyr.com
wordpress_id: 742
wordpress_url: http://developer.happyr.com/?p=742
date: '2016-09-14 09:35:00 +0200'
date_gmt: '2016-09-14 07:35:00 +0200'
categories:
- Developing
tags: []
comments: []
---

<strong>The answer may be trivial. Efficiency is when you work as hard as you can and as quick as possible to complete a
task. The answer could also be a bit more complicated…</strong>

My MacBook just broke, I googled the issues I was experiencing and I found out that this is a common error for my particular
model. I went to the Apple Store and the “geniuses” made the same conclusion as I did and sent my computer to the reparation
service. It was in a service queue for 4 days before a technician touched my computer. He/She ran a test and ordered some
parts that arrived 2 days later. The technician spent 30 minutes with my computer to run the test. When my parts finally
arrived it took 2 hours to repair.

I know for sure that this technician was working as hard and quick as possible, but still it took 6 days to complete 2.5
hours of work. Is that really efficient?

Working hard and fast is a type of efficiency called <em>resource efficiency</em>. Another example of resource efficiency
could be that you are using all your machines 100% of the time. What I, as a customer, care about is not resource efficiency
but <em>flow efficiency</em>. This is the type of efficiency that consider a unit that goes through a process.

A way for the Apple repair service to improve on their flow efficiency would be to spend 20 minutes on computers with common
errors as soon there is an incoming repair to see if there is a quick way to diagnose the issue. That would drastically
reduce the average repair time. Another way would be to have common parts in stock.

The interesting question is: What could we, as developers, learn from this experience? How could we be more <em>flow efficient</em>?
First we need to understand what processes we have and then what the unit is that flows within that process. It could be
an issue report or a Scrum ticket that will go through the process of your work to be marked as complete. It could also be
some form of data processing in your application. You could maybe identify some data input that are trivial that may be allowed
to cut the line (queue). You could maybe also predict that some input data need external dependencies that you could start
to fetch while the input data is still in the queue.

This post has hopefully given you an alternative way of thinking on efficiency, if you want to learn more about this I would
recommend “This is Lean” by Niklas Modig and Pär Ahlström.

