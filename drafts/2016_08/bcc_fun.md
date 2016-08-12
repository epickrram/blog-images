# Fun times with BCC

Ever since I read some initial blogs posts about the upcoming eBPF tracing functionality in the 4.x Linux kernel, 
I have been looking for an excuse to get to grips with this technology.

With a planned kernel upgrade in progress at [LMAX](https://lmax.com), I now have access to an interesting environment and 
workload in order to play around with [BCC](https://github.com/iovisor/bcc).


## BPF Compiler Collection

BCC is a collection of tools that allows the curious to express programs in C or Lua, and then load those programs
as optimised kernel modules, hooked in to the runtime via a number of different mechanisms.

At the time of writing, the [motivation](https://github.com/iovisor/bcc#motivation) subsection of the BCC README explains this best:

    End-to-end BPF workflow in a shared library
        A modified C language for BPF backends
        Integration with llvm-bpf backend for JIT
        Dynamic (un)loading of JITed programs
        Support for BPF kernel hooks: socket filters, tc classifiers, tc actions, and kprobes

## Tracking process off-cpu time

One of the more in-depth investigations I've been involved in lately is tracking down a throughput problem on one of our 
services. We have a service that will sometimes, during a test run in our performance environment, fail to keep up with
the message rate.

The service should be reading multicast traffic from the network as fast as possible. We dedicate a physical core
to the network processing thread, and make sure that the Java thread doing this work is interrupted as little as possible.

Now, given that we run with the SLAB allocator and want useful `vmstat` updates, our user-land processes will sometimes
get pre-empted by kernel threads. Looking in `/proc/sched_debug` shows the other processes that are potentially runnable
on our dedicated core:


    runnable tasks:
                task   PID         tree-key  switches  prio     wait-time             sum-exec        sum-sleep
    ----------------------------------------------------------------------------------------------------------
            cpuhp/35   334         0.943766        14   120         0.000000         4.365325         0.000000 1 0 /
        migration/35   336         0.000000        26     0         0.000000       185.499061         0.000000 1 0 /
        ksoftirqd/35   337        -5.236392         6   120         0.000000         0.010490         0.000000 1 0 /
        kworker/35:0   338 139152554.767297       456   120         0.000000         9.883127         0.000000 1 0 /
       kworker/35:0H   339        -5.241396        12   100         0.000000         0.045604         0.000000 1 0 /
        kworker/35:1   530 139227632.577336       902   120         0.000000        18.416365         0.000000 1 0 /
       kworker/35:1H  7190         6.745434         3   100         0.000000         0.013923         0.000000 1 0 /
    R           java 102825   1306133.326251       479   110         0.000000     38881.247855         0.000000 1 0 /autogroup-30
        kworker/35:2 102845 139065252.390586         2   120         0.000000         0.003843         0.000000 1 0 /

We know that our Java process will sometimes be kicked off the CPU by one of the `kworker` threads, so that it can do some house-keeping.
In order to find out if there is a correlation between network traffic build-up and the java process being off-cpu, we have traditionally
used `ftrace` and the built-in tracepoint `sched:sched_switch`.

## Determining off-cpu time with `ftrace`

The kernel scheduler [emits an event](http://lxr.free-electrons.com/source/kernel/sched/core.c#L3346) 
to the tracing framework every time that a context switch occurs. The event arguments report the process that is being 
switched out, and the process that is about to be scheduled for execution.

The output from `ftrace` will show these data, along with a microsecond timestamp:

              <idle>-0     [020] 6907585.873080: sched_switch:         0:120:R ==> 33233:110: java
               <...>-33233 [020] 6907585.873083: sched_switch:         33233:110:S ==> 0:120: swapper/20
              <idle>-0     [020] 6907585.873234: sched_switch:         0:120:R ==> 33233:110: java

The excerpt above tells us that on CPU 20, the idle process (pid 0) was runnable (R) and was switched out in favour of a Java process (pid 33233).
Three microseconds later, the Java process entered the sleep state (S), and was switched out in favour of the idle process. After another 150 microseconds 
or so, the Java process was again ready to run, so the idle process was switched out to make way.


Using the output from `ftrace`, we could build a simple script to track the timestamps when each process was switched *off* the CPU, then subtract
that timestamp from the next time that the process gets scheduled back onto a CPU. Aggregating over a suitable period would then give us the 
maximum time off-cpu in microseconds for each process in the trace. This information would be useful in looking for a correlation between network backlog
and off-cpu time.

## Determining off-cpu time with `BCC`

So that's the old way, and it has its drawbacks. `ftrace` can be quite heavy-weight, and we have found that running it can cause significant overhead in
our performance-test environment when running under heavy load. Also, we are only really interested in the single dedicated network-processing core. While
it is possible to supply `ftrace` with a cpumask to restrict this, the interface is a little clunky, and doesn't seem to work very will with the `trace-cmd` front-end 
to `ftrace`.

Ideally, we would like a lighter-weight mechanism for hooking in to context switches, and performing the aggregation in kernel-space rather than by 
post-processing the `ftrace` output.

Luckily, `BCC` allows us to do just that.


Looking at the [kernel source](http://lxr.free-electrons.com/source/kernel/sched/core.c#L3346), 
we can see that the `trace_sched_switch` event is emitted immediately before the call to `context_switch`:

        trace_sched_switch(preempt, prev, next);
        rq = context_switch(rq, prev, next, cookie); /* unlocks the rq */

At the end of the [`context_switch`](http://lxr.free-electrons.com/source/kernel/sched/core.c#L2819) function, `finish_task_switch` is called:

        /* Here we just switch the register state and the stack. */
        switch_to(prev, next, prev);
        barrier();
        
        return finish_task_switch(prev);
    }

The argument passed to `finish_task_switch` is a `task_struct` representing the process that is being switched out. This looks like a good spot to 
attach a trace, where we can track the time when a process is switched out of the CPU.

In order to do this using `BCC`, we use a [`kprobe`](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#1-kprobes) to hook in to the kernel
function `finish_task_switch`. Using this mechanism, we can attach a custom tracing function to the kernel's `finish_task_switch` function.

## `BCC` programs

The current method for interacting with the various probe types is via a Python-C bridge. Trace functions are written in C, then compiled and injected by a Python 
program, which can then interact with the tracing function via the kernel's tracing subsystem.

There is a lot of detail to cover in the various [examples](https://github.com/iovisor/bcc/tree/master/examples) 
and [user-guides](https://github.com/iovisor/bcc/blob/master/docs/tutorial_bcc_python_developer.md) 
available on the `BCC` site, so I will just focus on a walk-through of my use-case.

First off, the C function:

```
#include <linux/types.h>
#include <linux/sched.h>
#include <uapi/linux/ptrace.h>

struct proc_name_t {
    char comm[TASK_COMM_LEN];
};

BPF_TABLE("hash", pid_t, u64, last_time_on_cpu, 1024);
BPF_TABLE("hash", pid_t, u64, max_offcpu, 1024);
BPF_TABLE("hash", pid_t, struct proc_name_t, proc_name, 1024);

int trace_finish_task_switch(struct pt_regs *ctx, struct task_struct *prev) {
    // bail early if this is not the CPU we're interested in
    u32 target_cpu = %d;
    if(target_cpu != bpf_get_smp_processor_id())
    {
        return 0;
    }

    // get information about previous/next processes
    struct proc_name_t pname = {};
    pid_t next_pid = bpf_get_current_pid_tgid();
    pid_t prev_pid = prev->pid;
    bpf_get_current_comm(&pname.comm, sizeof(pname.comm));

    // store mapping of pid -> command for display
    proc_name.update(&next_pid, &pname);

    // lookup current values for incoming process
    u64 *last_time;
    u64 *current_max_offcpu;
    u64 current_time = bpf_ktime_get_ns();
    last_time = last_time_on_cpu.lookup(&next_pid);
    current_max_offcpu = max_offcpu.lookup(&next_pid);

    // update max offcpu time
    if(last_time != NULL) {
        u64 delta_nanos = current_time - *last_time;
        if(current_max_offcpu == NULL) {
            max_offcpu.update(&next_pid, &delta_nanos);
        }
        else {
            if(delta_nanos > *current_max_offcpu) {
                max_offcpu.update(&next_pid, &delta_nanos);
            }
        }
    }

    // store outgoing process' time
    last_time_on_cpu.update(&prev_pid, &current_time);
    return 0;
};
```

Conceptually, all this program does is update a map of `pid` -> `timestamp`, every time a process is switched off-cpu.

Then, if a timestamp exists for the task currently being scheduled *on-cpu*, we calculate the delta in nanoseconds
(i.e. the time that the process was not on the cpu), and track the max value seen so far.

This is all executed in the kernel context, with very low overhead.

Next up, we have the Python program, which can read the data structure being populated by the trace function:

```
while 1:
    time.sleep(1)
    for k,v in b["max_offcpu"].iteritems():
        if v != 0:
            proc_name = b["proc_name"][k].comm
            print("%s max offcpu for %s is %dus" % (datetime.datetime.now(), proc_name, v.value/1000))
    b["max_offcpu"].clear()
```

Here we sleep for one second, then iterate over the items in the `max_offcpu` hash. This will contain an entry for every process
observed by the tracing function that has been switched off the CPU, and back in at least once.

After printing the off-cpu duration of each process, we then clear the data-structure so that it will be populated with
fresh data on the next iteration.

I'm still a little unclear on the raciness of this operation - I don't understand well enough whether there could be lost updates
between the reporting and the call to `clear()`.


Lastly, this all needs to be wired up in the Python script:

```
b = BPF(text=prog % (int(sys.argv[1])))
b.attach_kprobe(event="finish_task_switch", fn_name="trace_finish_task_switch")
```

Here we are requesting that the kernel function `finish_task_switch` be instrumented, and our custom function `trace_finish_task_switch` attached.

## Results

Using my laptop to test this script, I simulated our use-case by isolating a CPU (3) from the OS scheduler via the `isolcpus` kernel parameter.

Running my `offcpu.py` script with `BCC` installed generated the following output:

```
[root@localhost scheduler]# python ./offcpu.py 3
2016-08-12 16:13:42.163032 max offcpu for swapper/3 is 5us
2016-08-12 16:13:43.164501 max offcpu for swapper/3 is 18us
2016-08-12 16:13:44.166178 max offcpu for kworker/3:1 is 990961us
2016-08-12 16:13:44.166405 max offcpu for swapper/3 is 10us
2016-08-12 16:13:46.169315 max offcpu for watchdog/3 is 3999989us
2016-08-12 16:13:46.169413 max offcpu for swapper/3 is 6us
2016-08-12 16:13:50.175329 max offcpu for watchdog/3 is 4000011us
2016-08-12 16:13:50.175565 max offcpu for swapper/3 is 9us
```

This tells me that most of the time the `swapper` or idle process was scheduled on CPU 3 
(this makes sense - the OS scheduler is not allowed to schedule any userland programs on it because of `isolcpus`).

Every so often, a `watchdog` or `kworker` thread is schedued on, kicking of the `swapper` process for a few microseconds.


If I now simulate our workload by executing a user process on CPU 3 (just like our network-processing thread that is 
busy-spinning trying to receive network traffic), I should be able to see it being kicked off by the kernel threads.


The user-space task is not complicated:

```
cat hot_loop.sh

while [ 1 ]; do
    echo "" > /dev/null
done
```

and executed using `taskset` to make sure it executes on the correct CPU:

```
taskset -c 3 bash ./hot_loop.sh
```

Running the `BCC` script now generates this output:

```
[root@localhost scheduler]# python ./offcpu.py 3
2016-08-12 16:14:29.120617 max offcpu for bash is 4us
2016-08-12 16:14:30.121861 max offcpu for kworker/3:1 is 1000003us
2016-08-12 16:14:30.121925 max offcpu for bash is 5us
2016-08-12 16:14:31.123140 max offcpu for kworker/3:1 is 999986us
2016-08-12 16:14:31.123201 max offcpu for bash is 4us
2016-08-12 16:14:32.123675 max offcpu for kworker/3:1 is 999994us
2016-08-12 16:14:32.123747 max offcpu for bash is 5us
2016-08-12 16:14:33.124976 max offcpu for kworker/3:1 is 999994us
2016-08-12 16:14:33.125046 max offcpu for bash is 5us
2016-08-12 16:14:34.126287 max offcpu for kworker/3:1 is 999995us
2016-08-12 16:14:34.126366 max offcpu for bash is 5us
```

Great! The tracing function is able to show that my user-space program is being kicked off the CPU for around 5 microseconds.

I can now deploy this script to our performance environment and see whether the network-processing thread is being descheduled for 
long periods of time, and whether any of these periods correlate with increases in network buffer depth.

## Further reading

There are plenty of examples to work through in the `BCC` repository. Aside from `kprobes`, there are `uprobes` that allow
user-space code to be instrumented. Some programs also contain user-defined static probes (UDSTs), which are akin to the kernel's
static tracepoints.

Familiarity with the functions and tracepoints within the kernel is a definite bonus, as it helps us to understand what information is
available. My first port of call is usually running `perf` while capturing stack-traces to get an idea of what a program is actually
doing. After that, it's possible to start looking through the kernel source code looking for useful data to collect.

The scripts referenced in this post are [available on github](https://github.com/epickrram/tracing/tree/master/scheduler).

Be warned, they may not be the best example of using `BCC`, and are the result of an afternoon's hacking.

