---
title: Symfony Lock and Messenger component
author: Tobias Nyholm
date: '2021-02-14 11:51:00 +0200'
header:
  image: images/posts/locks.jpg
  teaser: images/posts/locks.jpg
  caption: "Photo credit: [**Marcos Mayer**](https://unsplash.com/@mmayyer)"
categories:
- Symfony
- Messenger
- Lock
---

The lock component have saved me so many times. It helps me with race conditions,
it makes my code simpler and my application more reliable. I'm using it to fix all
kinds of problems and I've noticed that I use a few different methods. This article will
try to explain these methods or "strategies". My hope is to make life simpler for everybody else to
implement Symfony Lock component with Messenger.

## No two messages handled at the same time

This strategy is the simplest one and is good if you want something that "works".
It is not the most efficient one and it required a predefined list of messages.

The idea is that two message of the same class cannot be handled at the same time.
One needs to wait for the other to finish.

The handler for ``CreateOrganization`` and ``StorePerformanceData`` will create an entity
if none exists. That is why I need a lock or the second message will complain that
an object with id X already exists.

{% highlight php %}
use Symfony\Component\Lock\LockInterface;
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Middleware\MiddlewareInterface;
use Symfony\Component\Messenger\Middleware\StackInterface;

class LockMiddleware implements MiddlewareInterface
{
    private LockFactory $lockFactory;

    public function __construct(LockFactory $lockFactory)
    {
        $this->lockFactory = $lockFactory;
    }

    public function handle(Envelope $envelope, StackInterface $stack): Envelope
    {
        // Get a lock or null
        $lock = $this->getLock($envelope);

        // If we got a lock, do a blocking wait
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
            CreateOrganization::class,
            StorePerformanceData::class,
        ];

        $class = \get_class($envelope->getMessage());
        if (\in_array($class, $blockingCommands, true)) {
            return $this->lockFactory->createLock($class);
        }
        return null;
    }
}

{% endhighlight %}

## The message holds the resource ID

This strategy is a bit more flexible. It introduces an interface that is added
to the messages. We can now handle two messages from the same class at the same time
as long as they have different keys. The key should contain some ID to avoid race
conditions.

A potential drawback of this is that a message cannot configure auto release timeout
or if it should be blocking wait or not.

{% highlight php %}
interface LockableMessageInterface
{
    public function getLockKey(): string;
}

class CreateOrganization implements LockableMessageInterface
{
    private $id;
    private $name;

    // ...

    public function getLockKey(): string
    {
        return 'create_organization_'.$this->id;
    }
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

## Configurable Locks

To build on top of the previous strategy, one can create a ``LockConfig`` class
that holds instructions for the ``LockMiddleware``. This strategy is more generic
and may be good for complex applications or as a reusable package.

{% highlight php %}
interface LockableMessageInterface
{
    /**
     * @return LockConfig[]
     */
    public function getLockConfigurations(): iterable;
}

class LockConfig
{
    private $resource;
    private $read;
    private $ttl;
    private $blocking;

    public function __construct(string $resource, bool $read = false, float $ttl = null, bool $blocking = false)
    {
        $this->resource = $resource;
        $this->read = $read;
        $this->ttl = $ttl;
        $this->blocking = $blocking;
    }

    public function getResource(): string
    {
        return $this->resource;
    }

    public function isRead(): bool
    {
        return $this->read;
    }

    public function getTtl(): ?float
    {
        return $this->ttl;
    }

    public function isBlocking(): bool
    {
        return $this->blocking;
    }

    public function withTtl(float $ttl): self
    {
        $self = clone $this;
        $self->ttl = $ttl;

        return $self;
    }
}

use Symfony\Component\Lock\Exception\ExceptionInterface;
use Symfony\Component\Lock\LockFactory;
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Exception\RecoverableMessageHandlingException;
use Symfony\Component\Messenger\Middleware\MiddlewareInterface;
use Symfony\Component\Messenger\Middleware\StackInterface;
use Symfony\Component\Messenger\Stamp\ConsumedByWorkerStamp;

class LockableMessageMiddleware implements MiddlewareInterface
{
    /** @var LockInterface[] */
    private $locks = [];
    private $lockFactory;

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

        $resources = $message->getLockConfigurations();

