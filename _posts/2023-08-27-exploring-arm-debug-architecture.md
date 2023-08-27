---
title: "Exploring Arm debug architecture"
published: 2023-08-27
excerpt: "This article is a deconstruction and sorting of the debug function in the Arm architecture, including a discussion of some newer debug features and IP implementations."
permalink: /posts/2023/08/exploring-arm-debug-architecture/ 
---

> This blog is also available in Chinese in https://zhuanlan.zhihu.com/p/639899970

## 0 Background

Arm's definition of the debug architecture is scattered across three documents:

- Arm ARM<a name="_ftnref1" href="#_ftn1">[1]</a>, as an instruction set manual, defines the debug/trace function within the processor, which is also the cornerstone of the debug debugging architecture
- Coresight<a name="_ftnref2" href="#_ftn2">[2]</a> architecture defines debug/trace behavior that is compatible with ARM processors, essentially an extension of the debug feature in Arm architecture
- ADI<a name="_ftnref3" href="#_ftn3">[3]</a> architecture defines the specification for the physical connection (JTAG/SWD) between Arm based SoC and the external environment

> The meaning of the name Coresight is to provide users with a visibility into the kernel. Both ARM's own legacy design suite RealView and the RISC-V camp's Sifive Insight express the same meaning

Readers familiar with these three documents will feel that they appear to have a clear division of the debug function and can effectively achieve the goal of building a unified Arm style debug infrastructure in SoC. However, from the perspective of a beginner, this decentralized architecture may be confusing. In particular, while developing a debug document independent of Arm ARM, Arm proposed two rather than one unified document, which is intuitively confusing. Considering the significant overlap in the architecture documents between Coresight and ADI, as well as the chaotic compatibility between the two, this confusion will only deepen further.

This situation is to some extent determined by the order in which Coresight and ADI appeared in history. It should be clear that the introduction of Coresight is to address the issue of debugging multi-core architectures that first appeared in ARM 11. Prior to this, the debug feature defined by the Arm architecture was sufficient to handle single core debugging scenarios, and the debug interface at that time was entirely based on the JTAG scan chain approach.

That is to say, ADI existed before the advent of Coresight. After the emergence of Coresight, Arm did not simply merge it into Coresight in order to achieve forward compatibility with ADI. In this way, ADI is architecturally compatible with the emerging multi-core Core sight architecture and the so-called legacy scan chain based (non-Coresight) architectures of ARM7 and ARM9. The former uses MEM-AP access in ADI, while the latter uses JTAG-AP access, which is also one of the meanings of the AP topology diagram in ADI documents.

![Figure 0-1](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/0-1.png)
Figure 0-1 DAP topology in ADI


On the contrary, the scope of the Coresight architecture includes a DAP implementation that conforms to the ADI architecture. That is, the Coresight architecture stipulates that its components must be debugged using the ADI component's port, while the ADI architecture indicates that the implementation of the ADI architecture may not necessarily be used to debug Coresight components.

Below is a simplified debug function block diagram in SoC to illustrate the scope of responsibility and relationships between the ARM ARM/Coresight/ADI architectures in a real system.

![Figure 0-2](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/0-2.png)
<font size=2;fony color=#C0C0C0>Figure 0-2 Debug architecture in a real system</font>

As shown in the caption, the three main colors in this schematic represent the implementation of the three architecture definitions. The debug/trace unit functions within the Core are defined by Arm ARM, such as debug breakpoint/watchpoint or ETM/ETE implementations, but their special markings in the graph indicate that they have a series of registers (PIDx/CIDx) defined by Core to support the topology detection of the Coresight system.

Another type represented by concatenated colors indicates that both architectures have relevant descriptions of the component. For CTI, there are different coloring schemes in different places. What I want to express is that although CTI itself is a modular and reusable design, the specific usage method is closely related to the deployed design. For example, in A Core, it is generally configured with several fixed events such as debug request/restart. In non Arm architecture processing units, designers need to connect the signals of interest to them themselves. This is also why the CTI of the Heterogeneous Cluster in the figure is not related to Arm ARM.

## 1 Arm debug feature
### 1.1 debug register interface

The ultimate goal of various forms of debugging is to obtain the state of the core and control its behavior. This is achieved by reading and writing debug registers within the core. Therefore, first of all, we will discuss the debug register interface, mainly answering two questions: how are these registers accessed and who can access them?

For hardware engineers, it is intuitive to first think that the relevant registers within the core need to be accessible by external debuggers. Arm calls it the *external debug interface*, which is achieved by controlling the DAP to initiate an APB transfer to the core through the debugger. Since the debugger is not accessing a memory region at this time, a mechanism is needed to obtain the address of the debug register, which is the **ROM Table**.

The field in the ROM Table that stores component addresses is a set of read-only register heaps, with each entry holding a `[x:12]` address to point to a particular component. Each component occupies 4KB of address space, which is used to store the registers of the component. Each unit within the core with an external debug interface, such as debug/trace/pmu, is indexed as a component.

The following is an example of accessing a Cortex-A core within a DynamIQ Cluster to illustrate this mechanism:

![Figure 1-1](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/1-1.png)
Figure 1-1 Debug components in DynamIQ Cluster

