---
title: "Security extensions in Arm architecture (1/2): ROP/JOP attacks"
published: 2024-06-26
excerpt: "This article is the first of two articles introducing security features in Arm architecture, analyzing how Arm responds to ROP/JOP attacks at the architecture level."
permalink: /posts/2024/06/security-extensions-in-arm-architecture/ 
---

> This blog is also available in Chinese in https://zhuanlan.zhihu.com/p/699794216

With the rapid penetration of consumer electronics into our lives and the increasing number of devices being connected to networks, computational security has become an issue that cannot be overemphasized. When the Arm v9 architecture was launched three years ago (2021), although it seemed lackluster, please note that security was still one of the three important updates in its promotion (ML, SVE2, and CCA). In addition, although these security patches for hardware architecture are considered to solve the problem at the root compared to software workaround, Arm even had a more radical idea, which is to start from scratch and design a new architecture that integrates security into its bloodline: Morello Program[^1][^2].

We can divide architecture security features into two types based on design intent: security design and security patches. Security design is a top-down methodology that, as a system solution, typically requires architecture level expansion. TrustZone and CCA both belong to this type of security design. And security patches, as the name suggests, are often a remedial measure, especially for attacks that break through security design defenses. Compared to security design, although sometimes involving changes to the instruction set, these patches have a relatively trivial overall impact on the architecture.

This article is the first of two articles introducing security features in Arm architecture, focusing on security patches and analyzing how Arm responds to ROP/JOP and Spectre/Meltdown attacks at the architecture level.

## 1 Mitigate control flow hijacking

ROP/JOP both belong to program control flow hijacking attacks, and building both types of attacks has a certain level of complexity. However, in the early days, control flow hijacking could be achieved directly through simpler buffer overflow. Strictly speaking, stack buffer overflow is just a phenomenon that may be caused unintentionally or intentionally. Only in the latter case is it a means of attack, and in this case it has a more proprietary term, namely stack smashing This attack directs program control flow into malicious code by covering the return address of a function.

![Figure 1-1](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2024-06-26-security-extensions-in-arm-architecture-1-ROPJOP-attacks/Figure_1-1.png)

Due to the widespread application of defense measures related to stack smashing, this type of attack is almost ineffective on modern architectures. A very effective patch made for this hardware architecture is W^X (Writable XOR Rxecutable), which means that a memory address can only have one of the two permissions: write or execute. The so-called 'executable' permission refers to the data in this memory address that can be executed as instructions, that is, this address can appear on the target of branch or return instructions. In this way, the original function return address area will become unable to be written, and any other memory area that can be written will also be unable to perform effective program jumps.

Arm added two control bits, UXN (User Execute never) and PXN (Privileged Execute never), to the memory attribute in the v8-A memory model to implement the functionality of class W. This means that the memory area W^X attribute is controlled on a page by page basis. If a branch instruction attempts to jump to a page where UXN/PXN is set to 1, it will be intercepted and reported as an exception. The consideration for distinguishing between User/Kernel instead of using a unified XN bit here is that if the memory address of user space is also open to the kernel, it will facilitate malicious system call attacks (unrelated to stack smashing). This type of attack can abuse kernel privileges and launch attacks from one application in user space to another.

### 1.1 Return-Oriented Programming

Stack smashing, also known as code injection attack, is a boot program that jumps into malicious code provided by attackers. W^X cuts off this injection pathway, and the ROP/JOP attack mainly discussed in this article evades this permission check by redirecting the bootloader to existing code snippets, hence also known as code reuse attack.

![Figure 1-2](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2024-06-26-security-extensions-in-arm-architecture-1-ROPJOP-attacks/Figure_1-2.png)

Referring to the above figure, the code snippets of gadgets that can be utilized by ROP usually need to have two characteristics: one is simple enough to perform a single function; The second is to end with a return. A single gadget cannot accomplish anything, but when a sufficient number of (different) gadgets are found and carefully designed to connect together, it can almost accomplish any attacker's intent. People who are new to this concept often feel confused. On the one hand, the word 'gadget' is not a frequently used word, and there are some semantic barriers; On the other hand, it is difficult to understand why these seemingly ordinary and legitimate code snippets can be manipulated to achieve the effect of attacking themselves (like immune system diseases).

To understand this, we can make a comparative observation: we mentioned that code injection attacks are accomplished by directing program flow to malicious code. In fact, a lightweight and efficient form of malicious code is to try to give the attacker a shell, which means the attacker gains direct control over the victim and can execute any command. So how does gadget (s) achieve risk equivalence with shell? I will answer this question by extracting a paragraph from the abstract of a widely influential ROP paper[^3]:

