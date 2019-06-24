---
title: Symfony Messenger on AWS Lambda
author: Tobias Nyholm
date: '2019-03-24 09:13:00 +0200'
header:
  image: "images/posts/non-blocking-fonts.png"
  teaser: images/posts/non-blocking-fonts.png
categories:
- Bref
- Messenger
- Symfony
---

For the past 4 years we have developed web applications with a message bus. The Symfony Messenger component was released
around a year ago and I've loved it since. In the past few weeks we've been deploying applications on AWS Lambda. Going
serverless is super cool since you can scale up and down, it is cheap bla bla bla. The major selling point to me is that 
you dont need to care about dev ops once it all up and running. We've been using [Bref](https://bref.sh) and I can highly 
recommend it. 

When running your application on Lambda, most things work just as normal but there is one thing in particular to be aware
of. Lambda is not meant to be running and pull on a queue. But that is exactly what is expected when using Symfony's Messenger
component with async messages. You are expected to run `bin/console messenger:consume` and keep that alive with Supervisord. 
Instead, the queue should make sure to start a Lambda function. 

So on AWS there are two major queue like services: SQS and SNS. They are pretty similar but SQS is more of a classic queue
like RabbitMQ. SNS is more of a publish-subscribe system. We've decided to use SNS because pub-sub is pretty much what we 
want to do. We don't have the issue that "the server might get too busy" which you would have if you ran on a normal server. 
This is Lambda, resources are indefinite. 

So the first thing I did was to configure my Messenger transport. We have to use php-enqueue to integrate with SNS and
Samuel Roze's messenger to php-enqueue bridge. 

{% highlight shell %}
composer require sroze/messenger-enqueue-transport "enqueue/sns:dev-master"
{% endhighlight %}

{% highlight yaml %}
framework:
    messenger:
        transports:
            sns:
                dsn: 'enqueue://foobar?topic[name]=my_sns_topic'

        buses:
            # ...
        routing:
            App\Message\Event\UserCreated: sns

enqueue:
    foobar:
        transport:
            dsn: "sns://foo" # Must start with "sns:". Topic details come from framework.mesenger.transport.sns.dsn
            connection_factory_class: 'Enqueue\Sns\SnsConnectionFactory'
            key: '%env(AWS_KEY)%'
            secret: '%env(AWS_SECRET)%'
            region: '%env(AWS_TARGET_REGION)%'
        client: ~
{% endhighlight %}

This will successfully publish messages to the SNS topic named "my_sns_topic". Now we need to create a new Lambda function
to consume the messages. We will use the exact same source code but deployed with this Sams template. 

{% highlight yaml %}
Resources:
    Consumer:
        Type: AWS::Serverless::Function
        Properties:
            FunctionName: 'my-app-consumer'
            Handler: bin/message-consumer
            Timeout: 20 # in seconds
            MemorySize: 2048
            # ...
            Events:
                Sns:
                    Type: SNS
                    Properties:
                        Topic: arn:aws:sns:eu-central-1:123456789:my_sns_topic
{% endhighlight %}

This will invoke a script at `bin/consumer` for every message pushed to "my_sns_topic". The job of this script is to 
read the SNS event and give it back to the SNS transformer. 

Lets start by defining the `bin/consume` script:

{% highlight php %}
#!/usr/bin/env php
<?php

use App\Kernel;
use App\Consumer\SnsConsumer;

require dirname(__DIR__).'/config/bootstrap.php';
require dirname(__DIR__).'/vendor/autoload.php';

lambda(static function (array $event) {

    $kernel = new Kernel($_SERVER['APP_ENV'], (bool) $_SERVER['APP_DEBUG']);
    $kernel->boot();
    $container = $kernel->getContainer();

    /** @var SnsConsumer $consumer */
    $consumer = $container->get(SnsConsumer::class);

    foreach ($event['Records'] as $record) {
        if (!isset($record['Sns']) && !isset($record['Sns']['Message'])) {
            continue;
        }

        $consumer->consume([ 'body' => $record['Sns']['Message']]);
    }

    return 'OK.';
});
{% endhighlight %}

This will call `SnsConsumer::consume`. The `SnsConsumer` must be a public service. I use the `SnsConsumer` because it
is easier to do dependency injection to a service than defining serivices as public and use them directly in `bin/consume`.

{% highlight php %}
// src/Consumer/SnsConsumer.php

namespace App\Consumer;

use Symfony\Component\Messenger\MessageBusInterface;
use Symfony\Component\Messenger\Transport\Serialization\SerializerInterface;

final class SnsConsumer
{
    private $bus;
    private $serializer;

    public function __construct(MessageBusInterface $bus, SerializerInterface $serializer)
    {
        $this->bus = $bus;
        $this->serializer = $serializer;
    }

    public function consume(array $input)
    {
        $envelope = $this->serializer->decode($input);
        $this->bus->dispatch($envelope);
    }
}

{% endhighlight %}

When defining the `SnsConsumer` service, make sure you select the correct serializer, ie the serializer you used when you 
published the message to SNS. You should also consider using the `Symfony\Component\Messenger\RoutableMessageBus` (`messenger.routable_message_bus`)
to automatically find the correct bus four you message. 

That is it. You have now setup async events with Lambda and Bref. You can of course have your published application on a
normal server and the workers on Lambda. 

## One more thing

We use SNS to communicate between applications. Messenger works great but it requires you to have the same namespace for 
your message for both application. That is not really ideal. The communication will also break if one application modifies 
message, like renaming a private property. 

A better solution is to **transform** the message to an array then send it to SNS. The other app recieves the array and 
hydrates it to a message. This will ensure the array is the contract, not the message. You have much more freedom to refactor, 
move or change messages. You can also easier version your messages. 

To achive this we used [Happyr Message Serializer](https://github.com/Happyr/message-serializer) as a substiture for the 
default serializer used with messenger. We configure it like: 

{% highlight yaml %}

# config/packages/messenger.yaml

framework:
    messenger:
        transports:            
            to_foobar_application:
                dsn: 'enqueue://foobar?topic[name]=my_sns_topic'
                serializer: 'Happyr\MessageSerializer\Serializer'
{% endhighlight %}
            
See more example in the [readme](https://github.com/Happyr/message-serializer).
