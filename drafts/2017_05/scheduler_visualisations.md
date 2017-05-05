In my last post, I looked at annotating Flamegraphs with contextual information in order to
aid in filtering on an interesting subset of the data. One of the things that stuck with me
was the idea of using SVGs to render data generated on a server that runs in headless mode.

Traditionally, I have recorded profiles and traces on remote servers, then pulled the data
back to my workstation to filter, aggregate and plot. The scripts used to do this data munging
tend to be one-shot affairs, and I've probably lost many useful utilities over the years. I
really like the idea of building the rendering into the server-side script, as it forces us to 
think about how we want to interpret the data, and also gives us the ability to deploy and serve
such monitoring from a whole fleet of servers.

Partly to address this, and partly because experimentation is fun, I've been working on some
new visualisations in the same vein as Flamegraphs. 


These tools are available in the [grav](https://github.com/epickrram/grav) repository.


### Scheduler Profile

The scheduler profile tool can be used to determine whether your application's threads are
getting enough time of CPU, or whether there is resource contention at play.

In an ideal scenario, your application threads will only ever yield the CPU to one of the 
kernel's helper threads, such as `ksoftirqd`, and then only sparingly. Running the 
`scheduling-profile` script will record scheduler events in the kernel and will determine
the state of your application threads at the point they are pre-empted and replaced with
another process.

Threads will tend to be in one of three states when pre-empted by the scheduler:

   * Runnable - happily doing work, moved off the CPU due to scheduler quantum expiry (we want to minimise this)
   * Blocked on I/O - otherwise known as 'Uninterruptible sleep', still an interesting signal but expected for threads handling IO
   * Sleeping - voluntarily yielding the CPU, perhaps due to waiting on a lock

There are a [number of other states](http://lxr.free-electrons.com/source/include/linux/sched.h?v=4.4#L207), 
which will not be covered here.
