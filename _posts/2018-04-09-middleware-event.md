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

This post will cover how you can add additional data to your events using a middleware class.

Events are great due to the fact that they are very versatile and can be very useful in most cases.
Maybe you want to log something specific in your system, or maybe you are building a gamification application that gives
users experience points based on their actions. Then events might just be right for you.

## How it works
Classes needed:
* Middleware - dispatches an event, that the decorators listen to
* ExtractSimpleBusMessage - The event to be dispatched
* DataContainer - can manipulate the array given to it. Such as adding data.
* A decorator class for each event - adds data to the event it's assigned to

In most cases, your middleware will accumulate a number of events, also known as messages. 
When your application terminates, the middleware will dispatch a SimpleBusMessage event for each accumulated message.  


The decorator classes are event subscribers and listen to the ExtractSimpleBusMessage event, they check the simple bus message
for their assigned event, and if there is a match, they add some more data to it.
This was just a quick summary of the process, further down you will see the implementation.

## Example of a decorator
Lets see an example of a decorator class.
This example refers to when a new user has registered to your application.
A decorator class listens to a simble bus message. When the specific message is dispatched, it will run the
decorate() function.

The purpose of the decorator is to add specific data to your message. In this case, lets say that 
your message is missing the users age and address, and that you need those values for later use,
therefore, you decorate the message, and add the data that you need.  

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

## The middleware
Down below you can see the middleware class. As you can see, it is an event subscriber that listens for
the symfony 'kernel.terminate' event. 

{% highlight php %}

class Middleware implements MessageBusMiddleware, EventSubscriberInterface
{
    const EXTRACT_SIMPLEBUS_MESSAGE = 'extract.simplebus.message';
    private $eventDispatcher;
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
        // Iterate through all messages
        foreach ($this->messages as $message) {

            // Add some message data
            $messageData = [
                'class' => get_class($message),
                'time' => $message['time']
            ]

            // Add the data to a DataContainer object
            $container = new DataContainer($messageData);

            // Dispatches the new ExtractSimpleBusMessage event
            try {
                // Dispatches the message and the DataContainer object
                $this->eventDispatcher->dispatch(self::EXTRACT_SIMPLEBUS_MESSAGE, new ExtractSimpleBusMessage($message, $container));
            } catch (\Throwable $e) {
                $this->log('alert', '', ['exception' => $e]);
            }
        }
        $this->messages = [];
    }
}

{% endhighlight %}

Upon the 'kernel.terminate' event the middleware dispatches the ExtractSimpleBusMessage event, and you will need to pass the message, and the DataContainer.
This way we can access the message and the DataContainer inside our decorators with the use of the getter functions that are set
inside the ExtractSimpleBusMessage class.   

The ExtractSimpleBusMessage class uses the DataContainer object instead of a
simple array, since an array is passed by value and not by reference.

Below you can see the ExtractSimpleBusMessage class.

{% highlight php %}

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