> To demonstrate the wide applicability of return-oriented programming, we construct a **Turing-complete** set of building blocks called gadgets using the standard C library from each of two very different architectures: Linux/x86 and Solaris/SPARC.

The descriptive object of 'Turing completeness' is usually a programming language, and a more general definition of it in Wikipedia is' a system of data manipulation rules' A Turing complete 'system' can be used to solve any computable problem. This enables gadgets (s) to achieve a "shell like" functionality, as any shell operation is essentially executing a Turing complete language. This theory is indeed too abstract, and one of my imprecise associations is the infinite monkey theory: it is imaginable to construct a complex operation through sufficient arithmetic operations, logical operations, load/store and other simple basic instructions.

Mitigating ROP attacks can be approached from two perspectives: firstly, it affects its preparation work, that is, the process of finding gadgets and connecting them as needed. The operating system can disrupt this preparation work through Address Space Randomization (ASLR), because if this randomization takes effect, even if the attacker finds the desired gadget, it will be difficult to complete the final connection work because they cannot obtain its address. However, many methods have been developed to bypass ASLR[^4] The second perspective is to perform one by one legality check on instruction jumps, so that even if the attacker completes the preparation work, this planned illegal path will be identified and invalidated.

PAC is the latter scheme, which is essentially a Message Authentication Code that performs integrity checks on information. Supporting PAC will introduce an additional set of instructions, which, in the usual usage paradigm, check whether the return address has been tampered with by signing and verifying at the beginning and end of the called program segment. Refer to the following code from Qualcomm whitepaper[^5], which also provides a pure software solution (stack canary) for comparison.

![Table 1-1](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2024-06-26-security-extensions-in-arm-architecture-1-ROPJOP-attacks/Table_1-1.png)

Stack canary will insert a random number obtained from memory between the function stack frame and the return address, and compare this copy value with the value in memory before returning to check if the return address is overwritten. Since buffer overflow will continuously overwrite subsequent addresses, if an attacker wants to overwrite the return address, they will definitely overwrite this random number at the same time. Although it has some basic leak prevention measures, there are still many ways for attackers to obtain canary values and bypass checks because it is actually in the stack.

As a comparison, PAC first signs LR (Link Register, destination of RET instruction) with a key to generate a small segment of PAC pointer Repeat the signature process before returning, check the two pointers before and after, and attempt to decrypt them to check if LR has been tampered with. In this example, in addition to using key values, the encryption and decryption process also utilizes SP (Stack Pointer) as an additional entropy source. Due to the fact that PAC's keys are stored in more secure (requiring certain permissions) system registers, their security has been strengthened. In addition, it is easy to see that the optimization methods rooted in hardware have clearly brought about better code density.

### 1.2 Jump-Oriented Programming

Due to ROP's heavy reliance on RET instructions for instruction flow scheduling, targeted protection of RET instructions or return addresses can easily mitigate such attacks. The JOP attack that will be introduced next removes the dependency on the RET instruction and instead uses the indirect branch instruction for redirection, thus bypassing the defense measures against ROP.

However, abandoning the RET instruction is not without cost, as it simultaneously abandons the convenience brought by the connection between the attack model and the stack. As mentioned earlier, ROP allows gadgets to relay through links, but this process is not entirely like a train traveling from one station to another (refer to the red line on the right in Figure 1-1), and the central scheduling role played by the stack cannot be ignored (black line on the right in Figure 1-1). All the addresses of the gadgets are placed in some order in the stack space after the victim program overflows, and concatenation is achieved by executing each RET instruction to the address pointed to by the SP and updating the SP's characteristics, which is more like the working process of a semi-automatic rifle.

Branch instructions obviously cannot work together with the stack to complete the loading and reloading action. In order to achieve similar usability as ROP, the authors of the JOP paper[^6] redefined the components of the JOP model: a dispatch table is used to store the gadget address, which is different from ROP in that the dispatch table can be placed at any location in memory rather than limited to the stack; In addition, gadgets are divided into two categories, among which functional gadgets are used to perform basic operations. After the operation is completed, all functional gadgets will jump to a special dispatcher gadget. The dispatcher gadget itself is also a code segment ending in branch, which updates the branch target address every time it is executed to complete continuous instruction scheduling.

Below is a comparison of ROP/JOP models cited in the paper. Through this diagram, we can understand why they are all labeled as "programming": their models are easily abstracted into the form of disassembly code (instruction address instruction), and are accompanied by a virtual "program counter" (SP in ROP and dispatcher in JOP).

