---
title: Real asynchronous events with Symfony2
author: Tobias Nyholm
date: '2014-05-15 19:33:15 +0200'
categories:
- Library
- Symfony
- Performance
- Java
---

PHP is a great language but there are some drawbacks. One thing I miss the most is multi-threading. Wouldn't it be great to be able to do some calculations in a separate thread? Or start some background task and don't care when it finish. I've found a solution that does that very thing with PHP.


It is quite strait forward but it might be little tricky to understand. Here is a picture that will make everything crystal clear.

![]({{ "/images/posts/symfony2events.jpg" | absolute_url }})

<h2>Symfony2 events</h2>

The key here is to use Symfony2 events. The event object can contain any serializable data as you like. That means that you should not store the User object in the event. You should prefer the User id. With <a href="https://github.com/fervo/FervoDeferredEventBundle">Fervo Deferred Event Bundle</a>  you can make the events to run in a different request. A event listener can say to the dispatcher that it should dispatch the event later. Likewise can the publisher say that all listeners for the event should be dispatched later.


What the Fervo Deferred Event Bundle does is to put these event on a message queue. Any message queue you like. I use RabbitMQ.

<h2>Message queue and workers</h2>

When the event is put on the message queue the PHP script will forget all about the event and finish making the response to the user. What happens next is that an other application will fetch the object form the message queue and tell PHP to start execute the event. That "other application" may be  <a href="https://github.com/fervo/deferred-event-worker">FervoDeferredEventWorker</a> (written in Ruby) or <a href="https://github.com/HappyR/DeferredEventJavaWorker">HappyrDeferredEventJavaWorker</a> (written in Java).


I wrote the Java application because I'm not a Ruby programmer and I wanted to add some features. With the Java worker you may have the same message queue and worker for multiple Symfony applications. The Java worker does also handle PHP errors. When a error occur the worker will store the error message and the event on the queue. That will make it easy to rerun the event when you've fixed the error.

<h2>Performance</h2>

With this setup you will have more control over you background tasks and you can run them as soon as there is CPU available. You probably want to know why you should use this instead of running cronjobs or Beanstalkd. That is a fair question. The answer is stability and performance. PHP-FPM is far better to handle <span style="color: #0d0d0d;">stability</span> problems than you are with Beanstalkd. It is also very expensive to spawn new processes with cronjobs. Magnus from Fervo <a href="http://joiedetech.se/2013-11-25-improving-symfony-workers">wrote an article</a> about this and he has also done some  performance tests.

<h2>PHP-FPM pools</h2>

There is other solutions for this. There is an article on <a href="http://puffingdev.com/async-eventdispatcher-in-symfony/">PuffingDev.com</a> where they do not use a message queue. But they dispatch the event after the respons is sent to the user. This is a fine solution but it is not optimal. Say that you have 3 PHP-FPM threads in 1 pool. Assume that each request serves the response in 1 sek but it runs a background task for 29 sek.


When 10 of these requests coms at T=0 it will serve 3 of them at T=1. The next 3 will be served at T=31 and the last will be served at T=91.


With the solution I've presented in this document you will probably set up a second PHP-FPM pool where you have one thread. You should also give that thread lower CPU priority than you thread for your real visitors. So you have 2 threads for serving responses and one for background tasks. This is not possible with the solution form PuffingDev.


When the same 10 requests comes at T=0 it will serve the first 2 at T=1. The next 2 at T=2 etc etc and the last request will be served at T=5. The background processes however will not finish until T=290. But that is okey. We don't really care when any background processes is done as long as we know that they will be executed as soon as possible.

