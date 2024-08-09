---
title: "Perf on Arm"
published: 2024-01-04
excerpt: "This article introduces Arm's perf infrastructure from a hardware perspective and demonstrates it with some practical code."
permalink: /posts/2024/01/perf-on-arm/ 
---

> This blog is also available in Chinese in https://zhuanlan.zhihu.com/p/671540004

It has almost become a consensus that the success of Arm needs to be attributed more to its ecological prosperity rather than its superior architecture or performance. As a latecomer, Arm does not have much room for development in some architectural designs, especially in areas where the upper level software has already matured. Therefore, due to helplessness or pragmatism, Arm followed the approach of x86 in its corresponding architecture design. The performance analysis system design discussed in this article belongs to one of these fields.

Perf is a widely used and influential performance analysis tool on the x86 Linux platform. In order to enable developers to achieve an out of the box experience without increasing learning costs on the Arm platform, ARM must concoct x86's support for perf in the same way. Especially in the context of Arm's fierce pursuit of the server market in the past two years, considering that comprehensive performance analysis support is indispensable for software migration work, Arm has accelerated the maturity of this technology.

At Linaro Connect 2023, a team from Arm introduced a work called "Arm Telemetry Solution [^1]", which is actually a comprehensive solution on how Arm supports performance analysis at the hardware level. The following figure shows the three components of this solution. These three parts are actually the arch features of the Arm instruction set.

![Figure 1](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2024-01-04-perf-on-arm/1.png)

Firstly, PMU forms the cornerstone of the entire system. Although from a hardware perspective, PMU is just a set of counters, its truly important components are the intricately intertwined event signals from other modules in the core, such as IFU, LSU, etc. These event signals provide the lowest level status information of the core, which can be recognized by tools such as perf and provided to users as software and hardware interfaces to some extent. From this perspective, PMU and debug unit have considerable commonality, both of which are used to provide kernel visibility to upper level software.
SPE and BRBE are used to optimize statistical results based on PMU or further explore hotspot paths. As pointed out in the above figure, SPE and BRBE are extensions brought by Arm A-profile v8.2 and v8.7, respectively.

SPE has been supported in many kernels, and its adaptation and development have become relatively mature. The Arm team mentioned above has just released a whitepaper on SPE applications [^2], which is also an important reference for this article; The BRBE extension itself was released relatively late, and as of the release of v9.2 generation products so far, there is no kernel supporting this extension, and there is not much official information provided.

> It is generally believed that SPE corresponds to Intel PEBS, while BRBE corresponds to Intel LBR. However, this article [^3] points out in more detail that compared to Intel PEBS and AMD IBS, Arm SPE is more similar in implementation mechanism to the latter

The ultimate goal of the above hardware resources is to make perf on Arm more comprehensive and powerful. Next, we will conduct a routine performance analysis using perf, exploring the role of these arch features in perf, analyzing the working process of SPE/BRBE, and the necessity of introducing them.

***

Perf supports a wide range of options, and a simple but typical performance analysis may include the following options:

```
perf list
perf stat
perf record
perf report
```
Although there are only a few PMCs in each core of the CPU (usually 6 in Arm), each of them can be configured as any event in a larger event space (usually supporting more than 100 events), which varies across different cores. The perf list is used to inform users of the event spaces supported by the current platform. As a preparatory task, it did not conduct any analysis of the code under test.

PMU uses system registers to record event counts, which can be configured to generate interrupts. These two hardware features are respectively used by perf in two analysis mechanisms, commonly referred to as counting mode and sampling mode, corresponding to perf stat and perf record/report, respectively These two mechanisms are not interchangeable options provided to achieve the same goal, they provide different perspectives for analyzing the load. The perf stat command accesses the system register after the load ends according to the configured event option to provide the statistical results of the specified event, which is more consistent with our intuitive purpose of calling PMU. Here is a simplified example [^4]:

