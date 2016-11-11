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

We don't have visibility of the data stored at the memory address `0xd6b590`, but we can make an educated guess.
Since we are running `perf` asking it to trace the 'major-faults' event, we can dig around in the source code to 
deduce what will happen in `perf_event_open()`.

The 'major-faults' event is mapped to the type 
[`PERF_COUNT_SW_PAGE_FAULTS_MAJ`](http://lxr.free-electrons.com/source/include/uapi/linux/perf_event.h?v=4.8#L109), 
which is passed on a page-fault to the `perf_sw_event()` function from 
[`fault.c`](http://lxr.free-electrons.com/source/arch/x86/mm/fault.c#L1394)
for each architecture.

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
call `perf_swevent_overflow()`, which will eventually call the event's handler_function and write an entry
into the per-cpu data file.

/home/pricem/dev/linux-4.8/tools/perf/tests/attr/README

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
