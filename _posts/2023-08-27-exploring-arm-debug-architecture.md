---
title: "Exploring Arm debug architecture"
published: 2023-08-27
excerpt: "This article is a deconstruction and sorting of the debug function in the Arm architecture, including a discussion of some newer debug features and IP implementations."
permalink: /posts/2023/08/exploring-arm-debug-architecture/ 
---

> This blog is also available in Chinese in https://zhuanlan.zhihu.com/p/639899970

## 0 Background

Arm's definition of the debug architecture is scattered across three documents:

- Arm ARM[^1], as an instruction set manual, defines the debug/trace function within the processor, which is also the cornerstone of the debug debugging architecture
- Coresight[^2] architecture defines debug/trace behavior that is compatible with ARM processors, essentially an extension of the debug feature in Arm architecture
- ADI[^3] architecture defines the specification for the physical connection (JTAG/SWD) between Arm based SoC and the external environment

> The meaning of the name Coresight is to provide users with a visibility into the kernel. Both ARM's own legacy design suite RealView and the RISC-V camp's Sifive Insight express the same meaning

Readers familiar with these three documents will feel that they appear to have a clear division of the debug function and can effectively achieve the goal of building a unified Arm style debug infrastructure in SoC. However, from the perspective of a beginner, this decentralized architecture may be confusing. In particular, while developing a debug document independent of Arm ARM, Arm proposed two rather than one unified document, which is intuitively confusing. Considering the significant overlap in the architecture documents between Coresight and ADI, as well as the chaotic compatibility between the two, this confusion will only deepen further.

This situation is to some extent determined by the order in which Coresight and ADI appeared in history. It should be clear that the introduction of Coresight is to address the issue of debugging multi-core architectures that first appeared in ARM 11. Prior to this, the debug feature defined by the Arm architecture was sufficient to handle single core debugging scenarios, and the debug interface at that time was entirely based on the JTAG scan chain approach.

That is to say, ADI existed before the advent of Coresight. After the emergence of Coresight, Arm did not simply merge it into Coresight in order to achieve forward compatibility with ADI. In this way, ADI is architecturally compatible with the emerging multi-core Core sight architecture and the so-called legacy scan chain based (non-Coresight) architectures of ARM7 and ARM9. The former uses MEM-AP access in ADI, while the latter uses JTAG-AP access, which is also one of the meanings of the AP topology diagram in ADI documents, as shown in the following Figure 0-1.

![Figure 0-1](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/0-1.png)

On the contrary, the scope of the Coresight architecture includes a DAP implementation that conforms to the ADI architecture. That is, the Coresight architecture stipulates that its components must be debugged using the ADI component's port, while the ADI architecture indicates that the implementation of the ADI architecture may not necessarily be used to debug Coresight components.

The following Figure 0-2 depicts a simplified section of the debug function in SoC, which illustrates the scope of responsibility and relationships between the Arm ARM/Coresight/ADI architectures in a real system.

![Figure 0-2](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/0-2.png)

As shown in the caption, the three main colors in this schematic represent the implementation of the three architecture definitions. The debug/trace unit functions within the Core are defined by Arm ARM, such as debug breakpoint/watchpoint or ETM/ETE implementations, but their special markings in the graph indicate that they have a series of registers (PIDx/CIDx) defined by Core to support the topology detection of the Coresight system.

Another type represented by concatenated colors indicates that both architectures have relevant descriptions of the component. For CTI, there are different coloring schemes in different places. What I want to express is that although CTI itself is a modular and reusable design, the specific usage method is closely related to the deployed design. For example, in A Core, it is generally configured with several fixed events such as debug request/restart. In non Arm architecture processing units, designers need to connect the signals of interest to them themselves. This is also why the CTI of the Heterogeneous Cluster in the figure is not related to Arm ARM.

## 1 Arm debug feature
### 1.1 debug register interface

The ultimate goal of various forms of debugging is to obtain the state of the core and control its behavior. This is achieved by reading and writing debug registers within the core. Therefore, first of all, we will discuss the debug register interface, mainly answering two questions: how are these registers accessed and who can access them?

For hardware engineers, it is intuitive to first think that the relevant registers within the core need to be accessible by external debuggers. Arm calls it the *external debug interface*, which is achieved by controlling the DAP to initiate an APB transfer to the core through the debugger. Since the debugger is not accessing a memory region at this time, a mechanism is needed to obtain the address of the debug register, which is the **ROM Table**.