```
$ perf stat -e 
$ perf stat -e cycles,instructions,branches,branch-misses -- ./workload
10580290629 cycles # 3,677 GHz
 8067576938 instructions # 0,76 insn per cycle
 3005772086 branches # 1044,472 M/sec
  239298395 branch-misses # 7,96% of all branches
```
The perf record, on the other hand, samples kernel states at specified intervals based on the configured event option. Combined with the perf report, it arranges the functions corresponding to the most sampled kernel states, namely the hotspot function, which is the performance bottleneck of the load to be optimized. Perf record is an event based sampling, and a special case is to configure the event as CPU cycles, which is equivalent to time based sampling Obviously, event based sampling is a more fine-grained analysis. **If we can accurately identify the bottleneck characteristics of the load,** such as cache miss, then event based sampling can eliminate other possible interferences and guide us towards a more accurate hotspot path. Here is also an example of perf record/report:

```
$ perf record -e branches -- ./x264
$ perf report 
# Samples: 364K of event 'cycles:ppp' # Event count (approx.): 300110884245 
# Overhead Samples Shared Object Symbol 
# ........ ....... ............. ........................................ 
# 
      6.99% 25349 x264 [.] x264_8_me_search_ref 
      6.70% 24294 x264 [.] get_ref_avx2 
      6.50% 23397 x264 [.] refine_subpel 
      5.20% 18590 x264 [.] x264_8_pixel_satd_8x8_internal_avx2 
      4.69% 17272 x264 [.] x264_8_pixel_avg2_w16_sse2 
      4.22% 15081 x264 [.] x264_8_pixel_avg2_w8_mmx2 
      3.63% 13024 x264 [.] x264_8_mc_chroma_avx2 
      3.21% 11827 x264 [.] x264_8_pixel_satd_16x8_internal_avx2 
      2.25%  8192 x264 [.] rd_cost_mb
```

It should be noted that only by accurately identifying the bottleneck characteristics of the load can one enjoy the convenience brought by fine-grained analysis. This bottleneck feature comes from the results in perf stat. We determine which type of feature the bottleneck may be attributed to by identifying which events contribute significantly to the load execution. However, since perf stat only returns the count of events we specify, it seems like we are caught in a trap of interdependence - how can we find events with higher counts?

The simplest and most direct method is to count all events at once, and then arrange their count values in order. However, considering that the number of events supported by the kernel far exceeds its PMC, even if multiplexing and scaling can be used to some extent to compensate for this gap, this approach may still not be feasible. Therefore, currently, perf has not implemented this simple and crude full caliber statistics. Intel offers a more elegant solution, Top down Microarchitecture Analysis TMA organizes the originally flat event space based on the relevance of microarchitecture, establishing a hierarchical structure to guide developers in discovering hot events through decision trees. The following figure shows an example of the hierarchical structure of TMA.

![Figure 2](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2024-01-04-perf-on-arm/2.png)

In the above figure, since the lower level events are a subset of their corresponding higher-level events, the low-level detailed microarchitecture hotspot events will be transmitted to the high-level abstract clustering events. In this way, with clues to go in the right direction, both developers and tools can safely ignore unimportant events, avoiding enumerating the entire event space, making it possible to search for hotspots in engineering.

Quoting the expression of the TMA author in the original paper proposing this work [^5], using TMA to analyze hotspots is a process of "drill down repeatedly in a hierarchical Manner until a specific performance issue is determined". The following example illustrates this point intuitively by performing three analyses step by step (with options indicating - l1/- l2/- l3). The most hot event in the example is DRAM Bound, so in the final level 3 analysis, the tool focused on statistical BE_ Bound at the level 3 level The subset of MemeBound also ignores the BadSpeculation and Retiring events at level 2.

> It should be noted that due to the limited support of perf for the top-down option, including the following examples, more pmu tools/toplev tools that encapsulate perf stat are used for top-down analysis on x86 platforms

