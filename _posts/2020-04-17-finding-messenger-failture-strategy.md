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

So with a delay of 10ms, multiplier of 2: 
- Dispatch message: 0ms
- Retry 1: 10 * (2 ^ 1) = 20ms
- Retry 2: 10 * (2 ^ 2) = 40ms
- Retry 3: 10 * (2 ^ 3) = 80ms
- Retry 4: 10 * (2 ^ 4) = 160ms

When the ``max_retries`` limit has been reached, the message will be moved to the 
``failure_transport``. The failure transport is special and how one retry messages 
in that transport is described [in the Symfony documentation](https://symfony.com/doc/current/messenger.html#saving-retrying-failed-messages). 

## My retry strategy

If a message fail, I assume it is because of something temporary. Say that an HTTP 
request failed because the remote server was unavailable. So I want to retry that 
message fairly soon. That is why I start with a low delay. 

If that message has failed a few times, I want to wait to back off quickly. That is
why I use a multiplier of 4. But I also don't want it to wait too long. So I set
``max_delay`` to 30 000ms.

- Dispatch message: 0ms
- Retry 1: 1000 * (4 ^ 1) = 4.000ms
- Retry 2: 1000 * (4 ^ 2) = 16.000ms
- Retry 3: 1000 * (4 ^ 3) = 30.000ms
- Retry 4: 1000 * (4 ^ 4) = 30.000ms

After 4 retries the message is moved to the failure queue. 

The failure queue is also configured with retries, but in it reties more slowly.
Here I've configured the ``delay`` to 300.000 and a ``multiplier`` of 3. The ``max_delay``
is configured to 86.400.000 (1 day). 

- Retry 1: 15 min
- Retry 2: 45 min
- Retry 3: 2 hours 15 min
- Retry 4: 6 hours 45 min
- Retry 5: 20 hours 15 min
- Retry 6: 24 hours
- Retry 7: 24 hours
- Retry 8: 24 hours
- Retry 9: 24 hours

After 20 retries I've decided to remove the message completely. I assume (maybe incorrectly)
that if an error has not been fixed after more than 2 weeks. Then there is no point
of continue to retry. 

The Symfony configuration looks like this: 

{% highlight yaml %}
framework:
    messenger:
        failure_transport: failed
        transports:
            workqueue:
                dsn: '%env(MESSENGER_WORKQUEUE_DSN)%'
                retry_strategy:
                    max_retries: 4
                    # milliseconds delay
                    delay: 1000
                    max_delay: 30000
                    multiplier: 4
            failed:
                dsn: 'doctrine://default?queue_name=failed'
                retry_strategy:
                    max_retries: 20
                    # milliseconds delay
                    delay: 300000
                    max_delay: 86400000
                    multiplier: 3

{% endhighlight %}
