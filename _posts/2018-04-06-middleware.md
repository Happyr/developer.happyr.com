---
title: Manipulate events using a middleware
author: Georgos Landin Xanthopoulos
date: '2018-02-02 15:06:47 +0200'
header:
  teaser: images/posts/routelarge.jpg
  caption: "Photo credit: [**Flickr**](https://www.flickr.com/photos/rockys_photos/355216949/in/photolist-6enFxo-KGos4s-bziePy-91523N-TpGWrG-dvMSg3-6ZREpd-xozCa-3CVfrH-4r4Dz7-pkqvKY-5CL4FN-fHW3WW-8vvEYu-4KfNRw-6xwQuD-6xHB5q-8xo6sb-7dphxn-7KgWrr-342fEp-6xX2tf-6dBTjp-9nB8TA-8xk5pz-P61pjV-dNXAvC-5n8o7A-u7GL9y-6jzRhi-MGt7o1-7A7jLS-i4tC8C-4hDeBQ-3eX22G-dvGvj4-7A3gUD-5b6sLw-5yjxW2-8hxfmE-4cTxKp-Cq8sKM-tQg5e-7A7m1d-6ygJN3-avLA1p-9mQJiX-8ZtQtV-rpFoVZ-7A1DLZ)"
categories:
- Symfony
---

Events are great due to the fact that they are very versatile and can be very useful in most cases.
Maybe you want to log something specific in your system, or maybe you are building a gamification application that gives
users experience points based on their actions. Then events might just be right for you.

This post will cover how you can add additional data to your events using the simple bus middleware.
## How it works
Classes needed:
* Middleware - dispatches the ExtractSimpleBusMessage event
* ExtractSimpleBusMessage - uses an event and a DataContainer class
* DataContainer - modifies the array given to it, through multiple available functions.
* A decorator class for each event - adds data to the event it's assigned to

In most cases, your middleware will accumulate a number of events, also known as messages. When your application terminates
, the middleware will dispatch a SimpleBusMessage event for each accumulated message.
The decorator classes are event subscribers and listen to the ExtractSimpleBusMessage event, they check the simple bus message
for their assigned event, and if there is a match, they add some more data to it.
This was just a quick summary of the process, further down you will see the implementation.

## The middleware
Down below you can see the middleware class. As you can see, it is an event subscriber that dispatches an ExtractSimpleBusMessage
event for each accumulated message once the symfony 'kernel.terminate' event is dispatched.

{% highlight php %}

class Middleware implements MessageBusMiddleware, EventSubscriberInterface
{
    use LoggerTrait;
    const EXTRACT_SIMPLEBUS_MESSAGE = 'extract.simplebus.message';

    /**
     * @var EventDispatcherInterface
     */
    private $eventDispatcher;

    /**
     * @var array
     */
    private $messages = [];

    public function __construct(EventDispatcherInterface $eventDispatcher)
    {
        $this->eventDispatcher = $eventDispatcher;
    }

    public static function getSubscribedEvents()
    {
        return [
            KernelEvents::TERMINATE => 'publish',
        ];
    }

    public function handle($message, callable $next)
    {
        $this->messages[] = ['message' => $message, 'time' => time()];

        $next($message);
    }

    public function publish()
    {
        // Iterate through all messages(events)
        foreach ($this->messages as $message) {

            // Add some event data
            $messageData = [
                'class' => get_class($message),
                'time' => $message['time']
            ]

            // Add the data to a DataContainer object
            $container = new DataContainer($messageData);

            // Dispatches the new ExtractSimpleBusMessage event
            try {
                // Dispatch event together with the message and the DataContainer object
                $this->eventDispatcher->dispatch(self::EXTRACT_SIMPLEBUS_MESSAGE, new ExtractSimpleBusMessage($message, $container));
            } catch (\Throwable $e) {
                $this->log('alert', '', ['exception' => $e]);
            }
        }
        $this->messages = [];
    }
}

{% endhighlight %}

As you can see, when dispatching the ExtractSimpleBusMessage event, you will need to pass the event object, and the DataContainer.

{% highlight php%}
// Dispatches the new ExtractSimpleBusMessage event
try {
    $this->eventDispatcher->dispatch(self::EXTRACT_SIMPLEBUS_MESSAGE, new ExtractSimpleBusMessage($message, $container));
} catch (\Throwable $e) {
    $this->log('alert', '', ['exception' => $e]);
}
{% endhighlight %}

Below you can see the ExtractSimpleBusMessage class. The class in it self is very simple, it takes the message, and the
DataContainer object as parameters, and adds some getters. This is what makes it possible for our decorators to access
the DataContainer and add additional data to the message.

{% highlight php %}
<?php

class ExtractSimpleBusMessage extends Event
{
    /** @var object */
    private $simpleBusMessage;

    /** @var DataContainer */
    private $container;

    public function __construct($simpleBusMessage, DataContainer $container)
    {
        $this->simpleBusMessage = $simpleBusMessage;
        $this->container = $container;
    }

    public function getSimpleBusMessage()
    {
        return $this->simpleBusMessage;
    }

    public function getContainer(): DataContainer
    {
        return $this->container;
    }
}
{% endhighlight %}

## The DataContainer class
Below is an example of how the DataContainer class could look like, as mentioned earlier, it's up to you to modify it
to your needs.

{% highlight php %}

class DataContainer
{
    /** @var array */
    private $data;

    /**
     * @param $data
     */
    public function __construct(array $data)
    {
        $this->data = $data;
    }

    public function addData(string $key, $value)
    {
        $this->data[$key] = $value;
    }
}

{% endhighlight %}

## Example of a decorator
Now lets see an example of a decorator class.
This example refers to when a new user has registered to your application.
The decorator class listens to the ExtractSimpleBusMessage event. When it is dispatched, it will run the
decorate() function.

{% highlight php %}

class UserHasRegisteredDecorator implements EventSubscriberInterface
{
    public static function getSubscribedEvents()
    {
        return [
            Middleware::EXTRACT_SIMPLEBUS_MESSAGE => 'decorate',
        ];
    }
    /**
     * Populates the event with more specific data based on what is needed for the specific event.
     *
     * @param ExtractSimpleBusMessage $event
     */
    public function decorate(ExtractSimpleBusMessage $event)
    {
        // Retrieve the DataContainer object
        $container = $event->getContainer();

        // Check if the message and assigned event match
        if ($event->getSimpleBusMessage() instanceof UserHasRegistered) {

            // Add the data you want
            $container->addData('age', '23');
            $container->addData('address', 'foobarbaz');
        }
    }
}
{% endhighlight %}

If the message provided by the ExtractSimpleBusMessage event was an UserHasRegistered event, then the decorator will add
'age' and 'address' to the container. This way you can add any data that you want for your events.