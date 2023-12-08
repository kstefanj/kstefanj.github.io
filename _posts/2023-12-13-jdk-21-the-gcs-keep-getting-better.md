---
layout: post
title:  "JDK 21: The GCs keep getting better"
date:   2023-12-13 10:00:00 +0100
tags: [Performance, Java, GC]
---

A couple of years ago I wrote a post about the [GC progress between JDK 8 and JDK 17](https://kstefanj.github.io/2021/11/24/gc-progress-8-17.html) for our three main GCs. With the JDK 21 release this fall, we now have a new LTS release to benchmark and generate some GC performance charts for. [JDK 21](https://inside.java/2023/09/19/the-arrival-of-java-21/) and the other releases since JDK 17 have delivered a set of noteworthy features such as [Virtual Threads](https://openjdk.org/jeps/444), [Pattern Matching for switch](https://openjdk.org/jeps/441) and [Generational ZGC](https://openjdk.org/jeps/439). Let's see how it performs.

## Introduction

When comparing performance across JDK releases it's hard to say exactly what features gave a certain performance boost. But it is easy to see that the **Java Platform** as a whole has become **significantly more performant since JDK 8**. In this post I'm using [SPECjbb® 2015](https://www.spec.org/jbb2015/)[^spec] to show the performance gains. This is a well-known standard benchmark, which is good for showing changes done to the GCs. The main reason for this is that the benchmark provides two scores:
* **max-jOPS** - raw throughput
* **critical-jOPS** - throughput under latency constraints

Improvements to GC will improve both scores, but improvements in the latency-constrained score are much more tied to changes done to the GC. Basically, shorter pauses will give a better score. For the raw throughput score, improvements to the JIT and other parts of the Java platform also come into play.

I'm running the benchmark without much tuning, but I set a fixed **heap size of 16 GB** and I enable large pages and make sure they are paged in before running the benchmark. I want the results to **reflect out-of-the-box behavior**, but configuration like this is good to get **fair and consistent results**.

## Choosing your GC

Oracle supports 4 different GCs and they all serve different use-cases. In this post I don't include Serial GC because it is not suitable for the benchmark I use. **Serial GC's main focus is low overhead** and is mostly suitable for use-cases where the amount of memory and CPU resources is limited. The GCs that are featured in this comparison are:
* **G1** - the default collector since JDK 9, with a focus on balance between latency and throughput
* **Parallel** - throughput-oriented collector, that might suffer long worse-case latencies
* **Z** - the ultra-low latency alternative, fully concurrent with sub-millisecond pauses

Which GC to use depends on what's most important for your application. There are use-cases where each of the GCs are the best alternative and **no GC can serve all use-cases optimally**. Some more details around this [here](https://kstefanj.github.io/2021/11/24/gc-progress-8-17.html#serving-different-use-cases).

## The progress

If we look as far back as JDK 8, the amount of **improvements** done to G1 and Parallel is **quite extraordinary**. These two collectors have improved in every aspect. They feature shorter pauses, use less memory and have better throughput than ever before. ZGC has not been in the mix as long, and in this post I mostly focus on the improvements brought by making **ZGC a generational collector**.

The charts in this post compare the different collectors individually. The main reason for this is that depending on the configuration of the heap size the results will be more favourable to one or the other collector. By doing this we can focus on the **great progress made for all GC**, rather than trying to crown the best GC.

The comparisons include **JDK 8**, **JDK 17** and **JDK 21**, for G1 and Parallel. For ZGC the three data points I've chosen are JDK 17, JDK 21 and **Generational ZGC** in JDK 21. Since JDK 17 was the first LTS where ZGC was fully supported it doesn't really make sense to look further back.

### Throughput

When it comes to raw throughput performance the gain since JDK 17 is not that big, but still a slight increase. But there are two things to really focus on in the chart below. First, the significant difference between JDK 8 and the recent JDKs for G1 and Parallel. From a performance point of view, leaving JDK 8 has never been more beneficial than right now.

![Throughput]({{ site.baseurl }}/assets/posts/gc-8-21/throughput.png){: class="center_85" }

The second thing to highlight is the 10% improvement seen when using Generational ZGC. The new generational support in ZGC allows it to reclaim memory more efficiently, not needing to consider the whole heap for every GC. The effect is that less CPU resources are spent doing GC work, and those resources can instead be used by the application improving its performance.

### Latency

The story is more or less the same for the latency score. G1 and Parallel see the big gains between JDK 8 and JDK 17, but the best results are still with JDK 21. We should keep in mind that between JDK 8 and JDK 17 more than 7 years of innovation took place, and between JDK 17 and 21 we only have two years. The shorter time span along with that fact that the GCs are pretty well oiled by now, make it hard to make big gains in large benchmarks like this.

![Latency]({{ site.baseurl }}/assets/posts/gc-8-21/latency.png){: class="center_85" }

The addition of generations to ZGC still makes it possible to see a significant change between Generational ZGC and the legacy mode, which is very nice. It should be noted that most of this gain comes from the fact that the throughput score improved. The length of the pauses doesn't differ much between the two ZGC modes, they are both well below 1 ms. Still, when looking at worse-case latencies, Generational ZGC is slightly better compared to legacy.

![P99-pause]({{ site.baseurl }}/assets/posts/gc-8-21/p99-pause.png){: class="center_85" }

For G1 and Parallel there have been no big changes when it comes to pause times. We spend more time on G1 and here looking at the higher percentile pause we can see that we have been able to shave off a few milliseconds.

### Footprint

The last chart compares the **peak native memory overhead** when running the benchmark with a fixed load. Parallel is very stable from this point of view and we haven't spent any time trying to optimize it more. For G1 on the other hand we have been able to cut away many inefficiencies over the last decade, and in JDK 20 we changed G1 to only need one marking bitmap instead of two. The savings from this is significant and in this benchmark G1 is now the most memory-efficient collector.

![Footprint]({{ site.baseurl }}/assets/posts/gc-8-21/footprint.png){: class="center_85" }

For Generational ZGC we can here clearly see the tradeoff we have done to get better latency and throughput in this benchmark. The cost is higher native memory consumption. To efficiently implement the generational support, we need to keep track of pointers from the old generation into the young generation. This is called remembered sets, and they consume memory. We also need some more memory for other metadata needed when handling multiple generations. This being said, in most cases the **total memory consumption is lower with Generational ZGC** when compared to legacy ZGC, because it doesn’t need as much heap to handle a given workload. So the additional native memory usage can often be saved by using a smaller heap and still experience better overall performance.

## Just upgrade and try out Generational ZGC

As you've seen by now, JDK 21 has significantly better performance compared to JDK 8. So if you are still on JDK 8, you should start looking into upgrading. When upgrading it's a great time to also re-evaluate which GC to use. If moving to JDK 21 I really encourage to try out Generational ZGC. In JDK 21 both the Generational version of ZGC and the legacy mode are available and to use Generational you need to specify both those flags:

> `-XX:+UseZGC -XX:+ZGenerational`

We eventually aim at getting rid of the legacy mode, and to do this smoothly we would like user feedback on use-cases where Generational ZGC is not performing as well as legacy ZGC. If you have a use-case like this please let us know using [this OpenJDK mailing list](https://mail.openjdk.org/mailman/listinfo/hotspot-gc-use).

For other general news and insights from the Java team at Oracle make sure to check out [inside.java](https://inside.java/).

---

[^spec]: SPEC® and the benchmark name SPECjbb® are registered trademarks of the Standard Performance Evaluation Corporation. <br>For more information about SPECjbb, see [www.spec.org/jbb2015/](http://www.spec.org/jbb2015/)
