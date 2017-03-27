## Flamegraphs with thread names

After watching a great talk by Nitsan Wakart at this year's QCon London,
I started playing around with flamegraphs a little more.

To get the gist of Nitsan's talk, you can read his blog post
[Java Flame Graphs Introduction: Fire For Everyone!](http://psy-lob-saw.blogspot.co.uk/2017/02/flamegraphs-intro-fire-for-everyone.html).

The important thing to take away is that the collapsed stack files
that are processed by Brendan Gregg's 
[FlameGraph scripts](https://github.com/brendangregg/FlameGraph) are 
just text files, and so can be filtered, hacked, and modified using
your favourite methods for such activities (your author is a fan of 
`grep` and `awk`).

The examples in this post are generated using a fork of the excellent
[perf-map-agent](https://github.com/jvm-profiling-tools/perf-map-agent).
I've added a couple of scripts to make these examples easier.

To follow the examples, clone 
[this repository](https://github.com/epickrram/perf-map-agent).

## Thread breakdown

One feature that was demonstrated in Nitsan's talk was the ability
to collapse the stacks by Java thread. Usually, flamegraphs show all
samples aggregated into one view. With thread-level detail though,
we can start to see where different threads are spending their time.

This can be useful information when exploring the performance of 
system that are unfamiliar.

Let's see what difference this can make to an initial analysis.
For this example, we're going to look at a very simple microservice 
built on dropwizard. The service does virtually nothing except
echo a response, so we wouldn't expect the business logic to show up
in the trace. Here we are primarily interested in looking at what
the framework is doing during request processing.

We'll take an initial recording without a thread breakdown:

```
$ source ./etc/options-without-threads.sh

$ ./bin/perf-java-flames 7731
```

You can view the rendered svg file here: 
[flamegraph-aggregate-stacks.svg](https://gist.github.com/epickrram/e3956b86e2a3984b49986ce49a8cf7d0).

From the aggregated view we can see that most of the time is spent in
framework code (jetty, jersey, etc), a smaller proportion is in log-handling (logback),
and the rest is spent on garbage collection.

Making the same recording, but this time with the stacks assigned to their
respective threads, we see much more detail.

```
$ source ./etc/options-with-threads.sh

$ ./bin/perf-java-flames 7602
```

In [flamegraph-thread-stacks.svg](https://gist.github.com/epickrram/99f4dda169cbb2540300c90393f79d26)
we can immediately see that we have five threads doing most of the work
of handling the HTTP requests; they all have very similar profiles, so we
can reason that these are probably threads belonging to a generic
request-handler pool.

We can also see another thread that spends most of its time writing log messages to disk.
From this, we can reason that the logging framework has a single thread for 
draining log messages from a queue - something that was more difficult to see
in the aggregated view.

Now, we have made some assumptions in the statements above, so is there 
anything we can do to validate them?

## Annotating thread names

With the addition of a simple script to replace thread IDs with thread names, 
we can gain a better understanding of thread responsibilities within the application.

Let's capture another trace:

```
$ source ./etc/options-with-threads.sh

$ ./bin/perf-thread-flames 8513
```

Although [flamegraph-named-thread-stacks.svg](https://gist.github.com/epickrram/39adabacecf2cf57dff2868e2e4b555c)
looks identical when zoomed-out, it contains one more very useful piece of context.

Rolling the mouse over the base of the image shows that the five similar stacks are all from 
threads named "dw-XX", giving a little more evidence that these are dropwizard handler threads.

There are a couple of narrower stacks named "dw-XX-acceptor"; these are the threads that manage
incoming connections before they are handed off to the request processing thread pool.

Further along is a "metrics-csv-reporter" thread, whose responsibility is to write out 
performance metrics at regular intervals.

The logging framework thread is now more obvious when we can see its name is
"AsyncAppender-Worker-async-console-appender".