The field in the ROM Table that stores component addresses is a set of read-only register heaps, with each entry holding a `[x:12]` address to point to a particular component. Each component occupies 4KB of address space, which is used to store the registers of the component. Each unit within the core with an external debug interface, such as debug/trace/pmu, is indexed as a component.

The following Figure 1-1 is an example of accessing a Cortex-A core within a DynamIQ Cluster to illustrate this mechanism:

![Figure 1-1](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/1-1.png)

In this figure, the ROM Table connected to DP is called DP ROM, which is usually located at address `0x0` to discover MEM-APs in the system. For the access path to the cluster (usually using APB-AP for A core), there will be another cluster level ROM table with an address equal to the APB-AP base address `+ 0 offset` where it is located, to discover debug resources within this APB-AP subsystem.

However, Figure 1-1 is a simplified diagram. In the actual A core SoC, there may be more nesting from DP ROM to the final Cluster level ROM Table. The following Figure 1-2 is an example from the Arm Corstone SSE-710 subsystem[^4]:

![Figure 1-2](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/1-2.png)

I have annotated the positions corresponding to DP ROM, APB-AP, and Cluster level ROM Table in Figure 1-1 in the upper middle. The 'Host' in SSE-710 refers to the AP (Application Processor) Compared to Figure 1-1, there are more Host ROMs and EXTDBGROMs on the path from DP to Host CPU. The former can not only point to the Cluster level ROM Table, but also to the Core sight component in the AP subsystem (roughly the green part in the dashed box in Figure 0-2). The latter involves inserting a stage between DP ROM and MEM-APs, allowing DP ROM to not only point to MEM-APs, but also to GPIO or APBCOM (related to secure debugging, as discussed below).

The TRM of Arm core will provide an external debug memory map. Taking A53[^5] as an example, it has a maximum of 4 cores in MPcore configuration, as shown in the following Figure 1-3.

