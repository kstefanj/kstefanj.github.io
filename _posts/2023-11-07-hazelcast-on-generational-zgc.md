---
layout: post
title:  "Hazelcast Jet on Generational ZGC"
date:   2023-11-07 03:00:01 +0200
tags: [Performance, Java, GC, ZGC]
---

A few years ago the developers of [Hazelcast Jet](https://jet-start.sh/) decided to try out the different GC alternatives available at the time. The result was this [blog-series](https://jet-start.sh/blog/2020/06/09/jdk-gc-benchmarks-part1) from 2020 and the results for ZGC looked very promising. Three years have passed since then and a lot more work has gone into ZGC. It became [production ready](https://openjdk.java.net/jeps/377) in JDK 15, [concurrent stack scanning](https://openjdk.org/jeps/376) was added in JDK 16 and now, in JDK 21, [generational support](https://openjdk.org/jeps/439) has been added making it even more suitable for low latency workloads. In this post we look at one of the Hazelcast Jet experiments and see how Generational ZGC performs.

## Generational ZGC

[JDK 21](https://inside.java/2023/09/19/the-arrival-of-java-21/) was just released and one of the big features in it was Generational ZGC. Having generational support allows ZGC to divide the heap into two generations (similar to most of the other OpenJDK garbage collectors):
* **the young generation** for newly allocated objects
* **the old generation** where object that have survived a number of GC cycles end up

This comes with the benefit of being able to collect the young generation more frequent, expecting most of the objects to be considered garbage (see [the weak generational hypothesis](https://docs.oracle.com/en/java/javase/21/gctuning/garbage-collector-implementation.html#GUID-71D796B3-CBAB-4D80-B5C3-2620E45F6E5D)). When most objects are garbage the cost of doing a collection goes down. This means that Generational ZGC can do more frequent GCs, **reclaiming more memory** still using **less resources** compared to legacy ZGC (the non-generational version).

Adding generations to ZGC was a very large feature that has been in development for over three years and if you want more detailed information about it I recommend reading [the JEP](https://openjdk.org/jeps/439) and watching [this video from JVMLS](https://inside.java/2023/08/31/generational-zgc-and-beyond/).

## The use-case

Hazelcast Jet is an "in-memory, distributed batch and stream processing engine"[^1]. In the mentioned blog-series there were a few different experiments with different setups and benchmarks. The one I re-created is from [part 3](https://jet-start.sh/blog/2020/06/23/jdk-gc-benchmarks-rematch) and focuses on higher percentile latencies over different levels of throughput.

I'm not running on the same kind of hardware as the old experiments, so comparing the new result to the old will not be possible. That said, the legacy ZGC results with JDK 21 show a lot of resemblance with the old results. I've also modified the benchmark slightly to fit our benchmarking environment as well as added some instrumentation to it. But none of that affects the results. The benchmark still runs on a **single node**, with a **fixed event rate** and the **throughput/allocation rate is varied** by using different sizes on the key-set.

## The results

To refresh our minds, back in the JDK 14 time-frame, when the original experiments were done, ZGC was looking great up until a certain point. There the allocation rate became too high and we started seeing longer and longer worse case latencies. Let's see how generations can change this picture.

![JDK 21]({{ site.baseurl }}/assets/posts/hazelcast/p9999-event-latency.png){: class="center_85" }

Very similar to the old runs, single generational ZGC performs very well under low load, but as the allocation pressure increases so does the worse case latencies. With **Generational ZGC** this is no longer the case. Even at a **high load** the **p99.99th latency is very low**.

The big reason for this improvement in latency is that application threads are more frequently available to handle user requests when using Generational ZGC. ZGC is a fully concurrent collector, which means that application threads sometimes have to help ZGC by relocating objects. This happens if ZGC's own threads haven’t finished relocating an object before the application needs to use it. (The work of relocation is invisible to the code of the application.) Legacy ZGC's threads have to process the entire heap, so application threads have to help out quite frequently, doing relocation work instead of "real" work. In contrast, most of the time, Generational ZGC's threads only have to process part of the heap, *the young generation*, so fewer objects need to be relocated and there is less chance of having application threads helping out with relocation.

If you want a more detailed explanation why Generational ZGC is getting so much better results compared to legacy ZGC, you can check out my [P99conf presentation](https://inside.java/2023/10/21/reducing-p99-latencies-with-genzgc/). In it I use the instrumented Hazelcast Jet benchmark to better understand where the differences lie and why it affects performance so much.

## Try it out and give us feedback

Hopefully this short post has made you eager to **try out Generational ZGC**. Just grab a [JDK 21 release](https://jdk.java.net/21/) of your choice and start your application using both:

> `-XX:+UseZGC -XX:+ZGenerational`

 The second flag is important right now to turn on the generational version of ZGC. The long-term goal is to only have the generational version of ZGC and to get there we like to **get user feedback**. If you have a use-case where Generational ZGC is not performing as well as legacy ZGC please let us know using [this OpenJDK mailing list](https://mail.openjdk.org/mailman/listinfo/hotspot-gc-use).


For other general news and insights from the Java team at Oracle make sure to check out [inside.java](https://inside.java/).

---

[^1]: From [https://github.com/hazelcast/hazelcast-jet](https://github.com/hazelcast/hazelcast-jet)
