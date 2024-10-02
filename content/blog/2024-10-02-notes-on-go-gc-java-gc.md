+++
title = "Notes on Go GC and Java GC"
+++

All the contents below are based on [A Guide to the Go Garbage Collector](https://go.dev/doc/gc-guide) of Go 1.23.  
It has amazing visualizations to help readers understand the how Go GC works, which I don't see in official Java docs.

---
Go has very few GC control options, only 2: `GOGC` and `GOMEMLIMIT`.  
While Java has many different collectors (serial, parallel, CMS, G1, ZGC, shenandoah...), and each of them has many options.

Go GC is mark-sweep GC, so much like Java's CMS.

Go defines a GC cycle as sweeping -> idle -> marking. The last phase is not *sweeping* but is *marking*.  
One reason may be `GOGC` determines the target heap size after each GC cycle, so cycle end == marking end, which is the time we know the live heap size.  
In the current implemetation, sweeping is fast, the cost can be ignored compared to marking.


> Target heap memory = Live heap + (Live heap + GC roots) * GOGC / 100

`GOGC` triggers a new GC when heap size reach the target size, which means it controls the GC frequency, also the trade-off between cpu and mem.  
While Java triggers GC when eden area (minor) or old generation (major) is full.  
So it's much simple to control when GC occurs in Go.

`GOGC` can be changed in real time. `-Xmx` can't.

> [D]oubling GOGC will double heap memory overheads and roughly halve GC CPU cost

Go added `GOMEMLIMIT` in 1.19 to solve the problem that GOGC has to be set based on peak live heap size. In such cases, mem is not fully used in steady-state. With the help of `GOMEMLIMIT`, `GOGC` can be set based on steady-state.  

`GOMEMLIMIT` functions similar with Java's `-Xmx`.  
From the Java world, it's hard to image why it takes so long to add this parameter, since `-Xmx` is a very fundimental one.  

If `GOMEMLIMIT` is not set, Go has no upper mem limit if physical permits. While Java heap upper limit is decided by runtime if not set by `-Xmx`.

If `GOGC` is not set, `GOMEMLIMIT` is set, then this repensents a maximization of resource economy, just like Java.

`GOMEMLIMIT` is a soft limit. Go has an upper limit on the CPU time GC can use: 50%, in time window: `2*GOMAXPROCS` CPU-seconds.  
So if GC time reaches this limit, mem usage will grow beyond the `GOMEMLIMIT` to ensure programs make reasonable progress.  
While Java has default `-XX:+UseGCOverheadLimit`, will throw an `OutOfMemoryError` if more than 98% of the total time is spent on garbage collection and less than 2% of the heap is recovered.

> [M]ost of the costs for the GC are incurred while the mark phase is active