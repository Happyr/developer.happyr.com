---
title: Finding the correct values for Symfony Messenger failure strategy 
author: Tobias Nyholm
date: '2020-04-17 10:44:00 +0200'
header:
  image: "images/posts/blackfire/header.png"
  teaser: images/posts/blackfire/header.png
categories:
- Symfony
- Messenger
---

Symfony Messenger is great. It has so many features and is super flexible. I want to 
highlight one of these features in this small post: Failures and retries. 

Over in the [Symfony docs](https://symfony.com/doc/current/messenger.html#retries-failures) 
falures and retires are briefly mentioned and it explains what values you can configure.
However, it does not give a good recommendation for what values to use. The simple 
answer to "What values should I use?" is "It's depends". 

Depending on what your application requirements are these values will obviously be 
different.

The parameters we can work with are: 

- max_retires: How many times we will try to process a message.
- delay: A time unit in milliseconds. 
- multiplier: Used with the ``delay`` to calculate the time to wait between retries.
- max_delay: The longest time between retries.

The complex thing here is how the time between retries is calculated. The default 
calculation could be found in ``Symfony\Component\Messenger\Retry\MultiplierRetryStrategy``. 
It says: ``delay * (multiplier ^ retry_count)``

So with a delay of 1ms, multiplier of 2

{% highlight php %}
blackfire.agent_socket = tcp://172.31.84.248:8307
blackfire.agent_timeout = 0.25

{% endhighlight %}
