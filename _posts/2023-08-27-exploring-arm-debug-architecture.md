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

![Figure 0-1 DAP topology in ADI](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/0-1.png)

On the contrary, the scope of the Coresight architecture includes a DAP implementation that conforms to the ADI architecture. That is, the Coresight architecture stipulates that its components must be debugged using the ADI component's port, while the ADI architecture indicates that the implementation of the ADI architecture may not necessarily be used to debug Coresight components.

Below is a simplified debug function block diagram in SoC to illustrate the scope of responsibility and relationships between the ARM ARM/Corespight/ADI architectures in a real system.

![Figure 0-1 DAP topology in ADI](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/0-2.png)

As shown in the caption, the three main colors in this schematic represent the implementation of the three architecture definitions. The debug/trace unit functions within the Core are defined by Arm ARM, such as debug breakpoint/watchpoint or ETM/ETE implementations, but their special markings in the graph indicate that they have a series of registers (PIDx/CIDx) defined by Core to support the topology detection of the Core Insight system

Another type represented by concatenated colors indicates that both architectures have relevant descriptions of the component. For CTI, there are different coloring schemes in different places. What I want to express is that although CTI itself is a modular and reusable design, the specific usage method is closely related to the deployed design. For example, in A Core, it is generally configured with several fixed events such as debug request/restart. In non Arm architecture processing units, designers need to connect the signals of interest to them themselves. This is also why the CTI of the Heterogeneous Cluster in the figure is not related to Arm ARM.

## 1 Arm debug feature
### 1.1 debug register interface

The ultimate goal of various forms of debugging is to obtain the state of the core and control its behavior. This is achieved by reading and writing debug registers within the core. Therefore, first of all, we will discuss the debug register interface, mainly answering two questions: how are these registers accessed and who can access them?


### 1.2 self-hosted debug


### 1.3 debug over powerdown


## 2 Coresight & ADI


### 2.1 DAP topology


### 2.2 power control


### 2.3 secure debug

## Glossary
| Term | Meaning                        |

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

*Table syntax cannot be displayed correctly here. Click [here](https://github.com/srleslie/srleslie.github.io/blob/master/_posts/2023-08-27-exploring-arm-debug-architecture.md#glossary) for a better reading experience*


<br/>

<a name="_ftn1" href="#_ftnref1">[1]</a> Arm Architecture Reference Manual for A-profile architecture https://developer.arm.com/documentation/ddi0487/ja/?lang=en

<a name="_ftn2" href="#_ftnref2">[2]</a> Arm CoreSight Architecture Specification v3.0 https://developer.arm.com/documentation/ihi0029/f/?lang=en

<a name="_ftn3" href="#_ftnref3">[3]</a> Arm Debug Interface Architecture Specification ADIv6.0 https://developer.arm.com/documentation/ihi0074/d/?lang=en