        try {
            $this->refreshLocks();
            foreach ($resources as $resource) {
                $this->acquireLock($resource);
            }

            try {
                return $stack->next()->handle($envelope, $stack);
            } finally {
                $this->releaseLocks();
            }
        } catch (ExceptionInterface $e) {
            throw new RecoverableMessageHandlingException('Failed to acquire lock', 0, $e);
        }
    }

    private function acquireLock(LockConfig $lockConfig): void
    {
        // ignore same key locked twice
        if (isset($this->locks[$lockConfig->getResource()])) {
            return;
        }

        $lock = $this->lockFactory->createLock($lockConfig->getResource(), $lockConfig->getTtl() ?? 300.0);
        if ($lockConfig->isRead() && $lock instanceof SharedLockInterface) {
            $res = $lock->acquireRead($lockConfig->isBlocking());
        } else {
            $res = $lock->acquire($lockConfig->isBlocking());
        }
        if (!$res) {
            $this->releaseLocks();
            throw new LockConflictedException(\sprintf('Failed to acquire the lock for "%s".', $lockConfig->getResource());
        }
        $this->locks[$lockConfig->getResource()] = $lock;
    }

    private function releaseLocks()
    {
        foreach ($this->locks as $lock) {
            try {
                $lock->release();
            } catch (\Throwable $e) {
            }
        }
        $this->locks = [];
    }

    private function refreshLocks()
    {
        foreach ($this->locks as $lock) {
            try {
                $lock->refresh();
            } catch (\Throwable $e) {
                $this->releaseLocks();

                throw $e;
            }
        }
    }
}
{% endhighlight %}

Note the use if ``RecoverableMessageHandlingException``. If we throw an instance
of ``RecoverableExceptionInterface``, then the message will not go to the failure
queue after 3 failed tries to handle the message.

## Lock acquired by the handler

Sometimes you want all the logic in the handler, but you also want to make sure
the lock is released after the database transaction is committed, ie released
after ``DoctrineTransactionMiddleware``.

This can be done by telling the middleware: "If you see that this message has been
processed, please release the lock".

{% highlight php %}
use Symfony\Component\Lock\LockInterface;
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Middleware\MiddlewareInterface;
use Symfony\Component\Messenger\Middleware\StackInterface;

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

class CreateOrganizationHandler implements MessageHandlerInterface
{
    private LockFactory $lockFactory;

    public function __construct(LockFactory $lockFactory)
    {
        $this->lockFactory = $lockFactory;
    }

    public function __invoke(CreateOrganization $command)
    {
        // ..
        $lock = $this->lockFactory->createLock('create_organization_'.$command->getId(), 10);
        $lock->acquire(true);
        try {
            $organization = // Find organization with id $command->getId()
        } catch (EntityNotFoundException $e) {
            $organization = new Organization($command->getId());
        }
        LockReleaseMiddleware::releaseLater($command, $lock);
    }
}
{% endhighlight %}


## Bonus topic

I discovered a hidden gem in the Lock component when I needed to acquire a
``Lock`` in a controller and then release it in a background process. You can serialize
the ``Key`` to the ``Lock``. Normally, the ``Key`` is automatically created in the
``LockFactory``, but you can create the ``Key`` yourself.

{% highlight php %}

class MyController
{
    // ...
    public function index($reportId)
    {
        $key = new Key('create-report-'.$reportId);
        $lock = $this->lockFactory->createLockFromKey($key, 1800, false);
        if (!$lock->acquire(false)) {
            // We could not acquire the lock.

            return new Response('The report is being generated, you cannot generate another report until the current one is finished');
        }

        $this->commandBus(new GenerateReport($reportId, serialize($key));
    }
}

class GenerateReportHandler implements MessageHandlerInterface
{
    public function __invoke(GenerateReport $command)
    {
        // ..

        $key = unserialize($command->getKey());
        if ($key instanceof Key) {
            $lock = $this->lockFactory->createLockFromKey($key);

            $lock->release();
        }
        // ..
    }
}
{% endhighlight %}

## Finally

Now when I forced you to read plenty of code examples, you may wonder why there isn't
a single solution that always work. Different applications may have different requirements
and doing complicated/flexible solution should only be valid on complex applications
or reusable code.

I'm currently using all of these strategies in different applications. One might
argue that applications should be consistent with each other within the same company.
And yes, that is true. I just wish I read a blog post like this before I started
implementing my solution.

Also a big shout out to [Jérémy Derussé](https://twitter.com/jderusse) who created
the Lock component and helped me with issues I had over the past year.
