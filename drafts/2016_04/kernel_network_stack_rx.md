# Navigating the Linux kernel network stack: receive path

## Background

At [work](https://lmax.com) we practice continuous integration in terms of performance testing alongside different stages of functional testing.

In order to do this, we have a performance environment that fully replicates the hardware and software used in our production environments. 
This is necessary in order to be able to find the limits of our system in terms of throughput and latency, and means that we make sure that the environments are 
identical, right down to the network cables.

Since we like to be ahead of the curve, we are constantly trying to push the boundaries of our system to find out where it will fall over, and the nature of the failure mode.

This involves running production-like load against the system at a much higher rate than we have ever seen in production. 
We currently aim to be able to handle a constant throughput of 2-5 times
the maximum peak throughput ever observed in production. 
We believe that this will give us enough headroom to handle future capacity requirements.

Our Performance & Capacity team has a constant background task of:

1. increase load applied to the system until it breaks
2. find and fix the bottleneck

Using this process, we aim to ensure that we are able to handle spikes in demand, 
and increases in user numbers, while still achieving a consistent and low latency-profile.

## Identifying a bottleneck

During the latest iteration of the break/fix cycle, we identified one particular service as problematic. 
The service in question is one of those responsible for consuming the output of our matching engine, 
which can peak at 250,000 messages per second at our current performance-test load.

When we have a bottleneck in one of our services, it usually follows a familiar pattern of back-pressure from the
application processing thread, resulting in packets being dropped by the networking card.

In this case however, we could see from our monitoring that we were not suffering from any processing back-pressure, 
so it was necessary to delve a little deeper into the network packet receive path in order to understand the problem.

## Monitoring back-pressure

http://lxr.free-electrons.com/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L1654


