---
title: Symfony Lock and Messenger component
author: Tobias Nyholm
date: '2021-01-26 11:51:00 +0200'
header:
  image: images/posts/birds-in-line.jpg
  teaser: images/posts/birds-in-line.jpg
categories:
- Symfony
- Messenger
- Lock
---


## No two messages handled at the same time

{% highlight php %}
class LockMiddleware implements MiddlewareInterface
{
    private LockFactory $lockFactory;
    public function __construct(LockFactory $lockFactory)
    {
        $this->lockFactory = $lockFactory;
    }
    public function handle(Envelope $envelope, StackInterface $stack): Envelope
    {
        $lock = $this->getLock($envelope);
        $lock && $lock->acquire(true);
        try {
            return $stack->next()->handle($envelope, $stack);
        } finally {
            $lock && $lock->release();
        }
    }
    private function getLock(Envelope $envelope): ?LockInterface
    {
        $blockingCommands = [
            CreateJobAnnouncement::class,
            CreateOrganization::class,
            StorePerformanceData::class,
        ];
        $class = \get_class($envelope->getMessage());
        if ($envelope->last(ReceivedStamp::class) && \in_array($class, $blockingCommands, true)) {
            return $this->lockFactory->createLock($class);
        }
        return null;
    }
}

{% endhighlight %}

## The message holds the key

{% highlight php %}
interface LockableMessageInterface
{
    public function getLockKey(): string;
}

class LockMiddleware implements MiddlewareInterface
{
    private LockFactory $lockFactory;

    public function __construct(LockFactory $lockFactory)
    {
        $this->lockFactory = $lockFactory;
    }

    public function handle(Envelope $envelope, StackInterface $stack): Envelope
    {
        $message = $envelope->getMessage();
        if (!$message instanceof LockableMessageInterface) {
            return $stack->next()->handle($envelope, $stack);
        }

        $lock = $this->lockFactory->createLock($message->getLockKey(), 60);
        $lock->acquire(true);
        try {
            return $stack->next()->handle($envelope, $stack);
        } finally {
            $lock->release();
        }
    }
}

{% endhighlight %}

## Lock acquired by the handler

Released after DoctrineTransactionMiddleware

{% highlight php %}
class LockReleaseMiddleware implements MiddlewareInterface
{
    private static $locks = [];

    public function handle(Envelope $envelope, StackInterface $stack): Envelope
    {
        try {
            return $stack->next()->handle($envelope, $stack);
        } finally {
            $this->releaseLocks($envelope->getMessage());
        }
    }

    public static function releaseLater(object $message, LockInterface $lock)
    {
        $hash = \spl_object_hash($message);
        if (!isset(self::$locks[$hash])) {
            self::$locks[$hash] = [];
        }

        self::$locks[$hash][] = $lock;
    }

    private function releaseLocks(object $message)
    {
        $hash = \spl_object_hash($message);
        if (!isset(self::$locks[$hash])) {
            return;
        }

        foreach (self::$locks[$hash] as $lock) {
            $lock->release();
        }
    }
}

{% endhighlight %}

## Advanced topic

Serialize the lock and release it later.

TODO
