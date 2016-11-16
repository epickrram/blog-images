# Linux Tracer notes

Here we will take a tour through the code and configuration involved in using Linux tracing tools such as `ftrace`, `perf`, `eBPF` and `system-tap`.

## Finding out what we don't know

My first attempt at understanding how a tracing tool such as `perf` works was to dig into the kernel documentation
and source and try to figure things out from the bottom up. 

While this would have been a worthy exercise, it struck me, as I waded through pages of C, that I could use another
tracing tool to help figure out what `perf` was doing: `strace`.

Running a simple `perf` command under `strace` will show the system calls being made by the `perf` program:

```
strace perf record -e major-faults -ag 2>&1 | head -n 10000 | less
```

Somewhere towards the bottom of this file can be seen the following:

```
perf_event_open(0xd6b590, -1, 0, -1, 0x8 /* PERF_FLAG_??? */) = 3
perf_event_open(0xd6b590, -1, 1, -1, 0x8 /* PERF_FLAG_??? */) = 4
perf_event_open(0xd6b590, -1, 2, -1, 0x8 /* PERF_FLAG_??? */) = 5
...
perf_event_open(0xd6b590, -1, 26, -1, 0x8 /* PERF_FLAG_??? */) = 29
perf_event_open(0xd6b590, -1, 27, -1, 0x8 /* PERF_FLAG_??? */) = 30
...
mmap(NULL, 528384, PROT_READ|PROT_WRITE, MAP_SHARED, 3, 0) = 0x7f8247356000
fcntl(3, F_SETFL, O_RDONLY|O_NONBLOCK)  = 0

```

I am running this on a 28-CPU machine, so it looks suspiciously like `perf` is calling `perf_event_open` for each CPU in the system.

Sure enough `man perf_event_open` yields the following snippet:

```
DESCRIPTION
       Given  a  list  of parameters, perf_event_open() returns a file descriptor, 
       for use in subsequent system calls (read(2), mmap(2), prctl(2),
       fcntl(2), etc.).

       A call to perf_event_open() creates a file descriptor that allows measuring 
       performance information.
```

