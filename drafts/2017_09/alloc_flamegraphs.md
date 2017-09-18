# Heap allocation Flamegraphs

The most recent addition to the [grav](https://github.com/epickrram/grav) collection
of performance visualisation tools is a utility for tracking heap allocations on the JVM.

This is another Flamegraph-based visualisation that can be used to determine hotspots
of garbage creation in a running program.

## Usage and mode of operation

Detailed instructions on installation and pre-requisites can be found in the 
[grav repository](https://github.com/epickrram/grav#jvm-heap-allocation-flamegraphs)
on github.

Heap allocation flamegraphs use the built-in user statically-defined tracepoints (USDTs), 
which have been added to recent versions of OpenJDK and Oracle JDK.

To enable the probes, the following command-line flags are required:

`-XX:+DTraceAllocProbes -XX:+ExtendedDTraceProbes`

Once the JVM is running, the [heap-alloc-flames script](https://github.com/epickrram/grav/blob/master/src/heap/heap_profile.py) can be used to generate a heap allocation flamegraph:

```
$ ./bin/heap-alloc-flames -p $PID -e "java/lang/String" -d 10
...
Wrote allocation-flamegraph-$PID.svg
```

BE WARNED: this is a fairly heavyweight profiling method - on each allocation, 
the entire stack-frame is walked and hashed in order to increment a counter.
The JVM will also use a slow-path for allocations when extended DTrace probes
are enabled.

It is possible to limit the profiling to record every `N` samples with the '`-s`'
parameter ([see the documentation for more info](https://github.com/epickrram/grav#jvm-heap-allocation-flamegraphs)).

For a more lightweight method of heap-profiling, see the excellent
[async-profiler](https://github.com/jvm-profiling-tools/async-profiler#heap-profiling),
which uses a callback on the first TLAB allocation to perform its sampling.

When developing low-garbage or garbage-free applications, it is useful to be able to 
instrument _every_ allocation, at least within a performance-test environment.
This tool could even be used to regression-test allocation rates for low-latency applications,
to ensure that changes to the codebase are not increasing allocations.

### eBPF + USDT

The allocation profiler works by attaching a 
[uprobe](https://www.kernel.org/doc/Documentation/trace/uprobetracer.txt) to the 
[dtrace_object_alloc](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/gc_interface/collectedHeap.inline.hpp#l80)
function provided by the JVM.

When the profiler is running, we can confirm that the tracepoint is in place by looking
at `/sys/kernel/debug/tracing/uprobe_events`:

```
$ cat  /sys/kernel/debug/tracing/uprobe_events
p:uprobes/
p__usr_lib_jvm_java_8_openjdk_amd64_jre_lib_amd64_server_libjvm_so_0x967fdf_bcc_7043 
/usr/lib/jvm/java-8-openjdk-amd64/jre/lib/amd64/server/libjvm.so:0x0000000000967fdf
p:uprobes/
p__usr_lib_jvm_java_8_openjdk_amd64_jre_lib_amd64_server_libjvm_so_0x96806f_bcc_7043 
/usr/lib/jvm/java-8-openjdk-amd64/jre/lib/amd64/server/libjvm.so:0x000000000096806f
```

Given that we know the type signature of the `dtrace_object_alloc` method, it is a simple
matter to extract the class-name of the object that has just been allocated.

As the profiler is running, it is recording a count against a compound key of _java class-name_ and 
_stack-trace id_. At the end of the sampling period, the count is used to 'inflate' the occurrences
of a given stack-trace, and these stack-traces are then piped through the usual flamegraph
machinery.

## Controlling output

![Allocation flamegraph](https://github.com/epickrram/blog-images/raw/master/2017_09/heap_alloc_flamegraph.png)

Stack frames can be included or excluded from the generated Flamegraph by using regular 
expressions that are passed to the python program.

For example, to exclude all allocations of `java.lang.String` and `java.lang.Object[]`,
add the following parameters:

```
-e "java/lang/String" "java.lang.Object\[\]"
```

To include only allocations of `java.util.ArrayList`, add the following:

```
-i "java/util/ArrayList"
```

## Inspiration & Thanks

Thanks to Amir Langer for collaborating on this profiler.

For more information on USDT probes in the JVM, see
Sasha's [blog](http://blogs.microsoft.co.il/sasha/2016/03/31/probing-the-jvm-with-bpfbcc/)
[posts](http://blogs.microsoft.co.il/sasha/2017/07/07/profiling-the-jvm-on-linux-a-hybrid-approach/).