![Figure 1-3](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2024-06-26-security-extensions-in-arm-architecture-1-ROPJOP-attacks/Figure_1-3.png)

In ROP defense, we ensure that every RET's destination is legitimate through information integrity checks. Similar checks cannot be performed for the indirect branch instruction, but fortunately, in actual code, only a very small number of code snippets (<2%) are valid indirect branch targets Therefore, the hijacking of such branch instructions by JOP can be alleviated by explicitly marking these legitimate addresses, and both Intel IBT and Arm BTI belong to this defense method. After enabling IBT/BTI (and recompiled by the compiler), a special tag instruction will be inserted before the function entry or branch statement. The next instruction of the branch instruction must point to this tag instruction (must go through the main gate), otherwise an exception will be reported. This greatly increases the difficulty for attackers to organize gadgets, as the basic operations they require often come from a small portion of these complete code snippets (climbing over the wall). The following diagram illustrates the defense mechanism of BTI.

![Figure 1-4](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2024-06-26-security-extensions-in-arm-architecture-1-ROPJOP-attacks/Figure_1-4.png)

## 2 Deatils of PAC
Although this section is called Deatils of PAC, I do not intend to introduce every detail of PAC like a white paper. There are quite a few documents that have already done this. This section is mainly aimed at readers who have a basic understanding of PAC to explore some issues that I have noticed during my learning and development process.

### 2.1 Backward compatibility

PAC/BTI, as a hardware level architecture extension, adds new instructions and system registers. If developers want to enable these new features, they must ensure that the new code is backward compatible, as older devices running the same code do not support these features. Arm achieves this compatibility by placing (some) PAC/BTI instructions into the NOP space in older architectures.

NOP, also known as No operation, is an instruction that exists in all architectures and does not perform any operations. It can be seen as a pipeline bubble. The instruction code of NOP is unique, but there is another HINT instruction code in the Arm architecture that reserves some positions unspecified. For specific architecture versions, some combinations without specified encoding are meaningful, while others are executed according to NOP instructions. Therefore, although older architectures cannot execute PAC/BTI instructions, they can still be decoded normally without generating exceptions. The following diagram illustrates this compatibility encoding.

![Figure 2-1](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2024-06-26-security-extensions-in-arm-architecture-1-ROPJOP-attacks/Figure_2-1.png)

For BTI, this processing is complete. However, for PAC, there is a minor issue that needs to be noted: although PAC has reduced considerable overhead compared to software defense measures, Arm still provides a means to further optimize code density: after enabling PAC, due to the fact that the checksum instruction (AUT *) will be inserted before the jump instruction (RET/BL), the two are adjacent to each other, a special instruction is provided to fuse the two. The leftmost and rightmost columns in the table provide an example of RETAA=AUTIASP+RET.

![Table 2-1](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2024-06-26-security-extensions-in-arm-architecture-1-ROPJOP-attacks/Table_2-1.png)

However, due to the fact that the combined PAC instruction will swallow the original RET instruction, it cannot be backward compatible through the NOP space mentioned above. Arm determines whether to generate such instructions by providing different compilation options, and developers should choose based on the actual situation of the target platform.

### 2.2 PACGA

PACGA is a special member of the PAC instruction family, which is a "one-way operation" that does not have a corresponding checksum instruction (such as AUTGA) to match it. Its execution method is also different from other PACxx instructions: other PACxx instructions fill the key on the basis of the encrypted object. Essentially, the source register (one of them) and the destination register are the same, and the key can only be filled into unused positions. Therefore, the key length is often very limited. When MTE is enabled and VA range=48bits, only 7 bits can be used.

However, in reality, regardless of the final key length used, the operation process of PAC instructions is the same. The QARMA algorithm always generates a 64 bit result internally, which the kernel intercepts according to actual needs. On the PACGA side, it will use the entire upper 32 bits of the result as a key to assign to another register outside of the encrypted object.

Why design this additional instruction? As mentioned earlier, the process of PAC encryption and decryption essentially belongs to an information integrity check: the information sender generates an information verification code by processing the information and sends it together with the information itself; After receiving the information, the information receiver performs the same processing as the sender, generates another information verification code, and compares it with the sender's to verify whether the information has been damaged during transmission.

When it comes to this more general definition, it is easy to think of CRC (cyclic redundancy check), a more well-known inspection method. In fact, PACGA provides developers with the ability to perform information integrity checks in a broader sense, similar to CRC. However, in terms of algorithm selection, it happens to be a free rider on QARMA engine. Therefore, it can be imagined that PACGA will be actively called and executed by developers according to their own needs, while other PACxx/AUTxx instructions will be directly inserted into the code by the compiler.

### 2.3 Performance

