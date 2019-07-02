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
using Symfony with a Neo4j database. Communication between the main application and the recommendation engine is done via messages over RabbitMQ. 
We are sending interesting events that happened on our website to the recommendation engine and then pulling some recommendations from it.

## The problem

Over time we noticed something – our message queue got quite clogged, over 2 millions of messages were waiting to be written in the database! So we dug up a bit and found out 
that Neo4j is not quite optimized for one by one writing, it is recalculating relations after each insert, and we had large amounts of messages that just kept coming 
in. So we needed to fix that, in order to have up to date recommendations for our system.

## The investigation

After reading some articles, and Neo4j official docs, we found several guidelines:
- don’t use the ORM
- use [UNWIND](https://neo4j.com/docs/cypher-manual/current/clauses/unwind/)   

We decided to check how that really impacts the write queries to the database so we made a little experiment. We ran the insert of 5000 entries with ORM, without ORM and using UNWIND:

{% highlight console %}
Started writing 5000 entries with ORM.
 5000/5000 [============================] 100%
Finished in 2 minutes and 21 seconds.
Started writing 5000 entries with CYPHER.
 5000/5000 [============================] 100%
Finished in 0 minutes and 43 seconds.
Started writing 5000 entries with CYPHER using UNWIND.
Finished in 0 minutes and 4 seconds.

{% endhighlight %}

The write with ORM took 2 minutes and 21 seconds, using CYPHER 43 seconds, using UNWIND only 4 seconds!

## The solution

When checking the messages stuck on the queue we noticed one message (`AdvertServedToUser`) was 80% of the total messages. So we decided to move it out of ORM and use UNWIND. 
We created another queue for our batch messages, and when consuming them from the original queue – we just re-queued them to another. We were using [RabbitMqBundle](https://github.com/php-amqplib/RabbitMqBundle) for consuming messages from the queue and Symfony Messenger inside application.

We had to change routing for the specific message in order to put it on the different queue:

{% highlight yaml %}
#config/packages/messenger.yaml
framework:
    messenger:
        transports:
            batch_messages_amqp: '%env(BATCH_MESSAGES_TRANSPORT_DSN)%/advert-served-to-user
        # ...
        routing:
            App\Message\Command\AdvertServedToUser: batch_messages_amqp

{% endhighlight %}

The next step was to get messages from the new queue in a batch of 50 or more (we ended up getting 5000 of them in one batch). In order to do that we had to create consumer class:

{% highlight php %}
<?php

declare(strict_types=1);

namespace App\Consumer;

use App\Message\Command\Advert\AdvertServedToUser;
use OldSound\RabbitMqBundle\RabbitMq\BatchConsumerInterface;
use PhpAmqpLib\Message\AMQPMessage;
use Psr\Log\LoggerInterface;
use GraphAware\Neo4j\OGM\EntityManager;
use Symfony\Component\Messenger\Envelope;

class AdvertServedToUserToUserBatchConsumer implements BatchConsumerInterface
{
    private $em;
    private $logger;

    public function __construct(EntityManager $em, LoggerInterface $logger)
    {
        $this->logger = $logger;
        $this->em = $em;
    }

    /**
     * @param AMQPMessage[] $messages
     */
    public function batchExecute(array $messages)
    {
        try {
            $this->runBatch($messages);
        } catch (\Exception $e) {
            $this->logger->log('error', $e->getMessage(), ['exception' => $e]);

            return false;
        }

        return true;
    }
    
    private function runBatch(array $messages): void
    {
        $batchArray = $this->getBatchArray($messages);
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

    private function getBatchArray(array $messages): array
    {
        $collection = [];
        foreach ($messages as $message) {
            /** @var Envelope $envelope */
            $envelope = \unserialize($message->getBody());
            /** @var AdvertServedToUser $originalMessage */
            $originalMessage = $envelope->getMessage();
            $collection[] = [
                'advertUuid' => $originalMessage->getAdvertId(),
                'userUuid' => $originalMessage->getUserId(),
            ];
        }

        return $collection;
    }
}

{% endhighlight %}

and use our consumer as callback for batch RabbitMQ

{% highlight yaml %}
#config/packages/old_sound_rabbit_mq.yaml
old_sound_rabbit_mq:
    connections:
        default:
            # ...
        advert_served_to_user:
            url: '%env(BATCH_MESSAGES_TRANSPORT_DSN)%'
    consumers:
        # ...
    	batch_consumers:
            advert_served_to_user:
                connection:       advert_served_to_user
                exchange_options: {name: 'advert-served-to-user', type: direct }
                queue_options:    {name: 'advert-served-to-user'}
                callback:         'App\Consumer\AdvertServedToUserToUserBatchConsumer'
                qos_options:      {prefetch_size: 0, prefetch_count: 5000, global: false}
                timeout_wait:     5
{% endhighlight %}

After we’ve set all up and deployed the code the queue went empty in few days, the app continued running smoothly without clogging of the queue.