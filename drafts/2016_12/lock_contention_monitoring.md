Recently, I've been working on some workshop material that will explore, 
among other things, the effect of lock contention on concurrent code.

Initially, I was trying to demonstrate the stop-the-world safepoint-pauses
that will be experienced, by even a garbage-free Java application,
when a previously __biased__ intrinsic lock is __unbiased__.

Let's unpack that statement quickly - 

   * safepoint-pause - the pause experienced by *all* application threads when the JVM needs to perform some administrative action. *At least* the length of time it takes to bring *all* application threads to a safepoint.
   * garbage-free - a program that generates no garbage (in steady state), in order to eliminate GC pauses.
   * intrinsic lock - a lock declared/used with the *synchronized* keyword (as opposed to a j.u.c.Lock instance).
   * biased - if the JVM detects that a lock is only accessed by a single thread, it will *bias* the lock towards that thread, meaning that subsequent lock/unlock operations can be highly optimised (see [Biased Locking in Hotspot](https://blogs.oracle.com/dave/entry/biased_locking_in_hotspot)).
   * unbiased - when the JVM detects that a *different* thread wishes to acquire an already-biased lock, it must *unbias* it. This requires that the JVM brings all threads to a halt in order to make the change to the lock state.

I had thought that it might be possible to observe a worst-case scenario
where a lock was repeatedly biased between two different threads, requiring 
safepoint pauses each time the bias-holder changed.

In practice, the JVM will detect if there is lock contention and mark the
lock as contended. Future bias operations will not be made on a lock that
is marked so. So, the safepoint overhead incurred when a lock is unbiased
will not occur repeatedly for a given lock.

## Thread contention monitoring

While looking for a way to observe the locking overhead, I stumbled across the
[`ThreadMXBean`](http://download.java.net/java/jigsaw/docs/api/java/lang/management/ThreadMXBean.html)
interface. Via the [`ThreadInfo`](http://download.java.net/java/jigsaw/docs/api/java/lang/management/ThreadInfo.html)
class, we can access 
[useful information](http://download.java.net/java/jigsaw/docs/api/java/lang/management/ThreadInfo.html#SyncStats) 
about the contention effects experienced for a given `Thread`.

In order to demonstrate the utility of this information, we will use a 
naive example that will magnify the effects of lock contention.

## The Java Memory Model, lock-free algorithms, and the single-writer principle

One of the features of the Java Memory Model (JMM) is the definition of a *happens-before*
relation, which describes an operation that guarantees all memory operations that happened
before it, will be visible to operations that happen after it.

We use this guarantee in concurrent code to reason about what changes to memory are
visible to a given thread at a given time.

There are a number of operations that provide *happens-before* edges in the JVM.
Reading through [Java Concurrency in Practice](http://jcip.net/), and looking at the summary 
[in the Javadoc](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/package-summary.html#MemoryVisibility)
gives us this (non-exhaustive) list:

   * monitor enter/exit (i.e. an intrinsic lock)
   * volatile write/read
   * lock()/unlock() on a java.util.concurrent.Lock

High-Performance Inter-Thread messaging libraries (yes, I'm looking at you, 
[Disruptor](https://github.com/LMAX-Exchange/disruptor)) exploit these properties to allow 
messages (or, more properly memory mutations) to pass safely between threads. Through proper placement
of the *happens-before* relation, a user of the library can make mutations to memory within one
thread, and be sure that once publication to the `Disruptor` has taken place, all mutations
will be visible to the reading thread.

Now, this would be reasonably fast if we were to use intrinsic locks or `Lock` objects, but that
would still involve the OS kernel in arbitrating between the threads contending on the lock.
To use the minimally-expensive *happens-before* relation, we can just use a volatile write/read 
pair (this is actually what the very early versions of the Disruptor used for *happens-before*).

Since a volatile write is a pure memory operation, it will be cheaper than using the 
software constructs that are locks.

At the time of writing, the Disruptor utilises another trick to optimise this process 
still further. For a single writing thread (i.e. memory mutator), we can exploit [program
order](http://stackoverflow.com/questions/15654276/interpretation-of-program-order-rule-in-java-concurrency) 
and the semantics of `AtomicLong.lazySet()` to further improve performance of inter-thread
message passing. Detailed write-ups of this technique are 
[available to the curious](http://psy-lob-saw.blogspot.co.uk/2012/12/atomiclazyset-is-performance-win-for.html).

This was always a slightly cheeky optimisation based on the fact that the `lazySet()` methods
have the same effect as a store fence, preventing reordering of writes before the invocation.
This effect was entirely implicit, and since JDK8, there is an explicit method on the hallowed
[Unsafe](http://hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/8b04ee324a1a/src/share/classes/sun/misc/Unsafe.java#l1123)
class called `storeFence()`, which will achieve the same result.

## A naive, flawed, and possibly dangerous experiment

For this post, I am comparing different mechanisms for providing a minimal *happens-before* edge
between two threads. Our requirement is that we can do some work on the writer thread *W*, publish a counter
value, read the counter value on the reader thread *R*, and assume that writes made in *W* are visible
to *R*.

Now, this sort of thing is not recommended and is included here merely for the purposes of 
demonstration and discussion. Be warned, if you use these techniques, you may attract the attention of those in the know:
 
![lazySet](https://github.com/epickrram/blog-images/raw/master/2016_12/lazyset.png)

With that caveat, let's look at our experiment. On our writer thread *W*, we are going to do the following:

   1. Perform a memory write to a particular location
   2. Publish a counter value

on the reader thread *R*:

   1. Read the counter value
   2. Assert that the memory write in *W*.1 is visible to *R*

We can loosely determine the throughput of each available *happens-before* mechanism by comparing
the maximum value of the counter value when the test is executed for a set amount of time.

### The Writer

Our writer thread will simply update a counter, set some unprotected variable, then publish the counter:

```
    public void run()
    {
        while(!Thread.currentThread().isInterrupted())
        {
            final long updated = value++;
            exchanger.unprotectedSetValueForCounter(updated);
            exchanger.updateCounter(updated);
        }
    }
```

### The Reader

The reader is a little more complicated. First we read the latest-published counter value,
then read the unprotected variable. After a quick (non-exhaustive, open to abuse) check to
make sure the counter update __probably__ hasn't been reordered with the variable write,
we update a couple of metrics (more on these later).

```
    public void run()
    {
        while(!Thread.currentThread().isInterrupted())
        {
            final long current = exchanger.getCounter();

            final long state = exchanger.unprotectedGetValueForCounter(current);
            /*
            If this condition does not hold true, then the write before the
            counter-publish in writer thread has been re-ordered
             */
            if(state < current)
            {
                System.err.println(
                        String.format("Previous write for counter %d was not visible to reader!%n" +
                                "Expected %d > %d%n", current, state, current));
                return;
            }

            if(current != value)
            {
                distinctUpdateCount++;
            }
            else
            {
                noUpdateCount++;
            }
            value = current;
        }
    }
```

### Variables

We will test a number of different mechanisms for providing memory ordering:

#### Intrinsic lock

```
    private long value;

    @Override
    public void updateCounter(final long v)
    {
        synchronized (lock)
        {
            value = v;
        }
    }

    @Override
    public long getCounter()
    {
        synchronized (lock)
        {
            return value;
        }
    }
```

#### j.u.c.Lock

```
    private long value;

    @Override
    public void updateCounter(final long v)
    {
        lock.lock();
        try
        {
            value = v;
        }
        finally
        {
            lock.unlock();
        }
    }

    @Override
    public long getCounter()
    {
        lock.lock();
        try
        {
            return value;
        }
        finally
        {
            lock.unlock();
        }
    }
```

#### Atomic

```
    private final AtomicLong value = new AtomicLong();

    @Override
    public void updateCounter(final long v)
    {
        value.set(v);
    }

    @Override
    public long getCounter()
    {
        return value.get();
    }
```

#### volatile

```
    private volatile long value;

    @Override
    public void updateCounter(final long v)
    {
        value = v;
    }

    @Override
    public long getCounter()
    {
        return value;
    }
```

#### lazySet

```
    private final AtomicLong value = new AtomicLong();

    @Override
    public void updateCounter(final long v)
    {
        value.lazySet(v);
    }

    @Override
    public long getCounter()
    {
        return value.get();
    }
```

#### storeFence

```
    private long value;

    @Override
    public void updateCounter(final long v)
    {
        value = v;
        THE_UNSAFE.storeFence();
    }

    @Override
    public long getCounter()
    {
        return value;
    }
```

Each run of the experiment is executed for the same amount of wall-time (15 sec) and we will
compare how many updates to the counter were made during that time. We can expect a higher 
number of updates when there is little or no *contention* on the counter variable.
This essentially demonstrates the __throughput__ of the writer.

### Results

Before looking at the test results, let's explore the other metrics recorded in the Reader thread:

   * updates: number of reads of the counter value where it had changed from its previous value (i.e. Writer has made progress since last read)
   * noUpdates: number of reads of the counter value where it __had not__ changed from its previous value (i.e. Writer has not made progress)
   
So, a high proportion of *noUpdates* implies that the reader was able to observe the counter value between updates
made by the Writer thread (read-biased), a lower proportion implies that the writer was making more
progress between subsequent read operations.


```
j.u.c.Lock   thrpt:     38868321, reads:     49733367, updates:     17779431 (35%), noUpdates:     31953936 (64%)
sync         thrpt:     59240576, reads:    210528095, updates:      1142108 ( 0%), noUpdates:    209385987 (99%)
volatile     thrpt:    260426497, reads:    218205980, updates:    201582574 (92%), noUpdates:     16623406 ( 7%)
atomic       thrpt:    351818790, reads:    223158685, updates:    204194198 (91%), noUpdates:     18964487 ( 8%)
lazySet      thrpt:    474050815, reads:    271181166, updates:    266660114 (98%), noUpdates:      4521052 ( 1%)
sfence       thrpt:    549367363, reads:    130982471, updates:    127470263 (97%), noUpdates:      3512208 ( 2%)
```