Qualcomm and Arm jointly developed the QARMA (Qualcomm ARM Authenticator) algorithm for PAC. I have no research on cryptography itself, but from a hardware implementation perspective, QARMA is not difficult. You don't even need to look at any information, just follow the relevant pseudocode in Arm ARM to create it. Therefore, what this section wants to discuss is not the implementation method, but the performance of the implementation.

The algorithm author provides delay & area data for different QARMA variants under 7nm FinFet in this paper[^7]. Due to the use of QARMA5 in the first batch of Arm v9.0 cores (A710/A510), we mainly focus on the data of QARMA5. Different σ represents the difference in a type of parameter (S-Box) in the algorithm. In fact, v9.0 cores chose σ 2 From the perspective of Minimum Delay, a larger σ will result in poorer timing, but even if we choose the optimal set of 2.2ns, this timing is already explosive for A710/510.

![Table 2-2](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2024-06-26-security-extensions-in-arm-architecture-1-ROPJOP-attacks/Table_2-2.png)

This is because, according to the Software Optimization Guide of A710/A510, it can be found that both have a execution delay of 5 cycles for non combined PAC instructions Therefore, there is no need to talk about A710 that can run up to 3GHz. Even for A510 that targets 2GHz, considering the over estimation of timing in physical implementation and the fact that PAC operation cannot fully occupy 5 cycles (the last cycle still needs to go through the forwarding path), the time left for PAC operation will not exceed 2ns, which cannot even meet the optimal timing in the paper.

![Figure 2-2](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2024-06-26-security-extensions-in-arm-architecture-1-ROPJOP-attacks/Figure_2-2.png)

The publication time of the above paper is 2016, while the publication time of A710/510 is 2021, and their typical PPA is also given under TSMC 7FF. If it weren't for the author's incorrect data, it can only be said that the progress in technology is truly amazing.

There is actually another question here. Regarding the v9.2 cores (A720/A520) released last year, I noticed that their PAC algorithm has been changed from QARMA5 to QARMA3, which means that the encryption operation has been shortened from 5 loops to 3 loops. In theory, the execution delay of their PAC instructions should be correspondingly reduced. However, upon reviewing the corresponding Software Optimization Guide, I found that both cores remained unchanged for 5 cycles.

## 3 PACMAN

Each version update of the Arm architecture (v8.x/v9. x) is actually composed of several independent features. For example, PAC first appeared in the form of FEAT-PAuth in v8.3, while BTI corresponds to FEAT_STI in v8.5 However, Arm also added many PAC related features in the subsequent architecture of v8.3, such as FEAT-PAuth2 and FEAT-FPAC in v8.6 Among them, FEAT_SPAC is related to the discussion in this section. It is a patch released by Arm specifically for fixing PACMAN vulnerabilities - PAC itself is a patch for ROP attacks, so this can be considered as a patch for patches.

PACMAN comes from a paper by the MIT team at ISCA'22[^8], which focuses on platforms that have already deployed PAC. By finding a gadget similar to that used in Spectre attacks, they create a special scenario where PAC authentication+load/store use that result is executed on the speculative path, and brute force (brute force) is performed on the attacker's unknown correct PAC pointer. The author validated this attack on the Apple M1 SoC. However, it can be seen that PACMAN attacks (like ROP) rely on finding gadgets that meet the attacker's needs, rather than being launched casually. Unfortunately, the author points out that such gadgets are widely present in practice (at least in the MacOS they tested).

PACMAN exploited a vulnerability in early PAC. Before the introduction of FEAT_SPAC, if an AUTxx verification was successful, there was no problem. The kernel would replace the original location where the PAC pointer was stored with extension bits (VA [55]), which is no different from now; But if the verification fails, this AUTxx instruction will not immediately report an exception, nor will it replace the original location where the pac pointer was stored with VA [55] - this will cause any behavior using that address to have a translation fault That is to say, PAC faults will propagate stealthily until they are checked by other instructions. Under normal circumstances, this is not a problem. To understand how PACMAN utilizes this, it is also necessary to introduce speculative execution in modern processors.

Speculative execution exists in almost all modern processors (whether in order or out of order), as the judgment condition of the execution unit may be pending at the moment when encountering branch instructions. The execution unit always speculates on executing the modified instruction based on the predicted result of the branch, causing the instruction flow to continue moving forward (in the direction of speculation) instead of pausing and waiting for the judgment condition to be resolved Before determining the condition resolve, the instruction flow after the branch instruction is on the so-called speculative path. Compared to completely pausing for judging conditions, if it is ultimately proven that the speculated direction is correct, this time difference is earned, resulting in performance improvement; If the speculative direction is incorrect, clear all speculative execution instructions and restart execution from the correct direction, which is no different from completely pausing in performance.

