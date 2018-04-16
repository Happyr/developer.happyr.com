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

This post will cover how you can add additional data to your SimpleBus events using a SimpleBus middleware. 

Using SimpleBus in your hexagonal architecture is great, due to the versatility it gives you when working with events/commands. 
If you would like to read more about the event bus or the command bus and their associated handlers/subscribers, see 
the [documentation](http://docs.simplebus.io/en/latest/index.html).  

Maybe you want to log something specific in your system, or maybe you are building a gamification application that gives
users experience points based on their actions. Then using SimpleBus might be just right for you. 

## How it works
The main point is to do some kind event manipulation on those events that have already been dispatched inside your application. 
Maybe you want to save the events to a log, or maybe add some more data to the events before passing them on to another 
application.

### The dispatched event
The event class will take the DataContainer object as a parameter instead of an array, 
since an array is passed by value and not by reference. Therefore by using the DataContainer object we can 
make sure that the passed parameter has the correct value. 

{% highlight php %}

use Symfony\Component\EventDispatcher\Event;

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
Now you will need something that can manipulate the existing event. Creating an symfony event subscriber that subscribes to the 'foo'
event is a good start. Now it's up to you to decide what your subscriber should do.     

In this case imagine that we need a users age and address for later use, but they are both missing in the SimpleBus event data that
was provided from our application. This means that we somehow want to add these values to the event, before it gets passed
on to another application. This will be done by the decorator class displayed below. 

### Example of a decorator class
This example refers to when a new user has registered to your application.

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
        
            $user = $this->getEntityManager()->getRepository(User::class)->find($event->getId()); 
            // Add the data you want
            $container->addData('age', $user->getAge());
            $container->addData('address', $user->getAddress());
        }
    }
}
{% endhighlight %}

Now we have successfully added age and address to the event data before it gets passed on to another application. 

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