--- 
layout: post
title: Interpreter Implementation Choices
---
Overview
========

Interpreters are a popular choice for execution engines in many virtual machine (VM) architectures. Interpreters are popular because they are;

-   portable: easy to port to new architectures.
-   simple: a small easy to understandable code base that makes language and VM evolution tractable.
-   quick to start executing code.

The disadvantage of interpreters is that they have relatively poor execution speed when compared to the equivalent natively compiled code. The performance challenge is usually a result of the low work-to-overhead ratio and poor branch prediction accuracy.

This article uses byte code interpreters as an example and describe different techniques for implementing an interpreter and the impact on execution speed, portability, complexity and start up time.

Representation
--------------

Different breeds of interpreter represent instructions differently. Below is several alternative representations of a statement. Interpreters typically walk a tree or iterate over a byte code representation, performing work at each node or instruction as appropriate. (Some interpreters use a series of rewriting rules but these are not considered in this article).

<div style="border: 1px solid black; padding: 10px;">
<table style="width: 100%; padding: 5px; margin: 5px">
<tr style="vertical-align: top">
<td style="width: 33%">
    (1)
        push #1
        push #2
        add
        store @var


</td>
<td style="width: 33%">
    (2)
        move r1, #1
        move r2, #2
        add r3, r1, r2
        move @var, r3


</td>
<td style="width: 33%">
    (3)
          store
            /\
           /  \
        @var  add
              /\
             /  \
            1    2

</td>
</tr>
</table>
<div style="margin-right: 10%; margin-left: 10%;">
Different representations of the statement <code>@var = 1 + 2</code>;
(1) stack-based byte code representation,
(2) register-based byte code representation, and
(3) tree-based representation.

</div>
</div>
Byte code seems to be the most popular representation form. Byte code is compact. Byte code is easy to decode. Byte code is trivial to cache to avoid re-parsing the source language. (Parsing is a relatively expensive operation). Byte code makes it possible to evolve the language syntax without overhauling the underlying instruction set architecture (ISA) too dramatically. Some VMs have a generic ISA that makes it possible for other languages to target the VM with relative ease (assuming that the VM supports a compatible computation model). Given the popularity of the byte code representation form, the remainder of this article will assume this representation.

Most interpreter implementations implement each instruction using a similar pattern. The prologue fetches the arguments from the data stack, registers and/or the environment. The prologue may also prefetch or otherwise prepare for the dispatch following the execution of the instruction. Following the execution of the instruction, results are stored back into the data stack, registers and/or the environment and a fetch+decode+dispatch to the next instruction occurs. The prologue and epilogue constitute the interpreter overhead.

<div style="border: 1px solid black; padding: 10px;">
    /* Start prologue */
    I_add:
    {
      int a = *(data_sp++); //retrieve first operand
      int b = *(data_sp++); //retrieve second operand
      int c;
    /* End prologue */

    /* Start instruction work */
      c = a + b;
    /* End instruction work */

    /* Start epilogue */
      *(data_sp--) = c; //store result
    }
    goto **(ip++); //dispatch to next instruction
    /* End epilogue */

<div style="margin-right: 10%; margin-left: 10%;">
Example handler for an ‘add’ instruction in a stack-based ISA.

</div>
</div>
Challenges
==========

Work-to-overhead Ratio
----------------------

Each instruction in the VM’s ISA results in native code to perform the actual work and overhead native code to traverse the instruction representation. i.e. updating the instruction counter, dispatching to the next instruction etc. The ratio varies depending on design of the ISA. i.e. Is the ISA RISC-like or CISC-like? Is the ISA designed so that the interpreter spends most of it’s time in libraries?

The ratio of useful work performed compared to the overhead of traversing the representation is often low for general purpose interpreters. It would not be unexpected for one VM instruction to translate into 1 or 2 native instructions of useful work and 10 or more native instructions to dispatch to the next VM instruction. This is even more disastrous when you consider that the overhead typically includes one or more difficult to predict branches that stall the instruction pipeline on most modern hardware.

Branch Prediction Accuracy
--------------------------

