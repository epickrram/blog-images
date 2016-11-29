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

== Thread contention monitoring ==

While looking for a way to observe the locking overhead, I stumbled across the
[`ThreadMXBean`](http://download.java.net/java/jigsaw/docs/api/java/lang/management/ThreadMXBean.html)
interface. Via the [`ThreadInfo`](http://download.java.net/java/jigsaw/docs/api/java/lang/management/ThreadInfo.html)
class, we can access 
[useful information](http://download.java.net/java/jigsaw/docs/api/java/lang/management/ThreadInfo.html#SyncStats) 
about the contention effects experienced for a given `Thread`.

In order to demonstrate the utility of this information, we will use a 
naive example that will magnify the effects of lock contention.

== The Java Memory Model, lock-free algorithms, and the single-writer principle