Due to the requirement of modern processors for instruction streams to retire sequentially, instructions on the speculative path do not retire until the conditional resolve is determined. This means that speculative executed instructions do not modify the architecture state (GPR or system registers). However, readers familiar with Spectre will know that they may modify the micro architecture state (cache or TLB), and even if speculative execution is proven to be incorrect, these modifications are irreversible.

The above information can form a key inference: if an instruction on a speculative path that is ultimately proven to be incorrect generates an exception, the instruction flow will not be interrupted by this exception. This is a prerequisite for PACMAN to perform brute force on PAC pointers while the kernel is running normally. From this, we now know that PACMAN attack is a process of guessing passwords>guessing incorrectly>guessing again until we guess correctly.

However, although every 'wrong guess' becomes costless due to being cleared, on the other hand, even' right guess' will have the same outcome. So how do we know the information of 'guessing correctly'? The answer is to search for traces of modifying the micro architecture state during the guessing process. The design intention of these micro architectures is to improve performance, and developers are not expected to be able to access their content (at least in user mode), so they cannot be directly accessed. However, the time spent accessing different memory hierarchies cannot be concealed, and there are already many mature methods to infer micro architecture states in reverse based on this PACMAN found the correct guess through several attempts using this method.

The following figure shows two types of PACMAN gadgets, data/structure, in their simplest form, and their respective process of performing a PAC pointer guess. Taking the data gadget as an example, the attacker first trains the branch predictor to always judge that it will execute on the BR1 branch. During the execution of the attack, the value of cond is temporarily changed and its cache is cleared. As a result, the instruction stream enters the Mispredict Window due to speculation execution while the kernel fetch cond Then execute the AUT instruction once in this window and use its result for a load In the vast majority of cases, this load will be unable to proceed due to the use of an illegal VA (reporting an exception but being suppressed); But for the exact guess, the load will execute smoothly and modify the cache and TLB (Specifically, PACMAN uses Prime+Probe to perform side channel attacks on TLB, which is not covered in this article)

![Figure 3-1](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2024-06-26-security-extensions-in-arm-architecture-1-ROPJOP-attacks/Figure_3-1.png)

Looking back, how did FEAT_SPAC solve this problem? The content of FEAT_SPAC itself only prompts an erroneous AUT to immediately report an exception. However, looking back at the structure of PACMAN gadgets, if that's the case, the effect seems to only add an exception source to the process of incorrect guessing, and the result will still be distinguishable from correct guessing because the load command cannot be issued. Following this line of thought, we found that the most crucial point is to make it difficult to distinguish between errors and correct guesses.

Arm introduced the recommended usage of FEA TFPAC on this webpage[^9]. Referring to the figure below, Arm actually expects to implement a platform that adds an XPAC instruction to each AUT instruction for FEA TFPAC. The function of the XPAC instruction is to directly clear the target's pac pointer (replaced with extension bits) without any validation. So, since AUT exceptions on the speculative path do not block subsequent instruction execution, even a single erroneous guess can obtain a valid address after XPAC and smoothly issue the load instruction, thereby eliminating the difference between incorrect and correct guesses.

![Figure 3-2](https://raw.githubusercontent.com/srleslie/srleslie.github.io/master/_posts/assets/2024-06-26-security-extensions-in-arm-architecture-1-ROPJOP-attacks/Figure_3-2.jpg)

The reason why this approach cannot be adopted on platforms that do not implement FEA TFPAC is that, obviously, by always clearing all PAC pointers, it will bypass the old version's checking mechanism and render PAC completely ineffective. FEAT_SPAC ensures that AUT instructions on non speculative paths have the ability to interrupt the execution of instruction streams in the event of a PAC pointer mismatch.


## Reference

[^1]: https://www.arm.com/zh-TW/architecture/cpu/morello

[^2]: https://newsroom.arm.com/blog/morello

[^3]: https://hovav.net/ucsd/papers/rbss12.html

[^4]: https://people.eecs.berkeley.edu/~daw/papers/rop-usenix14.pdf

[^5]: https://www.qualcomm.com/content/dam/qcomm-martech/dm-assets/documents/pointer-auth-v7.pdf

[^6]: https://www.comp.nus.edu.sg/~liangzk/papers/asiaccs11.pdf

[^7]: https://eprint.iacr.org/2016/444.pdf

[^8]: https://ieeexplore.ieee.org/document/10120652

[^9]: https://developer.arm.com/documentation/ka005109/latest/