On modern heavily pipelined hardware there is a heavy cost for mis-predicting a branch. The instruction pipeline is typically flushed and execution is stalled. As a consequence most modern hardware uses some form of branch prediction. Interpreters tend to make heavy use of indirect branches during instruction dispatch and thus the accuracy of indirect branch prediction can significantly impact an interpreters performance. Popular commercially available processors tend to use a Branch Target Buffer (BTB) for indirect branch prediction “\[ERTL01\]”:\#ERTL01. The BTB caches the last target of a branch instruction. The architecture predicts that the next execution of the branch has the same target. With interpreters this assumption rarely holds.

Techniques
==========

Dispatch Strategy
-----------------

The instruction stream is typically made up of a sequence of VM instructions and immediate values (i.e. the parameters for the instruction). The dispatch or “threading” strategy determines how the execution flows from one VM instruction to the next. There are a number of different strategies; direct threading, indirect threading, token threaded, switched dispatch and replicated switched dispatch.

### Direct Threaded Code

Direct threaded code (DTC) “\[BELL73\]”:\#BELL73 encodes each instruction as the address of the code that performs the instruction. Thus flowing from one instruction to the next involves a jump to a value retrieved from the instruction stream. DTC is typically considered one of the fastest dispatch strategies but it does come at the cost of making each instruction the size of an address.

<div style="border: 1px solid black; padding: 10px;">
    void *program[] = {&&I_const_1, &&I_const_1, &&I_add, ...}
    void **ip = program;

    goto **ip++;

    I_const_1: { ... /* Code to handle 'const_1' */ ... } goto **ip++;
    I_add:     { ... /* Code to handle 'add' */ ... }     goto **ip++;
    ...
    }

<div style="margin-right: 10%; margin-left: 10%;">
Stack based ISA implemented using DTC dispatch. The program performs 1 + 1 and places the result on the stack.

</div>
</div>
### Indirect Threaded Code

Indirect threaded code (ITC) “\[DEWAR75\]”:\#DEWAR75 encodes each instruction as the address of a cell in a lookup table. ITC adds one more level of indirection to DTC and thus typically results in more overhead during dispatch. ITC does have the advantage that the execution engine can be modified by changing the look up table without modifying all the generated code. This would make it possible to switch between normal execution, profile gathering execution or debug execution modes by simply overwriting the lookup table.

<div style="border: 1px solid black; padding: 10px;">
    static void *lut[] = {&&I_const_1, &&I_add, ...};

    ...

    void * program[] = {lut, lut, lut + 1, ...}
    void **ip = program;

    goto ***(ip++);

    I_const_1: { ... /* Code to handle 'const_1' */ ... } goto ***(ip++);
    I_add:     { ... /* Code to handle 'add' */ ... }     goto ***(ip++);
    ...
    }

<div style="margin-right: 10%; margin-left: 10%;">
Stack based ISA implemented using ITC dispatch. The program performs 1 + 1 and places the result on the stack.

</div>
</div>
### Token Threaded Code

Token threaded code (TTC) encodes each instruction as a byte that indicates the offset of a handler in a lookup table. Token threaded code is a much more compact representation than the DTC or ITC strategies. Like ITC, TTC can support multiple execution modes by modifying the lookup table.

<div style="border: 1px solid black; padding: 10px;">
    typedef enum 
    {
      const_1, add, ...
    } instruction_t;

    ...

    static void *lut[] = {&&I_const_1, &&I_add, ...};

    ...

    instruction_t program[] = {const_1, const_1, add, ...}
    instruction_t *ip = program;

    goto *lut[*ip++];

    I_const_1: { ... /* Code to handle 'const_1' */ ... } goto *lut[*ip++];
    I_add:     { ... /* Code to handle 'add' */ ... }     goto *lut[*ip++];
    ...
    }

<div style="margin-right: 10%; margin-left: 10%;">
Stack based ISA implemented using TTC dispatch. The program performs 1 + 1 and places the result on the stack.

</div>
</div>
### Switch-based Dispatching