```
$ python toplev.py --core S0-C0 -l1 -v --no-desc taskset -c 0 ./workload
# Level 1
S0-C0 Frontend_Bound: 13.81 % Slots
S0-C0 Bad_Speculation: 0.22 % Slots
S0-C0 Backend_Bound: 53.43 % Slots <==
S0-C0 Retiring: 32.53 % Slots

$ python toplev.py --core S0-C0 -l2 -v --no-desc taskset -c 0 ./workload
# Level 1
S0-C0 Frontend_Bound: 13.92 % Slots
S0-C0 Bad_Speculation: 0.23 % Slots
S0-C0 Backend_Bound: 53.39 % Slots
S0-C0 Retiring: 32.49 % Slots
# Level 2
S0-C0 Frontend_Bound.FE_Latency: 12.11 % Slots
S0-C0 Frontend_Bound.FE_Bandwidth: 1.84 % Slots
S0-C0 Bad_Speculation.Branch_Mispred: 0.22 % Slots
S0-C0 Bad_Speculation.Machine_Clears: 0.01 % Slots
S0-C0 Backend_Bound.Memory_Bound: 44.59 % Slots <==
S0-C0 Backend_Bound.Core_Bound: 8.80 % Slots
S0-C0 Retiring.Base: 24.83 % Slots
S0-C0 Retiring.Microcode_Sequencer: 7.65 % Slots

$ python toplev.py --core S0-C0 -l3 -v --no-desc taskset -c 0 ./workload
# Level 1
S0-C0 Frontend_Bound: 13.91 % Slots
S0-C0 Bad_Speculation: 0.24 % Slots
S0-C0 Backend_Bound: 53.36 % Slots
S0-C0 Retiring: 32.41 % Slots
# Level 2
S0-C0 FE_Bound.FE_Latency: 12.10 % Slots
S0-C0 FE_Bound.FE_Bandwidth: 1.85 % Slots
S0-C0 BE_Bound.Memory_Bound: 44.58 % Slots
S0-C0 BE_Bound.Core_Bound: 8.78 % Slots
# Level 3
S0-C0-T0 BE_Bound.Mem_Bound.L1_Bound: 4.39 % Stalls
S0-C0-T0 BE_Bound.Mem_Bound.L2_Bound: 2.42 % Stalls
S0-C0-T0 BE_Bound.Mem_Bound.L3_Bound: 5.75 % Stalls
S0-C0-T0 BE_Bound.Mem_Bound.DRAM_Bound: 47.11 % Stalls <==
S0-C0-T0 BE_Bound.Mem_Bound.Store_Bound: 0.69 % Stalls
S0-C0-T0 BE_Bound.Core_Bound.Divider: 8.56 % Clocks
S0-C0-T0 BE_Bound.Core_Bound.Ports_Util: 11.31 % Clocks
```
This blog about performance analysis [^6] and the whitepaper with its links introduce the top-down analysis methodology, indicating that Arm simply and straightforwardly followed Intel TMA However, perhaps due to the need to maintain a minimum level of dignity, Arm has created some differentiation here. Arm divides its top-down process into two stages and defines several metrics The so-called metric refers to performing simple operations on multiple original events to obtain more informative indicators. For example, define ipc=INST.RETIRED/CPU-CYCLES, or l1i_cache_mpki L1I-CACHE-REFILL/INST.RETIRED * 1000.

Arm defines stage 1 as Topdown Analysis, which is the process of truly searching for hotspots; Stage 2 is Micro architecture exploration, which assumes that the hotspot has been found and further analyzes the hotspot event. The diagram below shows a schematic diagram of two stages:

![Figure 3](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2024-01-04-perf-on-arm/3.png)

