---
title: Optimize write to neo4j recommendation engine 
author: Radoje Albijanic
date: '2019-07-03 11:24:00 +0200'
header: 
  image: "images/posts/neo4j.jpg"
  teaser: images/posts/neo4j.jpg
categories:
- Symfony
- Neo4j
- Recommendation engine
--- 

In today applications recommendations became quite necessary for better audience/customer targeting. So, as you could assume, we have our own recommendation engine for that, 
using Symfony and Neo4j as a database. Communication between the main application and the recommendation engine is done via messages sent to/consumed from RabbitMQ. 
We are sending some interesting events that happened on our website to the recommendation engine, and also pulling some suggestions from it.

##The problem

Over time we noticed something – our message queue got quite clogged, over 2 millions of messages were waiting to be written in the database! So we dug up a bit and found out 
that Neo4j is not quite optimized for one by one writing, it is recalculating relations after each insert, and we had large amounts of messages that were just coming and coming 
in and waiting to be processed. So we needed to fix that, 
in order to have up to date recommendations for our system.

##The investigation

After reading some articles, and Neo4j official docs, we found several guidelines:
- don’t use the ORM
- use [UNWIND](https://neo4j.com/docs/cypher-manual/current/clauses/unwind/)   

We decided to check how that really impacts the write queries to the database so we made a little experiment. We ran the insert of 500 entries with ORM, without ORM and using UNWIND:

{% highlight console %}
Started writing 500 entries with ORM.
 500/500 [============================] 100%
Started in 28 seconds.
Starting writing 500 entries with connection.
 500/500 [============================] 100%
Finished in 5 seconds.
Started writing 500 entries with connection and UNWIND.
Finished in 0 seconds.
{% endhighlight %}

The write with ORM took 28, using CYPHER 5, using UNWIND took below 1s!

##The solution

When checking the messages stuck on the queue we noticed one message (`AdvertServedToUser`) was 80% of the total messages. So we decided to move it out of ORM and use UNWIND. 
We created another queue for our batch messages, and when consuming them from the original queue – we just re-queued them to another. We were using [RabbitMqBundle](https://github.com/php-amqplib/RabbitMqBundle) for consuming messages from the queue and Symfony Messenger inside application.

We had to change routing for the specific message in order to put it on the different queue:

{% highlight yaml %}
#config/packages/messenger.yaml
framework:
    messenger:
        transports:
            batch_messages_amqp: '%env(BATCH_MESSAGES_TRANSPORT_DSN)%/advert-served-to-user
        ...
        routing:
            App\Message\Command\AdvertServedToUser: batch_messages_amqp

{% endhighlight %}

The next step was to get messages from the new queue in a batch of 50 or more (we ended up getting 5000 of them in one batch). In order to do that we had to create consumer class:

{% highlight php %}
<?php

declare(strict_types=1);

namespace App\Consumer;

use App\Message\Command\Advert\AdvertServedToUser;
use App\Message\Command\Advert\RunAdvertServedToUserBatch;
use OldSound\RabbitMqBundle\RabbitMq\BatchConsumerInterface;
use PhpAmqpLib\Message\AMQPMessage;
use Psr\Log\LoggerInterface;
use Symfony\Component\Messenger\MessageBusInterface;

class AdvertServedToUserToUserBatchConsumer implements BatchConsumerInterface
{
    private $commandBus;
    private $logger;

    public function __construct(MessageBusInterface $commandBus, LoggerInterface $logger)
    {
        $this->commandBus = $commandBus;
        $this->logger = $logger;
    }

    /**
     * @param AMQPMessage[] $messages
     *
     * @return array|bool
     */
    public function batchExecute(array $messages)
    {
        try {
            $command = new RunAdvertServedToUserBatch($messages);
            $this->commandBus->dispatch($command);
            return true;
        } catch (\Throwable $e) {
            $this->logger->log('error', $e->getMessage(), ['exception' => $e]);
            return false;
        } 
    }
}
{% endhighlight %}

registered class as a service:

{% highlight yaml %}
#config/services.yaml
services:
    ...
    app.consumer.batch:
        class: App\Consumer\AdvertServedToUserBatchConsumer
        arguments: ["@messenger.bus.command", '@logger']
{% endhighlight %}

and use our consumer as callback for batch RabbitMQ

{% highlight yaml %}
#config/packages/old_sound_rabbit_mq.yaml
old_sound_rabbit_mq:
    connections:
        default:
            ...
        advert_served_to_user:
            url: '%env(BATCH_MESSAGES_TRANSPORT_DSN)%'
            lazy:     true
            connection_timeout: 3
            read_write_timeout: 3
            # requires php-amqplib v2.4.1+ and PHP5.4+
            keepalive: false
            # requires php-amqplib v2.4.1+
            heartbeat: 0
    consumers:
        ...
    	batch_consumers:
            advert_served_to_user:
                connection:       advert_served_to_user
                exchange_options: {name: 'advert-served-to-user', type: fanout }
                queue_options:    {name: 'advert-served-to-user'}
                callback:         'app.consumer.batch'
                qos_options:      {prefetch_size: 0, prefetch_count: 5000, global: false}
                timeout_wait:     5
                auto_setup_fabric: false
                idle_timeout_exit_code: -2
                keep_alive: false
                graceful_max_execution:
                    timeout: 60
{% endhighlight %}

After that we created a handler for our batch command that uses UNWIND query:

{% highlight php %}
<?php

namespace App\Message\CommandHandler\Advert;

use App\Message\Command\Advert\RunAdvertServedToUserBatch;
use App\Message\CommandHandler\BaseCommandHandler;
use App\Model\AdvertServedToUserM

class RunAdvertServedToUserBatchHandler extends BaseCommandHandler
{
    public function __invoke(RunAdvertServedtoUserBatch $command)
    {
        $batchArray = $this->getBatchArrayFromCommand($command);
        $this->em->createQuery('
                UNWIND $batch AS row
                MATCH (a:Advert), (u:User)
                WHERE a.uuid = row.advertUuid and u.uuid = row.userUuid
                MERGE (a)-[st:SERVED_TO]->(u)
                    ON CREATE SET st.serveCount = 1
                    ON MATCH SET st.serveCount = st.serveCount + 1')
            ->setParameter('batch', $batchArray)
            ->execute();
    }
    
    private function getBatchArrayFromCommand(RunAdvertServedtoUserBatch $command): array
    {
        $batchArray = \array_map(function (AdvertServedToUserModel $item) {
            return [
                'advertUuid' => $item->getAdvertUuid(),
                'userUuid' => $item->getUserUuid(),
            ];
        }, $command->getAdvertsServed());
        return $batchArray;
    }
}
{% endhighlight %}

After we’ve set all up and deployed the code the queue went empty in few days, the app continued running smoothly without clogging of the queue.