Switch-based dispatching (SBD) represents each instruction as a byte. There is a single loop in which a switch statement is used to dispatch to the handler for each instruction. Like TTC, the representation is compact but the most significant advantage is that it can be implemented in almost any programming language. The previously described dispatch strategies require special support from the compiler (i.e. GCC’s “Labels as Values” extension “\[GCC\]”:\#GCC) or a small amount of platform specific assembly code.

<div style="border: 1px solid black; padding: 10px;">
    typedef enum 
    {
      const_1, add, ...
    } instruction_t;

    ...

    instruction_t program[] = {const_1, const_1, add, ...}
    instruction_t *ip = program;

    while( 1 )
    {
      switch (*ip++) 
      {
        case const_1: { ... /* Code to handle 'const_1' */ ... } break;
        case add:     { ... /* Code to handle 'add' */ ... }     break;
        ...
      }
    }

<div style="margin-right: 10%; margin-left: 10%;">
Stack based ISA implemented using SBD dispatch. The program performs 1 + 1 and places the result on the stack.

</div>
</div>
### Replicated Switch-based Dispatching

Replicated switch-based dispatching (R-SBD) is a variant of SBD that attempts to minimize indirect branch mispredictions. SBD uses a single indirect branch through which all instructions pass but R-SBD replicates the switch statement for each instruction (and includes an unconditional jump in the switch statement to instruction handling code). SBD results in a near 100% misprediction rate using the BTB where as R-SBD can result in a lower 50% mis-prediction rate “\[ERTL01\]”:\#ERTL01.

<div style="border: 1px solid black; padding: 10px;">
    typedef enum 
    {
      const_1, add, ...
    } instruction_t;

    ...

    instruction_t program[] = {const_1, const_1, add, ...}
    instruction_t *ip = program;

    switch (*ip++) 
    {
      case const_1: goto I_const_1;
      case add: goto I_add;
      ...
    }

    I_const_1: 
    { ... /* Code to handle 'const_1' */ ... }
    switch (*ip++) 
    {
      case const_1: goto I_const_1;
      case add: goto I_add;
      ...
    }
    I_add: 
    { ... /* Code to handle 'add' */ ... }
    switch (*ip++) 
    {
      case const_1: goto I_const_1;
      case add: goto I_add;
      ...
    }
    ...

<div style="margin-right: 10%; margin-left: 10%;">
Stack based ISA implemented using R-SBD dispatch. The program performs 1 + 1 and places the result on the stack.

</div>
</div>
### Summary

The performance of different dispatch methods varies between hardware configurations and architectures, VM ISA designs, execution workloads and other interpreter design decisions. For example, the low memory pressure of SBD, may result in an interpreter that outperforms DTC on certain workloads or if the system has small caches. The only real way to determine the most appropriate strategy is to measure the performance in realistic scenarios. “\[ERTL07\]”:\#ERTL07 measured the performance of different dispatch strategies across a range of architectures and compared it to the standard “subroutine” dispatch strategy (i.e. a normal function calls rather than interpretation). Somewhat expectantly, DTC outperformed “subroutine” dispatch strategy in most scenarios. This just goes to show that experimental investigation can yield surprising non-intuitive results.

Inlined Instructions
--------------------

Inlining instructions is a technique for eliminating the dispatch overhead associated with an instruction. The interpreter identifies the native instruction sequence for handling a particular VM instruction. The native instruction sequence is then copied into a new handler. The new handler may actually contain the native instruction sequences for several VM instructions and the control flow can flow from one VM instruction to the next without any dispatch overhead. Taken to the extreme, where the sequence of VM instructions that make up each “function” is copied into a single handler, and the interpreter becomes a poor mans JIT compiler.

Inlining instructions is not without it’s problems. Typically some form of cache is used to avoid regenerating the inlined sequences multiple times. This approach can dramatically increase the memory pressure of the interpreter as more VM instructions are concatenated into handlers. If a function or method is not invoked multiple times then the performance benefit of eliminating the dispatch is typically offset by the cost of generating the inlined sequences and caching the result.

Inlining instructions has several portability problems. Firstly, compiler specific extensions are typically needed to identify the start and end of instruction handler code (i.e. GCCs “Labels as Values” extension “\[GCC\]”:\#GCC). Secondly, not all code is position independent code (PIC) and thus not all code can be safely inlined. Code that is compiled with relative calls or branches that have targets outside the sequence being copied will not work as expected.

Depending on the compiler, compiler configuration options and hardware architecture, relative addresses can be used when making c function calls. The C function call may have been explicitly inserted by the programmer as part of the implementation of the instruction handler or they may be implicitly generated by the compiler. For example, the GCC compiler may implement the “divide by a long” operation as an hidden internal c function call on some platforms (i.e. x86).

Another problem arises when the interpreter is compiled at different levels of optimisation. Conditional code in a handler such as the C code <code> if( … ) { … } </code> may generate relative jumps to the end of the handler at low levels of optimization. At higher levels of optimization the jump becomes a relative jump into the dispatch code. As a result this code can be inlined at low levels of optimization but will fail at higher levels of optimization.

Inlining code offers a speed advantage but requires an in depth knowledge of compiler implementation details to identify which sequences compile down to PIC. As the compilers methods for generating code can change between releases this ultimately results in a far less portable product and portability is one of the advantages of writing interpreters in the first place. Several projects that have used this approach (i.e. QEMU) eventually begin to move away from this design.

<div style="border: 1px solid black; padding: 10px;">
    /* Instruction Implementations */
    I_const_1_start: 
    {
      *(data_sp++) = 1;
    }
    I_const_1_end: 
    goto **(ip++);

    I_add_start: 
    {
      int a = *(data_sp++);
      int b = *(data_sp++);
      int c;
      c = a + b;
      *(--data_sp) = c;
    }
    I_add_end: 
    goto **(ip++);

    ...

    /* Create an inline sequence of const_1, const_1, add */
    size_t const_1_size = &&I_const_1_end - &&I_const_1_start; 
    size_t add_size = &&I_add_end - &&I_add_start; 

    void *sequence = malloc(const_1_size + const_1_size + add_size); 
    memcpy(sequence + 0, &&I_const_1_start, const_1_size); 
    memcpy(sequence + const_1_size, &&I_const_1_start, const_1_size); 
    memcpy(sequence + const_1_size + const_1_size, &&I_add_start, add_size); 
    ... 

    /* Jump to the start of inlined sequence to start execution */ 
    goto **sequence; 

<div style="margin-right: 10%; margin-left: 10%;">
Example of creating an inline sequence of const\_1, const\_1, add instructions.

</div>
</div>
Super Instructions
------------------

Combining several VM instructions into superinstructions or “macro” instructions is a technique that has been used to reduce code size, reduce the overhead of dispatch and argument access “\[PRO95\]”:\#PRO95,“\[HOOG99\]”:\#HOOG99,“\[PIU98\]”:\#PIU98,“\[GREGG03\]”:\#GREGG03. The superinstruction replaces the sequence of the component instructions either when the byte code is loaded or after the initial parsing of the source language.

<div style="border: 1px solid black; padding: 10px;">
<table style="width: 100%; padding: 5px; margin: 5px">
<tr style="vertical-align: top;">
<td style="width: 25%">
    (1)
        load @var
        push #1
        add
        store @var

</td>
<td style="width: 25%">
    (2)
        load @var
        push_1
        add
        store @var

</td>
<td style="width: 25%">
    (3)
        load @var
        inc
        store @var

</td>
<td style="width: 25%">
    (4)
        inc @var


</td>
</tr>
</table>
<div style="margin-right: 10%; margin-left: 10%;">
Different instruction sequences to increment a variable:

(1) using primitive operations,
(2) using a superinstruction to push a <code>1</code> onto the stack,
(3) using a superinstruction to increment the value on the top of the stack, or
(4) using a superinstruction to increment the value of a variable.

</div>
</div>
The figure above presents several sequences of instructions for incrementing the value of a variable. Each successive version uses superinstructions that take on more responsibility. In (2) the <code>push \#1</code> sequence (i.e. instruction byte plus a literal integer value) becomes <code>push\_1</code>, a single instruction byte. In (3) the <code>push\_1; add</code> sequence becomes <code>inc</code>. In (4) the entire sequence is reduced to one superinstruction.

With each successive version the size of the VM code has decreased. After (2) the number of pushes and pops onto the stack decrease, reducing the overhead associated with passing parameters to instructions. After (2) the amount of useful work per VM instruction has increased and thus the proportion of dispatch overhead is decreased.

The tradeoff is that as superinstructions become more specific, they are less likely to be used. An implementer needs to strike a balance between the work achieved by the instruction and usefulness of the superinstruction.

It should be noted that there are also constraints on the type of instructions that can be combined into a superinstruction. Most approaches disallow instructions that alter the control flow from being merged into a superinstruction unless it is the last VM instruction in a the superinstruction. Instruction sequences that cross a basic block boundary are typically not candidates for merging as it can make it difficult to handle a jump to the start of the second basic block.

The code to handle the superinstruction can either be generated at build time or at run—time. The runtime or dynamic approach uses instruction inlining with all the associated costs and benefits.

Generating the superinstructions at build time means that the interpreter needs to know which instructions are good candidates for superinstructions ahead of time. This is not typically a problem as profiling and common sense can often suggest which instructions make good candidates for superinstructions. Interpreter generation frameworks such as vmgen “\[GREGG03\]”:\#GREGG03 provide built—in profiling to identify candidate superinstructions.

### External or Internal?

The other major consideration for superinstructions is whether they are exposed outside the interpreter. Of course this does not make any sense if the interpreter does not define the ISA or byte code format as part of the external specification.

Internal superinstructions are not part of the external specification. Internal superinstructions replace normal instructions as the byte code is loaded into the virtual machine. They have the advantage that they can evolve faster and change to meet changing workloads or architectures and the implementation is not constrained by the initial design of the VM ISA.

The Java VM instruction set includes several superinstructions as part of the external specification. The 2 byte instruction <code>ldc + \[constant pool value\]</code> loads a constant value onto the stack. The Java specification also supports the 1 byte instruction <code>iconst\_1</code> that puts the constant 1 on the stack. The second form is likely used because it is a frequently occurring operation in a java program and it decreases the size of the code format.

Of course there is nothing to stop the interpreter from supporting a set of external superinstructions and another set of internal superinstructions. In this fashion the VM can gain the advantage of increased flexibility of internal superinstructions where needed as well as the reduced code size that comes with external superinstructions.

### De-specializing

De-specializing a superinstruction replaces a superinstruction with it’s component VM instructions. This may be desirable when there is a need to reduce the number of distinct instructions. Typically the specialization or superinstructions that are de-specialized were originally specialized to reduce code size, not necessarily to improve performance. De-specializing an instruction can make the introduction of other superinstruction sequences more performance effective “\[CASEY07\]”:\#CASEY07. The CellVM, a JVM for the *Cell Broadband Engine* architecture de-specializes the load/store VM instructions for this reason “\[WIL08\]”:\#WIL08.

Instruction Replication
-----------------------

Replicating the code to handle a VM instruction multiple times, has been proposed as a mechanism for improving BTB prediction accuracy “\[CASEY07\]”:\#CASEY07. Each replica uses a separate indirect branch to dispatch to the next instruction. Therefore each replica has a separate BTB entry. Each instruction instance attempts to make use of a different replica.

Without replication, if an instruction appeared multiple times within a loop the BTB mis-prediction rate is close to 100%. Using a separate replica for each instruction instance can dramatically improve the accuracy of branch prediction. There will still be some mis-predictions if the number of replicas in a loop is less than the number of instruction instances.

Increasing the number of replicas increases the size of the interpreter. This can adversely impact the time it takes for the native compiler to build and optimize the code. In some cases the resulting code growth may also cause performance problems due to increased memory pressure. However, at least for some domains the benefit of increased BTB prediction accuracy out weighs any performance degradations due to the increased size of the generated code “\[CASEY07\]”:\#CASEY07.

While I am unaware of any published results, it is expected that an appropriate profiler can identify which particular instructions could benefit from replication and suggest suitable number of replicas. An enhanced version of vmgen “\[GREGG03\]”:\#GREGG03 (or other interpreter generator) could feedback profiling data into the build process much like it feeds back suitable superinstructions.

Replicas are typically generated statically at build time but some research has looked at replicating instructions dynamically at runtime. The dynamic approach uses instruction inlining with all the associated costs and benefits.

Manual Branch Prediction
------------------------

Some modern processors make it possible for the software developer to influence the branch prediction logic on the processor, thus limiting or eliminating mispredictions. Eliminate the mis-prediction and several other techniques become obsolete. R-SBD no longer has an advantage over SBD. Replicating instructions no longer improves performance and may cause degradation due to code bloat.

Some modern ISA’s (i.e. IA64 and Power/PPC) use split branches to try and reduce the mis-prediction penalty. There is a separate *prepare-to-branch* instruction and a *branch* instruction. As long as the software schedules a *prepare-to-branch* instruction prior to the *branch* then the indirect branches can be executed without a stall. In interpreters that have a low work-to-overhead ratio, it may be necessary to schedule a *prepare-to-branch* instruction in instruction N-1 with target of N+1 so that it is available in instruction N. The IBM POWER3 processor implements this with a special branch target register that can be set to target to avoid a misprediction penalty “\[OGATA02\]”:\#OGATA02.The branch hints feature of the Cell processor is a similar strategy, it allows the program to explicitly set BTB entry “\[WIL08\]”:\#WIL08.

Summary
=======

Interpreters are are used because they are simple to implement, portable across platforms and quick to start up but the performance does not compare with native code. The article outlines several different trade offs that can be made while building an interpreter to improve the execution speed at the cost of decreasing the portability and simplicity of the interpreter or increasing the start up time of executed code. The described techniques include; Dispatch Strategies, Inlined Instructions, Super Instructions, Instruction Replication and Manual Branch Prediction. There are other techniques that can significantly impact the interpreter implementation such as stack caching and aligned access vs non-aligned access that have not been explored.

The actual costs and benefit of any particular design choices for a particular interpreter will depend dramatically on the language being interpreted, the expected workload and the hardware environment. In a perfect world an interpreter generator, such as vmgen “\[GREGG03\]”:\#GREGG03, would take a virtual machine specification, several sample programs from which to determine the workload and would be able to build an interpreter that was optimized for the host environment, workload and language. The utility of such an approach may not be high but it would be a fun are to explore.

Bibliography
============

-   <a name="BELL73">BELL73</a>: J. R. Bell. *“Threaded code”*. Communications of the ACM, 16(6):370?372, 1973.
-   <a name="CASEY07">CASEY07</a>: K. Casey, M. A. Ertl, and D. Gregg. *“Optimizing indirect branch prediction accuracy in virtual machine interpreters”*. ACM Transactions on Programming Languages and Systems (TOPLAS), 29(6):37, 2007.
-   <a name="DEWAR75">DEWAR75</a>: R. B. K. Dewar. *“Indirect threaded code”*. Communications of the ACM, 18(6):330?331, 1975.
-   <a name="ERTL01">ERTL01</a>: M. Anton Ertl and David Gregg. *“The Structure and Performance of Efficient Interpreters”*, Journal of Instruction-Level Parallelism, Vol. 5, November, 2003.
-   <a name="ERTL07">ERTL07</a>: M. Anton Ertl. *“Speed of various interpreter dispatch techniques v2”*, 2007. <http://www.complang.tuwien.ac.at/forth/threading/>.
-   <a name="GCC">GCC</a>: *“Labels As Values”*, 2008. <http://gcc.gnu.org/onlinedocs/gcc/Labels-as-Values.html>
-   <a name="GREGG03">GREGG03</a>: D. Gregg and M. A. Ertl. *“A language and tool for generating efficient virtual machine interpreters”*. In C. Lengauer, D. S. Batory, C. Consel, and M. Odersky, editors, Domain-Speci?c Program Generation, volume 3016 of Lecture Notes in Computer Science, pages 196?215. Springer, 2003.
-   <a name="HOOG99">HOOG99</a>: J. Hoogerbrugge, L. Augusteijn, J. Trum, and R. V. D. Wiel. *“A code compression system based on pipelined interpreters”*. Softwware - Practice and Experience, 29(11):1005?2023, 1999.
-   <a name="OGATA02">OGATA02</a>: K. Ogata, H. Komatsu, and T. Nakatani. *“Bytecode Fetch Optimization for a Java Interpreter”*. In ASPLOS-X: Proceedings of the 10th international conference on Architectural support for programming languages and operating systems, pages 58?67, New York, NY, USA, 2002.
-   <a name="PIU98">PIU98</a>: I. Piumarta and F. Riccardi. *“Optimizing direct threaded code by selective inlining”*. SIGPLAN Notices, 33(5):291?300, 1998.
-   <a name="PRO95">PRO95</a>: T. A. Proebsting. *“Optimizing an ansi c interpreter with superoperators”*. In POPL ?95: Proceedings of the 22nd ACM SIGPLAN-SIGACT symposium on Principles of programming languages, pages 322?332, New York, NY, USA, 1995.
-   <a name="WIL08">WIL08</a>: K. Williams, A. Noll, A. Gal and D. Gregg. *“Optimization strategies for a java virtual machine interpreter on the cell broadband engine”*. In CF ’08: Proceedings of the 2008 conference on Computing frontiers, pages 189-198, New York, NY, USA, 2008.