![Figure 1-3](https://github.com/srleslie/srleslie.github.io/blob/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/1-3.png?raw=true)

Next, we will discuss another interface for debug register. Consider making a simple supplement to Figure 1-1: create an APB port from the interconnect back to the entrance of the subsystem accessed by APB-AP, as shown in the following Figure 1-4.

![Figure 1-4](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/1-4.png)

This routing provides the core with visibility into debug resources within a certain subsystem (which can also be extended to the entire SoC), and Arm calls it the memory mapped interface Essentially, this type of interface only reuses the external debug interface, without adding any additional interfaces to the debug register itself. By mapping external debug memory maps to the system's memory map, the core can access these debug registers without relying on external debuggers.

Observing the memory map of the Juno SoC[^6] integrated with A53 in the following Figure 1-5, it can be seen from 0x2300_ The starting address of 0000 is consistent with the external debug memory map of A53, with a slight difference being that Juno SoC has inserted some Core sight components into the A53 reserve area.

![Figure 1-5](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/1-5.png)

The external debug memory map provides support for on chip debugging at the hardware level, while in actual on chip debugging, Arm emphasizes another debugging model (different from external debugging), namely self-hosted debugging This model will be discussed separately in the next section, where it will explain its impact on the debug register interface - a third interface, commonly known as the system register interface, is introduced.

System registers are concrete entities that provide Arm architecture functionality and are accessed through specialized instructions. The system register interface provides a more convenient path for software debuggers to access debug resources. Considering the sensitivity of debug registers, most debug registers with system register interfaces require EL1 or higher privilege levels. Therefore, the term 'software debugger' here generally refers to kernel level debug agents, such as Linux gdb.

### 1.2 self-hosted debug

As mentioned earlier, self-hosted debugging is one of the two debugging models defined by the Arm architecture. These two models are not different choices that can be substituted for each other when facing the same requirement, and they are generally considered to be used in different scenarios:

**External debug** is mainly used in the debugging scenarios of bare metal, for hardware debugging or software bring-up To use external debugging, the chip needs to be connected to a debug probe (JLink/DSTREAM) through IO, and then connected to a Host host to run the development environment on the host as the debugger.

**Self-hosted debug** is mainly used in systems with already deployed OS for software debugging. Using self hosted debug eliminates the need to build a connection between the chip and the external, and uses the kernel/exception handler as the debugger.

Hardware engineers, including myself, are generally more familiar with the former because we are in a relatively 'left' position in the development cycle of a chip. When SoC is integrated by OEM, it is highly likely that there will be no JTAG interface left (some development boards may not even have it), so it is necessary for software developers to perform on chip debugging without the help of debug probes.

It should be noted that the on chip debug mentioned here does not mean that no external devices are needed. Arm often uses gdb based remote debugging as an example when introducing self-hosted debug[^7], as shown in the following Figure 1-6. In this case, in addition to the debug target being debugged, an additional host is also required to assist in debugging work. The most fundamental difference between the two debugging models lies in the different behaviors after a debug event (such as a breakpoint match) is triggered: in the external debug model, the core enters a halt state and hands control to the external debugger, which can obtain the internal state of the core through the DCC register. In the self hosted debug model, the core will report exceptions and trap them into the corresponding EL level for exception handling.

![Figure 1-6](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/1-6.png)

On the debug exception routing, Arm defines various subdivision models:

- Application debugging: EL0 exception trap to EL1 debugger
- Kernel debugging: EL0/EL1 exception trap to EL1 debugger
- OS debugging: EL0/EL1 exception trap to EL2 debugger
- Hypervisor debugging: EL0/EL1/EL2 exception trap to EL2 debugger

To my knowledge, two remote debugging methods based on gdb, namely host gdb + target gdbserver (as shown in Figure 1-6) and host gdb + target kgdb, can correspond to two models, Application debugging and Kernel debugging, respectively. Among them, gdbserver, as a user mode program, traps into the kernel through the ptrace interface of the kernel. Kgdb itself runs in kernel mode and has the ability to directly debug kernel code. Kgdb was merged into the kernel after 2.6.25 and requires architecture support. Taking the single step debugging mechanism under the self-hosted debug model as an example, we will briefly trace the hierarchical relationship between kgdb and Arm architecture code.

Similar to external debugging, Arm ARM also draws a state machine diagram for self-hosted debug in single step debugging, as shown in the following Figure 1-7. MDSCR_ EL1. SS is a global enable for single step debugging, while PSTATE.SS is a status bit. From the figure, it can be seen that when the core is in the debug exception (triggered by a debug event, such as a breakpoint), MDSCR can be set by_ EL1. SS and PSTATE.SS are both set to 1 to allow the core to enter a single step debugging state when exiting the exception, that is, to reach the active not pending and active pending states, and to trigger the debug exception again, completing a single step debugging.

> In the Arm architecture, both external and self hosted debugging models provide a single step debugging mechanism. For a single step debugging process, the comparison between the two mechanisms is as follows:
> <br/><br/>For software steps:
> <br/>PE in exception ->Configure software step ->exception return ->Execute an instruction ->Software step exception ->re entity exception
> <br/><br/>For the halting step:
> <br/>PE in halt state ->configuring halting step ->exit halt state ->executing an instruction ->halting step debug event ->re entry halt state

![Figure 1-7](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/1-7.png)

Correspondingly, let's take a look at the implementation of the Linux kernel. Here is an overall relationship diagram (Figure 1-8):

![Figure 1-8](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/1-8.png)

The *kernel/debug* directory stores the architecture agnostic top-level implementation of gdb. The `gdb_serial_stub()` function in the kernel/debug/gdbstub.c file provides the processing function of command/package (c for continue, s for step, etc.) in remote GDB communication. The scenario here is that the debugging stops at a certain point and waits for the user to input a command at the terminal, corresponding to the core being in the debug exception state in the previous paragraph. For the processing of the s command, `gdb_serial_stub()` will call the architecture defined `kgdb_arch_handle_exception()` function, so this function is located in the arch/arm/kernel directory.

Function `kgdb_arch_handle_exception()` mainly does two things to complete the s command: the first is to use `kgdb_arch_update_addr()` to try to obtain data from the command packet to update the PC, but it is easy to misunderstand the comments here. The PC is copied to ELR during exception entry, isn't it? Why is the opposite? This is because the PC mentioned in the comment refers to the processor state in the kernel stack, rather than the PC register in the hardware sense, where the `linux regs` here are actually defined by strcuct `pt_regs`. So the intention here is to change the `pt_regs.pc` to change the content in ELR and skip to the address specified in ELR during exception return, corresponding to the step of 'Programs the ELR_ELx...' in the single step debugging state machine. But for single step debugging, PC is generally not rewritten.

The second thing `kgdb_arch_handle_exception()` does is call the `kernel_enable_single_step()` to enable single step debugging. Through single-step debugging of the state machine, it can be seen that the enabling process is achieved by accessing the system register, so this step is architecture defined. The implementation of this function is in another file in the same directory, debug monitors.c.

The implementation of the `kernel_enable_single_step()` is relatively clear, which sets both MDSCR.SS and PSTATE.SS (i.e. SPSR. SS) to 1, allowing the core to return to an active not pending state from an inactive exception.

### 1.3 debug over powerdown

Some readers may have doubts about the topology in Figures 1-4, what is DebugBlock? It is actually a functional unit independent of the CPU cluster, which separates some debug functions from the cluster and is used with DynamIQ to better support debug over powerdown So, what is 'better support' ? What are the issues with the debug architecture in the previous version? This is the main issue discussed in this section.

The introduction to powerdown support has already appeared in the debug section of Arm ARM v7-A. Its meaning is not to debug during core powerdown - which is impossible in any case - but to support debugging of software with power shutdown capability (or software running on an OS with power shutdown capability). The so-called 'support' refers to saving some key debug register content, such as control bits related to the power up and down process or information for external debuggers to recognize. The loss of this information will result in inconsistent debug settings before and after power outage, or may cause external debuggers to lose their connection to the CPU during power outage, thereby interrupting a complete debugging process. Due to the fact that the actions taken by OSPM on power domains are often beyond the control of the debuggers, without powerdown support, it is inevitable that the debugging process will be frequently interrupted by powerdown before the debugging objectives are achieved.

> Not all debug registers require powerdown saving. For information that will not be involved in the power up and down process, relying on the OS Save and Restore process ensures that it will not be lost

Arm ARM v7-A achieves this support by distinguishing between core power domain and debug power domain. The Core domain Vdd and Debug domain Vdd in the following Figure 1-9 illustrate this distinction, and the registers that need to be saved are placed in the Debug power domain. At the same time, since two domains can be separately gated, it is easy to imagine that the core domain can be powered on through debug domain requests. The three bold signals are related to this mechanism.

![Figure 1-9](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/1-9.png)

This solution seems so promising that it was still used in early v8-A, so why did you come up with DebugBlock? Arm has not provided an explanation for this, and I will only make some speculations based on publicly available information. Arm Community has a blog about Juno SoC[^8], and the first part discusses the issue of unstable debugger connections. Juno adopts the configuration of A72/A57+A53, which belongs to what I call the 'early v8-A'. Looking at their TRM, it can be found that they all support independent debug power domains. So why would this problem still occur?

This blog raises a question related to the cpuidle enable in Linaro 14.10. CPU is a concept in OSPM, where the OS places a CPU without tasks into a low-power state, corresponding to hardware power control implementation, which may have different processing methods such as retention/powerdown. Around 2014, cluster idling[^9] was officially merged into power management in Linux. The so-called cluster idling refers to a modification of the arm big.LITTLE topology by traditional CPUs, allowing them to have the opportunity to idle the cluster after all cores within a cluster are idle.

However, for a cluster model consisting of cores, L2/3 cache, and misc.(debug for example), although some logic within the cluster may be individually power gating from a hardware implementation perspective, OSPM may not have a matching granularity in the regulation process. For Juno SoC, I believe that cluster idling may directly cause the debug power domain designed within the cluster to power down as the cluster enters the idle, leading to the previously mentioned issue.

So DebugBlock appeared. It's not much of a change, as the content inside is mostly the same, but completely detached from the cluster in terms of hierarchy As described in A55 TRM:

> It allows you to put the DebugBlock in a separate power domain and place it physically with other CoreSight logic in the SoC, rather than close to the cluster.

However, TRM also mentions that:

> If the DebugBlock is in the same power domain as the core, then debug over powerdown is not supported.

From here, it can be seen that Powerdown support is not necessary. The overall idea of this feature is the set of partitioning power domains in v7-A. When a certain problem arises from this partitioning, a different approach is used to make it possible again.

## 2 Coresight & ADI

Due to the fact that most definitions of Coresight & ADI can be found as corresponding instance references in Coresight SoC, the first step is to provide a background introduction to it. Arm currently offers two versions of the Coresight SoC, namely the 400 series and the 600 series. They represent the implementation of different versions of the Coresight & ADI architecture: SoC-400 implements Coresight v2 and ADIv5.2, while SoC-600 implements Coresight v3 and ADIv6.

| Year | Coresight SoC  | Coresight arch | ADI arch |
| :--- | :------------- | :------------- | :------- |
| 2012 | SoC-400 (r1p0) | v2.0           | v5.2     |
| 2017 | SoC-600        | v3.0           | v6.0     |

The Arm core itself also needs to be compatible with the Coresight & ADI architecture. Unlike SoC-600, which simultaneously updates Coresight v3 and ADIv6, ADIv6's compatibility in Arm core will be later until v9-A. It should be noted that not being compatible with ADIv6 only means that these cores are not compatible with certain new topologies in ADIv6 during design, but SoC-600 is backward compatible and can be designed in conjunction with any Arm core.

| Year | A-series core  | Coresight arch | ADI arch |
| :--- | :------------- | :------------- | :------- |
| 2016 | A73            | v2.0           | v5.2     |
| 2017 | A75/A55        | v3.0           | v5.2     |
| 2018 | A76            | v3.0           | v5.2     |
| 2019 | A77            | v3.0           | v5.2     |
| 2020 | A78            | v3.0           | v5.2     |
| 2021 | A710/A510      | v3.0           | v6.0     |

### 2.1 DAP topology

The so-called DAP topology refers to how DP and one or more APs form a DAP, and how DAP connects to the CPU subsystem. ADIv6 has brought some subtle but potentially significant changes to the DAP topology, and this section mainly focuses on these changes.

Some press releases about the Coresight SoC-600 claim that the release of this suite brings a 'next-generation' debugging solution. Based on the following Figure 2-1, we understand that the so-called 'next generation' debugging refers to the ability to enable debuggers to connect to the debugging system without relying on traditional JTAG/SWD interfaces (reusing functional ports such as PCIe/USB in the figure).

![Figure 2-1](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/2-1.png)

However, if you open the TRM of SoC-600 or ADIv6, you will find that you cannot find any new components designed to adapt to the new ports - the new architecture currently only adjusts the structure of DP/AP, providing system support for debugging without relying on JTAG/SWD debugging ports. Refer to the paragraph in ADIv6:

> ADIv6 permits debug links other than JTAG or SWD to access the AP layer, so that multiple different links can be used to access debug functionality.

The section 'A1.3 The debug link' provides an overview of the improvements to ADIv6. Debug link itself is a new term that refers to the implementation of physical interfaces and protocol layers that are not limited to DP, but more generally. The first sentence of A1.3 mentions that the debug link provides a memory mapped perspective on AP - which is not completely consistent with the design of ADIv5.2. Let's first take a look at the DP to AP access method in ADIv5.2.

How DP accesses an AP essentially depends on the AP's programmer's model, which is the design of the AP register. From the table in the following Figure 2-2, we can see that the AP (i.e. APv1) register in ADIv5.2 is located in an 8-bit wide address space.

![Figure 2-2](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/2-2.png)

ADIv5.2 DP (DPv2) addresses registers in this 8-bit address space by combining the `SELECT.APBANKSEL[7:4]` with the `A[3:2]` of an APACC request. Meanwhile, the high bit `APSEL[31:24]` of the SELECT register is used to select different APs. The following Figure 2-3 provides a schematic diagram of this access mechanism. The highlighted parts in the figure together constitute the address sent by DP.

![Figure 2-3](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/2-3.png)

From the above figure, it can be seen that for a DP + multiple AP system, an APSEL decoder is required as the intermediate layer. This intermediate layer also has an actual IP corresponding to it in the Coresight SoC-400, namely DAPBUS interconnect. The following Figure 2-4 shows its signal connection diagram. The bit width of `dappaddrs[15:2]` on the side of the slave port is exactly equal to the width of `APSEL + APBANKSEL + A`. The bit width of `dappaddrm<x>[7:2]` on one side of the master port is the width of `APBANKSEL + A`.

![Figure 2-4](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/2-4.png)

Next, compare the access methods of ADIv6. At the same chapter location, we see that the registers of APv2 are distributed within a 4KB address space, as shown in the following Figure 2-5. If you continue to scroll down, you will see the Coresight register segment starting from 0xF00. This seems very familiar - in ADIv6, AP is viewed as a debug/core component, which explains the meaning of 'debug link provides a memory mapped perspective on AP' in the previous text. At the same time, in accordance with the discovery principle of coresight components in the coresight architecture, a ROM table parallel to the AP hierarchy needs to be provided and pointed to these APs. You can find this ROM table in Figure 2-1, which does not exist in the ADIv5.2 system.

![Figure 2-5](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/2-5.png)

ADIv6 DP (DPv3) concatenates the ADDR bits of the SELECT and SELECT1 registers to form an address high bit of `[63:4]`, and then combines the `A[3:2]` of the APACC request to form the final address. The figure is omitted here, similar to Figure 2-3. For ADIv6 compatible systems, the DAP topology provided by Arm is DP + APBIC + AP. A significant difference between APBIC and DAPBUS is that it supports up to 4 slave ports, which allows more agent connections on the DP side, allowing other agents to access all APs from the same perspective as external debugging The following figure illustrates the differences between two DAP topologies, where SoC-600 forwards an interconnect access port as another debug agent.

![Figure 2-6](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/2-6.png)

PS: The system using the Coresight SoC-400 also supports forwarding the interconnect, but it needs to be forwarded to the lower level of the AP and cannot access other APs (as well as subsystems connected to other APs) in this DAP, which will look more like Figure 1-4.

### 2.2 power control

In Figures 1-9, you can already see a hint of debugging power control, which is mainly achieved by outputting certain bits of the DP/AP register as power request signals to the power controller In addition to simple power on/off requests, debug power control can also instruct the power controller to enter simulated powerdown, which is a power mode that replaces true power off by turning off the clock or declaring a reset.

Overall, debug power control ensures that debugging work can be completed independently and correctly at the minimum level, without the need for other channels requesting power. As for why it is 'the minimum level', refer to the Juno errata mentioned in the debug over powerdown section. From a hierarchical perspective, there are two levels of debug power control, which are named **DP power control** and **Granular power control** in this article. Among them, DP power control is described in a certain length in the ADI document, which is to some extent an "orthodox" power control mechanism. ADI divides the power domain of the system into three levels here, namely:

- Always on power domain
- System power domain
- Debug power domain

The System/Debug power domain can roughly correspond to the core/Debug power domain in Figures 1-9. The `CTRL/STAT[31:28]` of the DP register represents two pairs of handshake signals: **CxxxPWRUPREQ/ACK**. These two pairs of handshake signals are directly or indirectly connected to the PPU (through SCP) to achieve power control through DP. The ACK signal is returned by the PPU, and each pair of signals follows a basic four phase handshake. As shown in the following Figure 2-7, the debugger needs to combine the states of the REQ/ACK signals to determine the state of the target power domain.

> Due to DP's ability to achieve power control over System/Debug power domains, it can be inferred that DP itself must be in a higher-level power domain - typically placed in AON domain

![Figure 2-7](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2023-08-27-exploring-arm-debug-architecture/2-7.png)

In a multi-core system, the system power domain is further divided, allowing each core's power to be independently turned off or on. In this model, debug power control should also be able to utilize this, which introduces Granular power control Prior to Coresight v3 & ADIv6, this mechanism was implemented in the form of an independent power requestor. In the new architecture, the Granular power control has been integrated into the ROM table, which will be introduced separately below.

### 2.3 secure debug

## Glossary

| Term | Meaning                        |
| :--- | :----------------------------- |
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


## Reference

[^1]: Arm Architecture Reference Manual for A-profile architecture https://developer.arm.com/documentation/ddi0487/ja/?lang=en

[^2]: Arm CoreSight Architecture Specification v3.0 https://developer.arm.com/documentation/ihi0029/f/?lang=en

[^3]: Arm Debug Interface Architecture Specification ADIv6.0 https://developer.arm.com/documentation/ihi0074/d/?lang=en

[^4]: Arm Corstone SSE-710 Subsystem Technical Reference Manual https://developer.arm.com/documentation/102342/0000/?lang=en

[^5]: Arm Cortex-A53 MPCore Processor Technical Reference Manual https://developer.arm.com/documentation/ddi0500/j/?lang=en

[^6]: Juno r2 ARM Development Platform SoC Technical Reference Manual https://developer.arm.com/documentation/ddi0515/f/?lang=en

[^7]: Learn the architecture - Before debugging on Armv8-A https://developer.arm.com/documentation/102408/0100/?lang=en

[^8]: Common issues using DS-5 with Juno https://community.arm.com/oss-platforms/w/docs/544/common-issues-using-ds-5-with-juno

[^9]: Linux Kernel Power Management (PM) Framework for ARM 64-bit Processors https://events.static.linuxfound.org/sites/events/files/slides/lp-linuxcon14.pdf
