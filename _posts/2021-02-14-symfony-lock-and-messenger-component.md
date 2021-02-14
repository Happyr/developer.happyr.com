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
try to explain these methods. My hope is to make it simpler for everybody else to
implement Symfony lock component with Messenger.

## No two messages handled at the same time

This strategy is the simplest one and is good if you want something that "works".
It is not the most efficient one and it required a predefined list of messages.

The idea is that two message of the same class cannot be handled at the same time.
One needs to wait for the other to finish.

The handler ``CreateOrganization`` and ``StorePerformanceData`` will create an entity
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

## Lock acquired by the handler

Sometimes you want all the logic in the handler, but you also want to make sure
the lock is released after the database transaction is committed, ie after released
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

## The message holds the key

This second strategy is a bit more efficient. It introduce an interface that is added
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

## Configurable locks

To build on top of the previous strategy, one can create a ``LockConfig`` class
that holds instruction for the ``LockMiddleware``. This strategy is more generic
and may be good for complex applications or a third party package.


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
            throw new LockConflictedException(\Safe\sprintf('Failed to acquire the lock for "%s".', $lockConfig->getResource()));
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

## Advanced topic

Serialize the lock and release it later.

TODO