In the figure, the ROM Table connected to DP is called DP ROM, which is usually located at address `0x0` to discover MEM-APs in the system. For the access path to the cluster (usually using APB-AP for A core), there will be another cluster level ROM table with an address equal to the APB-AP base address `+ 0 offset` where it is located, to discover debug resources within this APB-AP subsystem.

The above figure is a simplified diagram. In the actual A core SoC, there may be more nesting from DP ROM to the final Cluster level ROM Table. The following figure is an example from the Arm Corstone SSE-710 subsystem<a name="_ftnref4" href="#_ftn4">[4]</a>:

![Figure 1-2](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/1-2.png)
Figure 1-2 ROM table structure of SSE-710

I have annotated the positions corresponding to DP ROM, APB-AP, and Cluster level ROM Table in Figure 1-1 in the upper middle. The 'Host' in SSE-710 refers to the AP (Application Processor) Compared to Figure 1-1, there are more Host ROMs and EXTDBGROMs on the path from DP to Host CPU. The former can not only point to the Cluster level ROM Table, but also to the Core sight component in the AP subsystem (roughly the green part in the dashed box in Figure 0-2); The latter involves inserting a stage between DP ROM and MEM-APs, allowing DP ROM to not only point to MEM-APs, but also to GPIO or APBCOM (related to secure debugging, as discussed below).

The TRM of Arm core will provide an external debug memory map. Taking A53<a name="_ftnref5" href="#_ftn5">[5]</a> as an example, it has a maximum of 4 cores in MPcore configuration:

![Figure 1-3](https://github.com/srleslie/srleslie.github.io/blob/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/1-3.png?raw=true)
Figure 1-3 Cortex-A53 external debug memory map

Next, we will discuss another interface for debug register. Consider making a simple supplement to Figure 1-1: create an APB port from the interconnect to bypass the entrance of the subsystem accessed by APB-AP, as shown in the following figure.

![Figure 1-4](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/1-4.png)
Figure 1-4 Debug components in DynamIQ Cluster

This routing provides the core with visibility into debug resources within a certain subsystem (which can also be extended to the entire SoC), and Arm calls it the memory mapped interface Essentially, this type of interface only reuses the external debug interface, without adding any additional interfaces to the debug register itself. By mapping external debug memory maps to the system's memory map, the core can access these debug registers without relying on external debuggers.

Observing the memory map of the Juno SoC<a name="_ftnref6" href="#_ftn6">[6]</a> integrated with A53, it can be seen from 0x2300_ The starting address of 0000 is consistent with the external debug memory map of A53, with a slight difference being that Juno SoC has inserted some Core sight components into the A53 reserve area.

![Figure 1-5](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/1-5.png)
Figure 1-5 Arm Juno SoC memory map

The external debug memory map provides support for on chip debugging at the hardware level, while in actual on chip debugging, Arm emphasizes another debugging model (different from external debugging), namely self hosted debugging This model will be discussed separately in the next section, where it will explain its impact on the debug register interface - a third interface, commonly known as the system register interface, is introduced.

System registers are concrete entities that provide Arm architecture functionality and are accessed through specialized instructions. The system register interface provides a more convenient path for software debuggers to access debug resources. Considering the sensitivity of debug registers, most debug registers with system register interfaces require EL1 or higher privilege levels. Therefore, the term 'software debugger' here generally refers to kernel level debug agents, such as Linux gdb.

### 1.2 self-hosted debug


### 1.3 debug over powerdown


## 2 Coresight & ADI


### 2.1 DAP topology


### 2.2 power control


### 2.3 secure debug

## Glossary

| **Term** | **Meaning**                |
| ADI  | Arm Debug Interface            |
| AON  | Always-ON                      |
| AP   | Access Port                    |
| ARM  | Architectural Reference Manual |
| BPU  | BreakPoint Unit                |
| CTI  | Cross Trigger Interface        |
| DAP  | Debug Access Port              |
| DP   | Debug Port                     |
| DWT  | Data Watchpoint and Trace      |
| ED   | External Debug                 |
| ETM  | Embedded Trace Macrocell       |
| ETE  | Embedded Trace Extension       |
| PPU  | Power Policy Unit              |
| SCP  | System Control Processor       |
| TRM  | Technical Reference Manual     |



<br/>

## Reference

<font size=2 color=#C0C0C0><a name="_ftn1" href="#_ftnref1">[1]</a> Arm Architecture Reference Manual for A-profile architecture https://developer.arm.com/documentation/ddi0487/ja/?lang=en</font>

<font size=2 color=#C0C0C0><a name="_ftn2" href="#_ftnref2">[2]</a> Arm CoreSight Architecture Specification v3.0 https://developer.arm.com/documentation/ihi0029/f/?lang=en

<font size=2 color=#C0C0C0><a name="_ftn3" href="#_ftnref3">[3]</a> Arm Debug Interface Architecture Specification ADIv6.0 https://developer.arm.com/documentation/ihi0074/d/?lang=en

<font size=2 color=#C0C0C0><a name="_ftnref4" href="#_ftn4">[4]</a> Arm Corstone SSE-710 Subsystem Technical Reference Manual https://developer.arm.com/documentation/102342/0000/?lang=en

<font size=2 color=#C0C0C0><a name="_ftnref5" href="#_ftn5">[5]</a> Arm Cortex-A53 MPCore Processor Technical Reference Manual https://developer.arm.com/documentation/ddi0500/j/?lang=en


