---
layout: post
title:  "Large pages and Java"
date:   2021-05-19 13:10:24 +0200
tags: [Performance, Java, GC, Large Pages]
---

I've recently spent a lot of time in memory reservation code of the JVM. It started out because we got an external contribution to enable use of multiple large page sizes for Linux. To do this in a good way some other things had to be refactored first. While taking this trip down _memory lane_ I realized that doing a short summary of how large pages are used by the JVM might be an interesting read.

## Introduction to large pages

Before we start talking about how the JVM makes use of them, let's start off with a brief introduction to what large pages is.

Large pages, or sometimes huge pages, is a technique to **reduce the pressure** on the processors [TLB](https://en.wikipedia.org/wiki/Translation_lookaside_buffer) caches. These caches are used to speed up the time to translate virtual addresses to physical memory addresses. Most architectures support **multiple page sizes**, often with a base page size of 4 KB. For applications using a lot of memory, for example large Java heaps, it makes sense to have the memory mapped with a **larger page granularity** to increase the chance of a hit in the TLB. On x86-64, 2 MB and 1 GB pages can be used for this purpose and for memory intense workloads this can have a really **big impact**.

![LP Enabled]({{ site.baseurl }}/assets/posts/lp/large-pages-4k-vs-2m.png){: class="center_85" }

In the above chart we can see the difference of running with and without large pages for a couple of SPECjbb®[^spec] benchmarks. The only difference in configuration is that the high performing JVMs have **large pages enabled**. The results are quite impressive and for many Java workloads enabling large pages is a big win.

## Enabling large pages

The generic switch to enable large pages for Java is `-XX:+UseLargePages`, but to leverage large pages the OS needs to be **properly configured** as well. Let's look at how this is done for Linux and Windows.

### Linux

On Linux there are mainly two different ways the JVM can make use of large pages: **Transparent Huge Pages** and **HugeTLB pages**. These differ in the way they are configured but also a bit in their **performance characteristics**.

#### Transparent Huge Pages

Transparent Huge Pages, or THP for short, is a way to simplify usage and enablement of large pages in Linux. When enabled the Linux kernel will try to use large pages for reservations large enough and eligible to use THP. THP support can be configured at three different levels:

* `always` - transparent huge pages are used automatically by any application.

* `madvise` - transparent huge pages are only used if the application uses [`madvise()`](https://man7.org/linux/man-pages/man2/madvise.2.html) with the flag `MADV_HUGEPAGE` to mark that certain memory segments should be backed by large pages.

* `never` - transparent huge pages are never used.

The configuration is stored in `/sys/kernel/mm/transparent_hugepage/enabled` and can easily be changed like this[^root]:
```
$ echo "madvise" > /sys/kernel/mm/transparent_hugepage/enabled
```

The JVM has support to make use of THP when configured in `madvise` mode, but it needs to be enabled by using `-XX:+UseTransparentHugePages`. When this is done the Java heap as well as other internal JVM data structures will be backed by transparent huge pages.

For the kernel to be able to satisfy requests to use transparent huge pages, there need to be **enough contiguous physical memory** available. If there isn't kernel will try to **defrag** memory to be able to satisfy the request. Defrag can be configured in a few different ways and the current policy is stored in `/sys/kernel/mm/transparent_hugepage/defrag`. For more details about this and other configuration, please refer to the [kernel documentation](https://www.kernel.org/doc/Documentation/vm/transhuge.txt).

#### HugeTLB pages

This type of large pages is pre-allocated by the OS and consume the physical memory used to back them. An application can reserve pages from this pool using `mmap()` with the flag `MAP_HUGETLB`. This is the default way to use large pages for the JVM on Linux and it can be enabled by either setting `-XX:+UseLargePages` or the specific flag `-XX:+UseHugeTLBFS`.

When the JVM uses this type of large pages it commits the whole memory range backed by large pages up front[^zgc]. This is needed to ensure that no other reservation depletes the pool of large pages allocated by the OS. This also means that there need to be enough large pages pre-allocated to back the whole memory range at the time of the reservation, otherwise the JVM will fall back to use normal pages.

To configure this type of large pages first check the page sizes available:
```
$ ls /sys/kernel/mm/hugepages/
hugepages-1048576kB  hugepages-2048kB
```
Then configure the number of pages you like for a give size like this[^root]:
```
$ echo 2500 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```
This tries to allocate 2500 2 MB pages. One should always read the value actually stored in `nr_hugepages` to make sure the kernel was able to allocate the requested amount.

By default, the JVM will use the environments default large page size when trying to reserve large pages, to see your systems default large page size run:
```
$ cat /proc/meminfo | grep Hugepagesize
Hugepagesize:       2048 kB
```
If you want to use a different large page size this can be done by setting the JVM flag `LargePageSizeInBytes`. For example, to use 1 GB pages run with `-XX:LargePageSizeInBytes=1g`.

More information about HugeTLB pages can also be found in the [kernel documentation](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt).

#### Which one to use

Both approaches have pros and cons and which one to choose depends on more than one aspect. THP is easier to setup and use, but you have more control when using HugeTLB pages. If **latency** is your biggest concern, you should probably **go with HugeTLB pages** since you will never stall waiting for the OS to free up enough contiguous memory. As an alternative you can configure the `defrag` option of THP to not stall if no large pages are available, but this might come with a throughput cost instead. If **memory footprint** is a concern **THP is the better choice** to avoid having to commit the whole Java heap up front.

Which type of large pages to use depends on the application and the environment, but in many cases just using any type of large pages will have a **positive effect on performance**.

### Windows

On Windows the configuration step is a bit easier, at least on newer versions. The user running the process that want to make use of large pages need to have permission to *Lock Pages in Memory*. On Windows 10 this is done by:
1. Run `gpedit.msc`
2. Find "User Rights Assignment" under "Computer Configuration" > "Windows Settings" > "Security Settings" > "Local Policies"
3. Locate "Lock pages in memory" and double click it
4. Click "Add User or Group" and add the correct user
5. Log out or restart to make the changes take effect

Once this permission is granted the JVM will be able to use large pages if run with `-XX:+UseLargePages`. The JVMs large pages implementation on Windows works very similar to HugeTLB pages on Linux. The whole reservation backed by large pages is committed up front to ensure we don't get any failures later on.

Until recently there was a [bug](https://bugs.openjdk.java.net/browse/JDK-8266489) preventing G1, the default GC, to make use of large pages for heaps larger than 4 GB on Windows. This is now fixed and going forward running big **Minecraft servers** with **G1** should be able to get a nice **boast by enabling large pages**.

## Checking the JVM

Once your environment is properly configured and you've enabled Java to run with large pages, it is **good to verify** that the JVM really makes use of large pages. You can of course use your favorite OS tool to check this, but the JVM also has a few **logging options** to help with this. To see some basic GC configuration you can run with `-Xlog:gc+init`. With G1 this you get this output:
```
> jdk-16/bin/java -Xlog:gc+init -XX:+UseLargePages -Xmx4g -version
[0.029s][info][gc,init] Version: 16+36-2231 (release)
[0.029s][info][gc,init] CPUs: 40 total, 40 available
[0.029s][info][gc,init] Memory: 64040M
[0.029s][info][gc,init] Large Page Support: Enabled (Explicit)
[0.029s][info][gc,init] NUMA Support: Disabled
[0.029s][info][gc,init] Compressed Oops: Enabled (Zero based)
[0.029s][info][gc,init] Heap Region Size: 2M
[0.029s][info][gc,init] Heap Min Capacity: 8M
[0.029s][info][gc,init] Heap Initial Capacity: 1002M
[0.029s][info][gc,init] Heap Max Capacity: 4G
[0.029s][info][gc,init] Pre-touch: Disabled
[0.029s][info][gc,init] Parallel Workers: 28
[0.029s][info][gc,init] Concurrent Workers: 7
[0.029s][info][gc,init] Concurrent Refinement Workers: 28
[0.029s][info][gc,init] Periodic GC: Disabled
```
This is run on Linux and we can see that _Large Page Support_ is enabled. It says _Explicit_ which means that HugeTLB pages are used. If run with `-XX:+UseTransparentHugePages` the log line would look like this:
```
[0.030s][info][gc,init] Large Page Support: Enabled (Transparent)
```

The above only shows if large pages are enabled or not, if you like more detailed information about which parts of the JVM that uses large pages you can enable `-Xlog:pagesize` and get output like this:
```
[0.002s][info][pagesize] CodeHeap 'non-nmethods':  min=2496K max=8M base=0x00007fed3d600000 page_size=4K size=8M
[0.002s][info][pagesize] CodeHeap 'profiled nmethods':  min=2496K max=116M base=0x00007fed3de00000 page_size=4K size=116M
[0.002s][info][pagesize] CodeHeap 'non-profiled nmethods':  min=2496K max=116M base=0x00007fed45200000 page_size=4K size=116M
[0.026s][info][pagesize] Heap:  min=8M max=4G base=0x0000000700000000 page_size=2M size=4G
[0.026s][info][pagesize] Block Offset Table: req_size=8M base=0x00007fed3c000000 page_size=2M alignment=2M size=8M
[0.026s][info][pagesize] Card Table: req_size=8M base=0x00007fed3b800000 page_size=2M alignment=2M size=8M
[0.026s][info][pagesize] Card Counts Table: req_size=8M base=0x00007fed3b000000 page_size=2M alignment=2M size=8M
[0.026s][info][pagesize] Prev Bitmap: req_size=64M base=0x00007fed37000000 page_size=2M alignment=2M size=64M
[0.026s][info][pagesize] Next Bitmap: req_size=64M base=0x00007fed33000000 page_size=2M alignment=2M size=64M
```
This is pretty detailed information, but it is a good way to verify which parts of the JVM are backed by large pages. The output above is generated using JDK 16 and it has a [bug](https://bugs.openjdk.java.net/browse/JDK-8261029) which causes the _CodeHeap_ page sizes to be incorrect, they are also backed by large pages.

# &nbsp; {#posts-label}

[^spec]: SPEC® and the benchmark name SPECjbb® are registered trademarks of the Standard Performance Evaluation Corporation. <br>For more information about SPECjbb, see [www.spec.org/jbb2015/](http://www.spec.org/jbb2015/)

[^root]: For this to work you need to have permission to write to the files in question.

[^zgc]: This is not the case for the ZGC heap which instead handles commit failures by limiting the heap size.