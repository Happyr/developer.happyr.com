---

status: publish
published: true
title: Real asynchronous events with Symfony2
author: Tobias Nyholm
author_login: Tobias Nyholm
author_email: tobias@happyr.com
wordpress_id: 507
wordpress_url: http://developer.happyr.com/?p=507
date: '2014-05-15 19:33:15 +0200'
date_gmt: '2014-05-15 17:33:15 +0200'
categories:
- Library
- Symfony2
- Performance
- Java
tags: []
comments:
- id: 371
  author: cordoval
  author_email: cordoval@gmail.com
  author_url: http://craftitonline.com
  date: '2014-05-26 01:55:16 +0200'
  date_gmt: '2014-05-25 23:55:16 +0200'
  content: you could have done the java part with php also right? just curious, thanks
    good post
- id: 373
  author: Tobias Nyholm
  author_email: tobias@happyrecruiting.se
  author_url: ''
  date: '2014-05-26 14:57:24 +0200'
  date_gmt: '2014-05-26 12:57:24 +0200'
  content: "Java (and many other languages, but not PHP) is made for such long running
    tasks. But sure you could, but it is not preferable. If you would write the worker
    in PHP you could (A) use a cron job to start a new thread every time it runs (bad
    for performance) or (B) use beanstalkd and deal with stability problems and memory
    leaks. \r\n\r\nDoing it with Java is simpler and better."
- id: 418
  author: Mmm
  author_email: mmm@yopmail.com
  author_url: ''
  date: '2014-06-06 09:18:59 +0200'
  date_gmt: '2014-06-06 07:18:59 +0200'
  content: 'I would not entirely agree with java vs php in that case. Have you tested
    scenario with long-running php process? I didn''t have problems with it, and as
    long as you use message queue, you can easily restart workers automatically and
    there''s one advantage: you can share code between workers and core application
    and you simplify workflow (imagine that you use Doctrine and you have listeners
    bound to persistance events, if you''d try to save data with external Java worker
    they''d be just an obstacle).'
- id: 419
  author: Tobias Nyholm
  author_email: tobias@happyrecruiting.se
  author_url: ''
  date: '2014-06-06 09:55:26 +0200'
  date_gmt: '2014-06-06 07:55:26 +0200'
  content: "I've tried that yeas ago. I know that Magnus got that setup running on
    one of his clients (http://joiedetech.se/2013-11-25-improving-symfony-workers).
    And if you use Java you don't need to restart the worker... Since it does not
    leak memory nor crash. =)\r\n\r\nYou don't want to share the code base. The workers
    only job is to fire up the PHP script (with PHP-FPM). So the worker itself has
    nothing to do with the application."
- id: 11646
  author: CSchulz
  author_email: christian.schulz@ewetel.net
  author_url: ''
  date: '2014-10-10 11:43:47 +0200'
  date_gmt: '2014-10-10 09:43:47 +0200'
  content: "I think in this context the SonataNotificationBundle is another good option.\r\n\r\nThere
    you have the advantage all code is PHP and you can use the Database as \"message
    queue\", but you have to implement an own method to talk with the FPM and execute
    the long running php process."
- id: 81490
  author: dizda
  author_email: dizzda@gmail.com
  author_url: ''
  date: '2015-03-13 00:11:10 +0100'
  date_gmt: '2015-03-12 23:11:10 +0100'
  content: Hmm I didn't know about this FervoDeferredEventBundle, this is interesting...
    Nice article Tobias!
- id: 95150
  author: Tez
  author_email: tesmond@gmail.com
  author_url: http://tesmond.blogspot.com
  date: '2015-06-24 17:16:15 +0200'
  date_gmt: '2015-06-24 15:16:15 +0200'
  content: "\"One thing I miss the most is multi-threading. \"\r\n\r\nYou can use
    <a href=\"http://php.net/manual/en/book.pthreads.php\" rel=\"nofollow\">pthreads
    </a> for multi-threading.\r\n\r\nOr another slightly cludgy solution is to use
    <a href=\"http://stackoverflow.com/questions/1019867/is-there-a-way-to-use-shell-exec-without-waiting-for-the-command-to-complete\"
    rel=\"nofollow\">exec / shell_exec</a> and set the output to null.\r\n\r\nThere
    are other suspect solutions using file system triggers available too.\r\n\r\nRabbit
    mq does allow multiserver listeners which makes things a little more powerful
    and interesting, but it is important to note that proper and pseudo multi-threading
    in php is possible just extremely uncommon."
- id: 106102
  author: branchbit
  author_email: contact@branchbit.be
  author_url: http://branchbit.be
  date: '2015-11-05 11:48:38 +0100'
  date_gmt: '2015-11-05 10:48:38 +0100'
  content: "Theres also this : https://packagist.org/packages/bbit/sqs-command-queue-bundle\r\n\r\nIts
    a very simple bundle, wich you can use, to queue commands on amazon SQS, you can
    then have several workers, on several servers, handle these tasks asynchronously"
- id: 106103
  author: Tobias Nyholm
  author_email: tobias@happyr.com
  author_url: ''
  date: '2015-11-05 12:11:07 +0100'
  date_gmt: '2015-11-05 11:11:07 +0100'
  content: That is great! There is also SimpleBus by Matthias Noback which is awesome
    =)
- id: 111447
  author: Maksym Kotliar
  author_email: kotlyar.maksim@gmail.com
  author_url: ''
  date: '2017-05-16 18:01:23 +0200'
  date_gmt: '2017-05-16 16:01:23 +0200'
  content: "I am working on an alternative solution https://github.com/php-enqueue/enqueue-dev/pull/86\r\n\r\nCould
    you please look at. The feedback would be much appreciated. \r\n\r\nThanks."
---

PHP is a great language but there are some drawbacks. One thing I miss the most is multi-threading. Wouldn't it be great to be able to do some calculations in a separate thread? Or start some background task and don't care when it finish. I've found a solution that does that very thing with PHP.


It is quite strait forward but it might be little tricky to understand. Here is a picture that will make everything crystal clear.

![]({{ "/assets/images/symfony2events.jpg" | absolute_url }})

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

