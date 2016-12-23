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
   * atomic write/read
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

### Under the hood

As I started writing this blog post, I expected to include a section showing that for the cases where locks were
used (sync, j.u.c.Lock), most CPU time was spent in lock contention. For this I intended to use `perf` to record
stack traces from the writer thread and demonstrate that more time was spent in kernel lock arbitration code
than actually executing the java code. This would fit with the observation that the lock-free versions of the 
algorithm manage to do more work.

When starting to record the traces though, I was initially perplexed by how little the locking functions
showed up. It should have been obvious of course, but the reason for this is that the locking functions are
fast, and that while waiting to acquire the lock, the thread is unscheduled and hence will not show up
in execution traces! This brings to mind one of the most important things we need to consider when reasoning about
the performance of our code: 
[make sure your code is actually running!](http://www.slideshare.net/MarkPrice7/when-the-os-gets-in-the-way-61820480/8?src=clipshare)

Now that we've had this realisation, it's a little easier to see what's going on. Turning to our trusty tracing workhorse
`ftrace`, we can take a look at how much CPU-time our writer threads are actually getting in each case.

Recording kernel scheduler events will show when our writer thread is descheduled (because it must wait for a lock),
when it is woken up (because the lock is now free), and how long it has spent on-CPU in each cycle. My command-line 
for capturing these events is:

```
trace-cmd record -e "sched:sched_stat*" -e "sched:sched_wa*" -e "sched:sched_switch" -P $WRITER_PID -T
```

Let's look at some results:

#### j.u.c.Lock

This is a typical trace from the writer thread when using a `j.u.c.Lock` for synchronisation. In this case
the writer thread has pid 5104, the reader thread has pid 5103:

```
            java-5104  [005]  1240.077311: sched_stat_runtime:   comm=java pid=5104 runtime=30878 [ns] vruntime=298762874053 [ns]
            java-5104  [005]  1240.077315: kernel_stack:         <stack trace>
=> dequeue_entity (ffffffff930d8819)
=> dequeue_task_fair (ffffffff930d92ec)
=> deactivate_task (ffffffff930ca95c)
=> __schedule (ffffffff937fdb14)
=> schedule (ffffffff937fe1e5)
=> futex_wait_queue_me (ffffffff9312306f)
=> futex_wait (ffffffff93123f86)
=> do_futex (ffffffff93125acd)
=> SyS_futex (ffffffff93126365)
=> entry_SYSCALL_64_fastpath (ffffffff93802932)
            java-5104  [005]  1240.077316: sched_switch:         java:5104 [120] S ==> swapper/5:0 [120]
            java-5104  [005]  1240.077319: kernel_stack:         <stack trace>
=> schedule (ffffffff937fe1e5)
=> futex_wait_queue_me (ffffffff9312306f)
=> futex_wait (ffffffff93123f86)
=> do_futex (ffffffff93125acd)
=> SyS_futex (ffffffff93126365)
=> entry_SYSCALL_64_fastpath (ffffffff93802932)
            java-5103  [000]  1240.077321: sched_wakeup:         java:5104 [120] success=1 CPU:005
            java-5103  [000]  1240.077325: kernel_stack:         <stack trace>
=> ttwu_do_activate (ffffffff930cac3f)
=> try_to_wake_up (ffffffff930cb7b6)
=> wake_up_q (ffffffff930cba54)
=> do_futex (ffffffff931259f7)
=> SyS_futex (ffffffff93126365)
=> entry_SYSCALL_64_fastpath (ffffffff93802932)
          <idle>-0     [005]  1240.077326: sched_switch:         swapper/5:0 [120] R ==> java:5104 [120]
          <idle>-0     [005]  1240.077328: kernel_stack:         <stack trace>
=> schedule (ffffffff937fe1e5)
=> schedule_preempt_disabled (ffffffff937fe4ae)
=> cpu_startup_entry (ffffffff930e5636)
=> start_secondary (ffffffff9304a564)
            java-5104  [005]  1240.077329: sched_stat_runtime:   comm=java pid=5104 runtime=9356 [ns] vruntime=298762883409 [ns]
            java-5104  [005]  1240.077333: kernel_stack:         <stack trace>
=> dequeue_entity (ffffffff930d8819)
=> dequeue_task_fair (ffffffff930d92ec)
=> deactivate_task (ffffffff930ca95c)
=> __schedule (ffffffff937fdb14)
=> schedule (ffffffff937fe1e5)
=> futex_wait_queue_me (ffffffff9312306f)
=> futex_wait (ffffffff93123f86)
=> do_futex (ffffffff93125acd)
=> SyS_futex (ffffffff93126365)
=> entry_SYSCALL_64_fastpath (ffffffff93802932)
            java-5104  [005]  1240.077333: sched_switch:         java:5104 [120] S ==> swapper/5:0 [120]
```

Here are the important parts:

Our writer thread reports its runtime as it is descheduled:

```
            java-5104  [005]  1240.077311: sched_stat_runtime:   comm=java pid=5104 runtime=30878 [ns] vruntime=298762874053 [ns]
            java-5104  [005]  1240.077315: kernel_stack:         <stack trace>
=> dequeue_entity (ffffffff930d8819)
=> dequeue_task_fair (ffffffff930d92ec)
=> deactivate_task (ffffffff930ca95c)
```

Our thread is in the sleep (S) state, so is swapped out for the idle process (`swapper`). The kernel stack trace informs
us that this change in state is due to a `SyS_futex` (acquire lock) call resulting in a `wait`:

```
            java-5104  [005]  1240.077316: sched_switch:         java:5104 [120] S ==> swapper/5:0 [120]
            java-5104  [005]  1240.077319: kernel_stack:         <stack trace>
=> schedule (ffffffff937fe1e5)
=> futex_wait_queue_me (ffffffff9312306f)
=> futex_wait (ffffffff93123f86)
=> do_futex (ffffffff93125acd)
=> SyS_futex (ffffffff93126365)
```

Our writer thread is woken up by the _reader_ thread as it exits the critical section:

```
            java-5103  [000]  1240.077321: sched_wakeup:         java:5104 [120] success=1 CPU:005
            java-5103  [000]  1240.077325: kernel_stack:         <stack trace>
=> ttwu_do_activate (ffffffff930cac3f)
=> try_to_wake_up (ffffffff930cb7b6)
=> wake_up_q (ffffffff930cba54)
=> do_futex (ffffffff931259f7)
=> SyS_futex (ffffffff93126365)
=> entry_SYSCALL_64_fastpath (ffffffff93802932)
```

Given the start/end timestamps of the trace, and the sum of the `sched_stat_runtime` reports, we can make a 
stab at estimating what proportion of time the writer thread was executing. For this particular synchronisation
method, the writer thread spent ~5.7sec on-CPU during the ~9.5sec tracing period (60%). A little over half the time
would make sense, since we have two threads that are contending on the same single resource and in a fair system
they would each get an equal share.

#### intrinsic lock (synchronized)


This trace is broadly similar to the previous version:

```
            java-7625  [000]  4374.787978: sched_stat_runtime:   comm=java pid=7625 runtime=242858 [ns] vruntime=370008643155 [ns]
            java-7625  [000]  4374.787983: kernel_stack:         <stack trace>
=> dequeue_entity (ffffffff930d8819)
=> dequeue_task_fair (ffffffff930d92ec)
=> deactivate_task (ffffffff930ca95c)
=> __schedule (ffffffff937fdb14)
=> schedule (ffffffff937fe1e5)
=> futex_wait_queue_me (ffffffff9312306f)
=> futex_wait (ffffffff93123f86)
=> do_futex (ffffffff93125acd)
=> SyS_futex (ffffffff93126365)
=> entry_SYSCALL_64_fastpath (ffffffff93802932)
            java-7625  [000]  4374.787984: sched_switch:         java:7625 [120] S ==> swapper/0:0 [120]
            java-7625  [000]  4374.787989: kernel_stack:         <stack trace>
=> schedule (ffffffff937fe1e5)
=> futex_wait_queue_me (ffffffff9312306f)
=> futex_wait (ffffffff93123f86)
=> do_futex (ffffffff93125acd)
=> SyS_futex (ffffffff93126365)
=> entry_SYSCALL_64_fastpath (ffffffff93802932)
            java-7624  [004]  4374.787990: sched_wakeup:         java:7625 [120] success=1 CPU:000
            java-7624  [004]  4374.787994: kernel_stack:         <stack trace>
=> ttwu_do_activate (ffffffff930cac3f)
=> try_to_wake_up (ffffffff930cb7b6)
=> wake_up_q (ffffffff930cba54)
=> do_futex (ffffffff931259f7)
=> SyS_futex (ffffffff93126365)
=> entry_SYSCALL_64_fastpath (ffffffff93802932)
          <idle>-0     [000]  4374.787995: sched_switch:         swapper/0:0 [120] R ==> java:7625 [120]
          <idle>-0     [000]  4374.787998: kernel_stack:         <stack trace>
=> schedule (ffffffff937fe1e5)
=> schedule_preempt_disabled (ffffffff937fe4ae)
=> cpu_startup_entry (ffffffff930e5636)
=> rest_init (ffffffff937f52c7)
=> start_kernel (ffffffff93f7ffeb)
=> x86_64_start_reservations (ffffffff93f7f2ca)
=> x86_64_start_kernel (ffffffff93f7f419)
```


Writer thread descheduled due to waiting on the lock:

```
            java-7625  [000]  4374.787984: sched_switch:         java:7625 [120] S ==> swapper/0:0 [120]
            java-7625  [000]  4374.787989: kernel_stack:         <stack trace>
=> schedule (ffffffff937fe1e5)
=> futex_wait_queue_me (ffffffff9312306f)
=> futex_wait (ffffffff93123f86)
=> do_futex (ffffffff93125acd)
=> SyS_futex (ffffffff93126365)
```

Later woken up by the reader thread (pid 7624):

```
            java-7624  [004]  4374.787990: sched_wakeup:         java:7625 [120] success=1 CPU:000
            java-7624  [004]  4374.787994: kernel_stack:         <stack trace>
=> ttwu_do_activate (ffffffff930cac3f)
=> try_to_wake_up (ffffffff930cb7b6)
=> wake_up_q (ffffffff930cba54)
=> do_futex (ffffffff931259f7)
=> SyS_futex (ffffffff93126365)
```

How about runtime? Using the same approach as before, we can see that in ~10.2sec of trace time, the writer thread
was on-CPU for 6.9sec (~67%).

Looking back at the benchmark results, we can see that for this particular test the synchronisation method had
higher throughput that the j.u.c.Lock method, so it is perhaps expected that the writer thread gets more CPU time
as observed in the trace.


#### lazySet

In the case of the lazySet method, we are not expecting any lock contention at all, and this is born out in the trace:
 
```
            java-8624  [007]  5795.011955: sched_stat_runtime:   comm=java pid=8624 runtime=987897 [ns] vruntime=520053021294 [ns]
            java-8624  [007]  5795.011957: kernel_stack:         <stack trace>
=> task_tick_fair (ffffffff930dc292)
=> scheduler_tick (ffffffff930ccd09)
=> update_process_times (ffffffff93111237)
=> tick_sched_handle.isra.15 (ffffffff93120f75)
=> tick_sched_timer (ffffffff9312165d)
=> __hrtimer_run_queues (ffffffff93111c8e)
=> hrtimer_interrupt (ffffffff9311240a)
=> local_apic_timer_interrupt (ffffffff9304be58)
=> smp_apic_timer_interrupt (ffffffff938054ad)
=> apic_timer_interrupt (ffffffff9380466c)
```

The only reason our writer thread is reporting its runtime is because of the scheduler tick forcing the process
times to be updated.

If we look at a sample of the `sched_switch` events, we can see that the java thread is always in the runnable (R)
state (i.e. it is not voluntarily yielding the CPU):

```
            java-8624  [007]  5795.036973: sched_switch:         java:8624 [120] R ==> trace-cmd:8850 [120]
       trace-cmd-8850  [007]  5795.037012: sched_switch:         trace-cmd:8850 [120] S ==> java:8624 [120]
            java-8624  [007]  5795.188971: sched_switch:         java:8624 [120] R ==> trace-cmd:8850 [120]
       trace-cmd-8850  [007]  5795.189005: sched_switch:         trace-cmd:8850 [120] S ==> java:8624 [120]
            java-8624  [007]  5795.274966: sched_switch:         java:8624 [120] R ==> trace-cmd:8850 [120]
       trace-cmd-8850  [007]  5795.274986: sched_switch:         trace-cmd:8850 [120] S ==> java:8624 [120]
            java-8624  [007]  5795.514961: sched_switch:         java:8624 [120] R ==> trace-cmd:8850 [120]
```

How about CPU time? In the trace of ~11.5sec, the kernel reports the writer thread's total runtime as
11.4sec (~99%), giving an indication of why the write throughput is so much greater.


## Thread contention monitoring - remember that?

Before I got side-tracked by looking into the kernel primitives relied upon by the JVM's locking operations,
I mentioned monitoring lock contention. If you're on OpenJDK, this can be achieved 
[relatively simply](https://github.com/epickrram/lock-contention/blob/master/src/main/java/com/epickrram/sync/SynchronisationThroughput.java#L76):

```
final ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
threadMXBean.setThreadContentionMonitoringEnabled(true);
threadMXBean.setThreadCpuTimeEnabled(true);

LockSupport.parkNanos(unit.toNanos(sleepTime));

final ThreadInfo[] threadInfos = threadMXBean.getThreadInfo(ids);
for (ThreadInfo threadInfo : threadInfos)
{
    System.out.printf("%s blocked %dms (%d), waited %dms (%d)%n",
            threadInfo.getThreadId() == writerThreadId ? "writer" : "reader",
            threadInfo.getBlockedTime(), threadInfo.getBlockedCount(),
            threadInfo.getWaitedTime(), threadInfo.getWaitedCount());
}
```

Enabling this monitoring while running our tests shows the JVM's view of how long our reader & writer threads
spent waiting for locks:

```
Starting test with SyncExchanger for 15s
reader blocked 4089ms (867640), waited 0ms (0)
writer blocked 6989ms (871378), waited 0ms (0)

Starting test with LockExchanger for 15s
reader blocked 0ms (0), waited 280ms (2964277)
writer blocked 0ms (0), waited 696ms (6056816)

Starting test with LazyExchanger for 15s
reader blocked 0ms (0), waited 0ms (0)
writer blocked 0ms (0), waited 0ms (0)
```

Now this clearly doesn't tell the whole story - we know that the writer thread was off-CPU for about
the same proportion of time in both the j.u.c.Lock and intrinsic cases, but this is not shown the in
numbers reported by the JVM.

One thing that is interesting to note is that when instrinsic locks are used (synchronized), contention is
reported as _blockedTime_ and _blockedCount_; when j.u.c.Locks are used, contention is reported as 
_waitingTime_ and _waitedCount_. The reason for this is left as an exercise for the reader.

Whether these numbers can actually tell us anything meaningful is dependent on the application, but they
can certain give an insight into what threads in our systems are experiencing a lot of lock activity.