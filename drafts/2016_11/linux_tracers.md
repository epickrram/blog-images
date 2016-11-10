# Linux Tracer notes

Here we will take a tour through the code and configuration involved in using Linux tracing tools such as `ftrace`, `perf`, `eBPF` and `system-tap`.

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