There are two aspects worth mentioning in this diagram: firstly, from a hardware perspective, in terms of top-down methodology, only the lowest level events actually require the kernel design team to spend time and effort defining and designing. Any event in any layer above the lowest level can be indirectly expressed through lower level events, whether through hardware or software methods. That is to say, upper level events are redundant, and benefiting from the efficiency of top-down is not without cost. In the above figure, it can be approximated that stage 2 corresponds to the bottom level event, while stage 1 corresponds to the top level event. Compared to the Intel TMA hierarchy, some intermediate levels are missing here. Arm also emphasized in whitepaper that currently stage 1 only provides level 1 events, which means that if Arm enriches intermediate level events in the future, these level 2/level 3 events will be used to expand the block diagram of stage 1 in the above figure, as they are collectively used to form a decision tree for finding hot spots.

Another point is that in the hierarchy of Intel TMA, any bottommost event has only one unique path to point to a unique top-level event. However, it seems that Arm has a different view on this. If we assume that the dashed line in the above figure represents a corresponding relationship (which is not explicitly stated in the whitepaper), there will be a situation where stage 2 events correspond to multiple stage 1 events simultaneously. This approach clearly increases the complexity of designing intermediate events, which may be why only level 1 events are currently provided. At present, the intention of Arm here has not been fully understood.

Arm also provides a tool similar to pmu tools/toplev called top-down tools [^7] Below is also a test case for this tool If I compare the display results of this tool with those of pmu tools/toplev, I personally feel that the appearance of Arm tool may be slightly inferior, possibly due to the lack of intermediate events, and the direct exposure of underlying events makes people feel abrupt and unable to find the key point. In addition, for the display of event statistics results, absolute values or percentages are already very intuitive data, and the so-called metrics may actually make people feel redundant.

```
$ topdown-tool -s combined ./workload
[Cycle Accounting]                   [Topdown group]
Frontend stalled Cycles...............0.00% cycles
  [Branch Effectiveness]             [uarch group]
  Branch MPKI.........................0.371 misses per 1,000 instructions...
  Branch Misprediction Ratio..........0.001 per branch

  [Instruction TLB Effectiveness]     [uarch group]

Backend stalled Cycles................42.78% cycles
  [Data TLB Effectiveness]            [uarch group]
  DTLB MPKI...........................0.000 misses per 1,000 instructions
  L1 Data TLB MPKI....................0.002 misses per 1,000 instructions
  L2 Unified TLB MPKI.................0.000 misses per 1,000 instructions
  DTLB Walk Ratio.....................0.000 per TLB access
  L1 Data TLB Miss Ratio..............0.000 per TLB access
  L2 Unified TLB Miss Ratio...........0.002 per TLB access

  [L1 Data Cache Effectiveness]       [uarch group]
  ...
```

The above TMA content is essentially an in-depth discussion of perf stat. After a given hotspot event, it is necessary to further locate the function or code that caused the hotspot event, so we will further discuss the relevant issues of perf record/report.

The following content will be added later :)

## Reference

[^1]: https://static.sched.com/hosted_files/linaroconnect2023/a9/Arm_Telemetry_Solution_Linaro_Connect_2023_Jumana_Mundichipparakkal.pdf

[^2]: https://developer.arm.com/documentation/109429/latest/

[^3]: https://www.semanticscholar.org/paper/Precise-Event-Sampling-on-AMD-Versus-Intel%3A-and-Sasongko-Chabbi/bfd22bbaac5fcb14e3dd1bda7b17a9c49916174e

[^4]: The code examples in this article are all from this book https://faculty.cs.niu.edu/~winans/notes/patmc.pdf

[^5]: https://www.semanticscholar.org/paper/A-Top-Down-method-for-performance-analysis-and-Yasin/6776ff919597ca4feccd413208dedc401f6e655d

[^6]: https://community.arm.com/arm-community-blogs/b/infrastructure-solutions-blog/posts/arm-neoverse-v1-top-down-methodology

[^7]: https://gitlab.arm.com/telemetry-solution/telemetry-solution

