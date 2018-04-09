---
title: Manipulate events using a middleware
author: Georgos Landin Xanthopoulos
date: '2018-04-09 13:39:40 +0200'
header:
  teaser: images/posts/events.jpg
  caption: "Photo credit: [**Flickr**](https://www.flickr.com/photos/jone_vasaitis/16022260923/in/photolist-qpQgXZ-asu2YH-63NQ3W-85Gk4g-aEUFCZ-4YvJCc-7pTvgs-7LmUC9-eHjnFV-8E6J8V-hXw713-5DudKc-4VxoHr-9PD8vo-8FHkig-bAACjP-6tpDjd-ab2WT-54tSFr-pqLvEd-dL7Ksg-24g6kGj-ov9pFN-aBQ4zJ-8byiTm-atRt3o-ygkPwT-6SfG7U-24BjkTY-WyZ1Jx-ejbULs-8HjSq7-65nqnz-6G5nvX-22K5Rei-yjR3d-sDFU8-abosaA-cSdMUo-vCuvY-4jKtyC-22wib-cd2Bm3-FkXEeW-6MucmN-dd9ak6-78nBuc-SFNBS8-YYf8G3-5RLXMQ)"
categories:
- Symfony
---

This post will cover how you can add additional data to your events using a simple bus middleware. 

Events are great due to the fact that they are very versatile and can be very useful in most cases.
Maybe you want to log something specific in your system, or maybe you are building a gamification application that gives
users experience points based on their actions. Then events might just be right for you. 

## How it works
The main point is to do some kind event manipulation on those events that have already been dispatched inside your application. 
Maybe you want to save the events to a log, or maybe add some more data to the events before passing them on to another 
application.
Your middleware will accumulate these existing events, and dispatch a new event, that passes on the existing ones.
Below is a section of the middleware class, the entire middleware is shown at the bottom of this post. 

{% highlight php %}
$eventData = [
    // .. Add some event data
]
// Add the data to a DataContainer object
$container = new DataContainer($eventData);

$this->eventDispatcher->dispatch('foo', new Foo($event, $container));
{% endhighlight %}

### The dispatched event
The Foo event class uses the DataContainer object instead of a
simple array, since an array is passed by value and not by reference.

{% highlight php %}

class Foo extends Event
{
    private $event;
    private $container;

    public function __construct($event, DataContainer $container)
    {
        $this->event = $event;
        $this->container = $container;
    }

    public function getEvent()
    {
        return $this->event;
    }

    public function getContainer(): DataContainer
    {
        return $this->container;
    }
}
{% endhighlight %}

### The DataContainer class
Below is the DataContainer class, used to add some new data to the existing event. 

{% highlight php %}

class DataContainer
{
    /** @var array */
    private $data;

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

## Manipulate the event
Now you will need something that can manipulate the existing event. Creating an event subscriber that subscribes to the 'foo'
event is a good start. Now it's up to you to decide what your subscriber should do.   
In this case imagine that we need a users age and address for later use, but they are both missing in the event data that
was provided from our application. This means that we somehow want to add these values to the event, before it gets passed
on to another application. This will be done by the decorator class, which is a simple event subscriber. 

### Example of a decorator class
This example refers to when a new user has registered to your application.
A decorator class listens to the event that was dispatched by your middleware. 

The purpose of this event subscriber is to add specific data to your event. 

{% highlight php %}

class UserHasRegisteredDecorator implements EventSubscriberInterface
{
    public static function getSubscribedEvents()
    {
        return [
            'foo' => 'decorate',
        ];
    }
    /**
     * Populates the event with more specific data based on what is needed for the specific event.
     *
     * @param Foo $event
     */
    public function decorate(Foo $event)
    {
        // Retrieve the DataContainer object
        $container = $event->getContainer();

        // Check if the message and assigned event match
        if ($event->getEvent() instanceof UserHasRegistered) {

            // Add the data you want
            $container->addData('age', '23');
            $container->addData('address', 'foobarbaz');
        }
    }
}
{% endhighlight %}

Now we have successfully added age and address to the specific event data before it gets passed on to another application. 

## The middleware
Down below you can see the entire middleware class. 
{% highlight php %}

class Middleware implements MessageBusMiddleware, EventSubscriberInterface
{
    private $eventDispatcher;
    private $events = [];

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

    public function handle($event, callable $next)
    {
        $this->events[] = ['event' => $event, 'time' => time()];

        $next($event);
    }

    public function publish()
    {
        // Iterate through all events
        foreach ($this->events as $event) {

            // Add some message data
            $eventData = [
                'class' => get_class($event),
                'time' => $event['time']
            ]

            // Add the data to a DataContainer object
            $container = new DataContainer($eventData);

            // Dispatches the new ExtractSimpleBusMessage event
            try {
                // Dispatches the message and the DataContainer object
                $this->eventDispatcher->dispatch('foo', new Foo($event, $container));
            } catch (\Throwable $e) {
                $this->log('alert', '', ['exception' => $e]);
            }
        }
        $this->events = [];
    }
}
{% endhighlight %}