The first argument to `perf_event_open()` is a struct of type 
[`perf_event_attr`](http://lxr.free-electrons.com/source/include/uapi/linux/perf_event.h?v=4.8#L283)
that describes what is to be observed.

We don't have visibility of the data stored at the memory address `0xd6b590` (from the `strace` output), 
but we can make an educated guess.
Since we are running `perf` asking it to trace the 'major-faults' event, we can dig around in the source code to 
deduce what will happen in `perf_event_open()`.


Using `perf` as an example starting point, we can now start taking a look at the different event types that 
are emitted from the kernel.

## Software events


The 'major-faults' event (used in the example `perf` command above) is mapped to the type 
[`PERF_COUNT_SW_PAGE_FAULTS_MAJ`](http://lxr.free-electrons.com/source/include/uapi/linux/perf_event.h?v=4.8#L109).

Having a look through the source code, we can see that this event type is used when a major fault is
encountered in the architecture-specific  
[`fault.c`](http://lxr.free-electrons.com/source/arch/x86/mm/fault.c#L1394).

The handler code invokes the `perf_sw_event()` function.

After a few levels of indirection and some error-checking, execution will end up in the
[`perf_swevent_event()`](http://lxr.free-electrons.com/source/kernel/events/core.c?v=4.8#L7140) function:


```
static void perf_swevent_event(struct perf_event *event, u64 nr,
                   struct perf_sample_data *data,
                   struct pt_regs *regs)
{
    struct hw_perf_event *hwc = &event->hw;

    local64_add(nr, &event->count);

    if (!regs)
        return;

    if (!is_sampling_event(event))
        return;

    if ((event->attr.sample_type & PERF_SAMPLE_PERIOD) && !event->attr.freq) {
        data->period = nr;
        return perf_swevent_overflow(event, 1, data, regs);
    } else
        data->period = event->hw.last_period;

    if (nr == 1 && hwc->sample_period == 1 && !event->attr.freq)
        return perf_swevent_overflow(event, 1, data, regs);

    if (local64_add_negative(nr, &hwc->period_left))
        return;

    perf_swevent_overflow(event, 0, data, regs);
}
```

Now, before falling down the rabbit-hole of function-chasing, it's fair to assume that at some point, we will 
call 
[`perf_swevent_overflow()`](http://lxr.free-electrons.com/source/kernel/events/core.c?v=4.8#L7114), 
which will eventually call the event's 
[`overflow_handler`](http://lxr.free-electrons.com/source/kernel/events/core.c?v=4.8#L7052)
function.

In the case of a software event, this resolves to 
[`__perf_output_begin()`](http://lxr.free-electrons.com/source/kernel/events/ring_buffer.c?v=4.8#L117), which writes an entry
to `perf`'s ring_buffer so that the data can be read by the user-space side of `perf`.


## Hardware events

`perf` also allows us to spy on hardware events (cache-misses, bus-cycles, etc). These are architecture specific and each event type
will generally map to a [`MSR`](https://en.wikipedia.org/wiki/Model-specific_register).

Making a wild guess, when we try to record a hardware event, `perf` will use timers or workqueues to sample the hardware register
at the desired frequency, and write the retrieved value into the ring-buffer for capture in user-space.


## Tracepoint events

The third type of instrumentation readily available is provided by in-kernel tracepoints.

One of the tracepoints reported by `perf list` is `sched:sched_switch`. The tracepoint is defined in the 
[scheduler events file](http://lxr.free-electrons.com/source/include/trace/events/sched.h?v=4.8#L124), and invoked
from the [`__schedule()`](http://lxr.free-electrons.com/source/kernel/sched/core.c?v=4.8#L3397) method of the scheduler:

```
         if (likely(prev != next)) {
                 rq->nr_switches++;
                 rq->curr = next;
                 ++*switch_count;
 
                 trace_sched_switch(preempt, prev, next);
                 rq = context_switch(rq, prev, next, cookie);
```

The content of the trace function will be expanded from a common macro, and will end up calling
[`perf_trace_run_bpf_submit`](http://lxr.free-electrons.com/source/kernel/events/core.c?v4.8#L7547) 
with the event-specific payload. 

At this point, the kernel will run any `BPF` program associated with the tracepoint:

```
	if (prog) {
		*(struct pt_regs **)raw_data = regs;
		if (!trace_call_bpf(prog, raw_data) || hlist_empty(head)) {
			perf_swevent_put_recursion_context(rctx);
			return;
		}
	}
```

finally forwarding the event to the `perf_tp_event()` function.

At this point, our previous reading comes in handy, since the tracepoint event is recorded as a software event:


```
void perf_tp_event(u16 event_type, u64 count, void *record, int entry_size,
		   struct pt_regs *regs, struct hlist_head *head, int rctx,
		   struct task_struct *task)
{
	struct perf_sample_data data;
	struct perf_event *event;

...

	hlist_for_each_entry_rcu(event, head, hlist_entry) {
		if (perf_tp_event_match(event, &data, regs))
			perf_swevent_event(event, count, &data, regs); // record as a software event
	}
```

As we have seen earlier, this event will now be written to the `perf` ring-buffer, to be captured in user-space.



## Kernel probes (kprobes)

When the statically-defined event types are not enough, and we want to capture arbitrary function calls in the kernel,
we can fall back to using `kprobes`.

For background, you can read the kernel's documentation on this feature: 
[kprobes.txt](http://lxr.free-electrons.com/source/Documentation/kprobes.txt?v=4.8).

In short, we can use the kprobe mechanism to snoop on certain kernel functions dynamically.

Let's take another look at the scheduler code, and find a function to observe:

```
/**
 * finish_task_switch - clean up after a task-switch
 * @prev: the thread we just switched away from.
 *
...
 */
static struct rq *finish_task_switch(struct task_struct *prev)
{
```

Now we need to make sure that we can hook into its call-site:


```

[root@localhost ~]# perf probe --funcs | grep finish_task_switch
finish_task_switch

```

This looks pretty likely, so we can now use `perf` to instrument the function and print some data:

```
[root@localhost ~]# perf probe --add finish_task_switch
Added new event:
  probe:finish_task_switch (on finish_task_switch)

```

Our new probe point is now listed as available by `perf list`, and we can intercept calls to it using the standard `perf` commands:

```
[root@localhost ~]# perf list | grep finish_task_switch
  probe:finish_task_switch                           [Tracepoint event]

[root@localhost ~]# perf record -e probe:finish_task_switch -g -a
^C[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 1.718 MB perf.data (2649 samples) ]

[root@localhost ~]# perf script | head
swapper     0 [005] 27509.843061: probe:finish_task_switch: (ffffffff810cf370)
	    7fff810d2371 finish_task_switch+0x80007f003001 ([kernel.kallsyms])
	    7fff817dfef5 schedule+0x80007f003035 ([kernel.kallsyms])
	    7fff817e01be schedule_preempt_disabled+0x80007f00300e ([kernel.kallsyms])
	    7fff810ee4d1 cpu_startup_entry+0x80007f0031a1 ([kernel.kallsyms])
	    7fff81054c61 start_secondary+0x80007f003161 ([kernel.kallsyms])


```

Using kprobe syntax, it is also possible to record variable and parameter values. More information on this is 
available in the `man` page for `perf-probe`.


## User probes (uprobes)

A similar mechanism to `kprobes` exist for instrumenting user code. They are called `uprobes`, and we can also use the 
`perf` program to interact with them. More background can be found in the 
[documentation](http://lxr.free-electrons.com/source/Documentation/trace/uprobetracer.txt?v=4.8).

Let's have a look at an example of instrumenting a user program; in this case the JVM.

We will use `perf probe` to look for probe points in `libjvm.so`:

```
[root@localhost ~]# perf probe --funcs -x /home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so | wc -l
33468

```

Over 30 thousand possible probe points exist; let's use one we know will be triggered:

```
[root@localhost ~]# perf probe --funcs -x /home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so |grep vm_exit
JavaThread::block_if_vm_exited
VM_Exit::set_vm_exited
VM_Exit::wait_if_vm_exited
vm_exit
vm_exit_during_initialization
vm_exit_during_initialization
vm_exit_during_initialization

[root@localhost ~]# perf probe -x /home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so --add vm_exit
Added new event:
  probe_libjvm:vm_exit (on vm_exit in /home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)

[root@localhost ~]# perf list | grep vm_exit
  probe_libjvm:vm_exit                               [Tracepoint event]

[root@localhost ~]# cat /sys/kernel/debug/tracing/uprobe_events 
p:probe_libjvm/vm_exit /home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so:0x0000000000688b10


```

Now we can record events using the familiar `perf` command:

```
[root@localhost ~]# perf record -e probe_libjvm:vm_exit -g -a

# - execute java in another terminal

^C[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 1.398 MB perf.data (3 samples) ]


[root@localhost ~]# perf script
java 14200 [004] 30011.362953: probe_libjvm:vm_exit: (7fdc531bcb10)
                  688b10 vm_exit+0xffff00475a998000 (/home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)
...
                  68bbe6 JavaCalls::call_helper+0xffff00475a999056 (/home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)
                  68c0f1 JavaCalls::call_virtual+0xffff00475a998321 (/home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)
                  68c597 JavaCalls::call_virtual+0xffff00475a998047 (/home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)
                  7232d0 thread_entry+0xffff00475a9980a0 (/home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)
                  a68f3f JavaThread::thread_main_inner+0xffff00475a9980df (/home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)
                  a6906c JavaThread::run+0xffff00475a99811c (/home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)
                  91cb88 java_start+0xffff00475a998108 (/home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)
                    75ca start_thread+0xffff004757a2c0ca (/usr/lib64/libpthread-2.23.so)


```

## Instrumentation mechanism

For more detail on how dynamic probing works, refer to the 
[documentation](http://lxr.free-electrons.com/source/Documentation/kprobes.txt?v=4.8#L56).

In summary, dynamic probes work by:

   * Modifying the code loaded into memory and overwriting the first byte of the instrumented instruction with a breakpoint instruction
   * Trapping the resulting interrupt when the instruction is next executed, and redirecting execution to the tracing handler function
   * Once the handler function is complete, execution is returned the original function

Since hitting the breakpoint instruction raises an interrupt, and interrupts are costly, the `kprobe` mechanism comes with an 
optimisation step.

Certain functions can be optimised if certain conditions hold true, and in this case, when the breakpoint instruction is first 
encountered, it will be replaced with a jump instruction, which will redirect execution straigh to the tracing handler function.


## Recap

Let's just go over what we've seen so far:

   * The linux kernel has 3 built-in event types: hardware events, software events and tracepoints
   * Tracing tools such as `perf` can intercept and record these events as they occur
   * We can dynamically instrument kernel (via kprobes) and user (via uprobes) code
   * Arbitrary event handler code can be attached to these tap points


kprobe/uprobe:
http://lxr.free-electrons.com/ident?i=perf_trace_buf_submit


/home/pricem/dev/linux-4.8/tools/perf/tests/attr/README
tools/include/uapi/linux/perf_event.h:

userspace:

--call-graph = dwarf


## Hardware debug registers

Tracers and debuggers rely on a feature of the CPU called a debug register.

These registers are used to specify a memory address that can trigger an interrupt on the CPU.
On some architectures, debug registers can be used to specify what kind of memory access will trigger an interrupt
(i.e. read/write/execute). The Wikipedia entry on 
[x86 debug registers](https://en.wikipedia.org/wiki/X86_debug_register#DR7_-_Debug_control) 
goes into some detail of how these options are encoded.

So in order to perform tracing of some function __f__, a tracing program simply needs to be able to resolve the function
to a particular memory address, and then instruct the CPU to fire an interrupt when that memory is accessed.

Static trace points

http://lxr.free-electrons.com/source/include/linux/tracepoint.h#L365


Software breakpoints

Software breakpoints are in fact set by replacing the instruction to be breakpointed with a breakpoint instruction. The breakpoint instruction is present in most CPUs, and usually as short as the shortest instruction, so only one byte on x86 (0xcc, INT 3). On Cortex-M CPUs, instructions are 2 or 4 bytes, so the breakpoint instruction is a 2 byte instruction.

http://www.nynaeve.net/?p=80




## Function address resolution



When we query a tracing tool such as `perf`, it will display a list of available function trace-points:

```
[root@localhost ~]# grep finish_task_switch /proc/kallsyms 
ffffffff810cf370 t finish_task_switch


```

https://onebitbug.me/2011/03/04/introducing-linux-kernel-symbols/

http://events.linuxfoundation.org/sites/events/files/lcjp13_takata.pdf

http://www.cs.columbia.edu/~nahum/w6998/papers/ols2009-breakpoint.pdf


user tracing: ptrace

Documentation/kprobes.txt


```
[root@localhost ~]# perf probe -F -x /home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so | grep -v "::"
[root@localhost ~]# perf probe  -x /home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so vm_exit
Added new event:
  probe_libjvm:vm_exit (on vm_exit in /home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)

You can now use it in all perf tools, such as:

	perf record -e probe_libjvm:vm_exit -aR sleep 1

[root@localhost ~]# perf record -a -e probe_libjvm:vm_exit


[root@localhost ~]# perf script
Failed to open /tmp/perf-9083.map, continuing without symbols
java  9105 [003] 16470.231000: probe_libjvm:vm_exit: (7fb310d53b10)
                  688b10 vm_exit (/home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)
            7fb2fc7ad994 [unknown] (/tmp/perf-9083.map)
            7fb2fc79fc4d [unknown] (/tmp/perf-9083.map)
            7fb2fc79fc4d [unknown] (/tmp/perf-9083.map)
            7fb2fc79fc4d [unknown] (/tmp/perf-9083.map)
            7fb2fc79fc92 [unknown] (/tmp/perf-9083.map)
            7fb2fc79fc92 [unknown] (/tmp/perf-9083.map)
            7fb2fc7987a7 [unknown] (/tmp/perf-9083.map)
                  68bbe6 JavaCalls::call_helper (/home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)
                  68c0f1 JavaCalls::call_virtual (/home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)
                  68c597 JavaCalls::call_virtual (/home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)
                  7232d0 thread_entry (/home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)
                  a68f3f JavaThread::thread_main_inner (/home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)
                  a6906c JavaThread::run (/home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)
                  91cb88 java_start (/home/mark/Programs/jdk1.8.0_65/jre/lib/amd64/server/libjvm.so)
                    75ca start_thread (/usr/lib64/libpthread-2.23.so)

  mem:<addr>[/len][:access]                          [Hardware breakpoint]
```
