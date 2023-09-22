# Lab1实验报告


## 一、Lab1
## (1)练习1：理解内核启动中的程序入口操作

### 1. la sp, bootstacktop
作用：将bootstacktop的地址加载到sp寄存器中，将栈指针指向bootstacktop, 即设置了内核栈的起始地址。

目的：为了初始化内核栈，让内核能够使用bootstack作为其栈空间。

### 2. tail kern_init

作用：该指令是一个尾调用指令，它将当前函数的栈帧清除，然后跳转到kern_init函数。

目的：为了节省栈空间，避免不必要的栈帧堆叠。

## (2)练习2：完善中断处理 （需要编程）
根据实验要求，参考了实验文档中提供的思路，编写了kern/trap/trap.c中"IRQ_S_TIMER"中断情况下的代码（具体代码详情见lab1/kern/trap/trap.c）。

首先调用了clock_set_next_event()函数实现中断，接着使操作系统每遇到100次时钟中断后，向屏幕上打印一行文字”100 ticks”，调用print_ticks子程序，在打印完10行后，调用sbi.h中的sbi_shutdown()函数关机。

<img src="C:\Users\复活少年\AppData\Roaming\Typora\typora-user-images\image-20230922125729525.png" alt="image-20230922125729525" style="zoom: 67%;" /><img src="C:\Users\复活少年\AppData\Roaming\Typora\typora-user-images\image-20230922125518120.png" alt="image-20230922125518120" style="zoom:50%;" />

![image-20230922125818929](C:\Users\复活少年\AppData\Roaming\Typora\typora-user-images\image-20230922125818929.png)

## (3) 扩展练习 Challenge1：描述与理解中断流程

a. ucore中处理中断异常的流程（从异常的产生开始）?

答：首先，在kern/init/init.c中，调用clock_init()触发时钟中断，而该函数触发中断的函数调用流程如下：clock_init()->clock_set_next_event()->sbi_set_timer()->sbi_call()，然后程序会跳转到kern/trap/trapentry.S的__alltraps标记处，接着保存当前进程的上下文（SAVE_ALL），并通过函数调用（jal trap），切换到kern/trap/trap.c的中断处理函数trap()的上下文，进入trap()的执行流。切换前的上下文作为一个结构体（trapframe），传递给trap()作为函数参数，然后trap()按照中断类型（此参数或许是sbi_call的参数SBI_SET_TIMER），进行分发（interrupt_handle或者exception_handle），执行时钟中断对应的处理语句，累加计数器，并设置下一次时钟中断，完成处理之后，会返回到kern/trap/trapentry.S，最后恢复到原先的上下文（RESTORE ALL），中断处理结束。

b. mov a0，sp的目的是什么?

答：将栈指针sp的值存储在寄存器a0中。栈指针此时栈帧（stack frame）的顶部，而栈帧包含了中断前程序的上下文寄存器，所以a0，即上下文寄存器，将作为参数传递给下面的trap函数。

c. SAVE_ALL中寄存器保存在栈中的位置是什么确定的?

答：每个寄存器的保存位置都是通过将其索引乘以REGBYTES得到的，REGBYTES即为寄存器的大小。

d. 对于任何中断，__alltraps 中都需要保存所有寄存器吗？请说明理由。

答：需要保存所有的寄存器。这是因为中断可能会发生在任何时刻，而且中断处理程序（trap）可能需要访问或修改所有寄存器的值，保存所有寄存器可以确保中断处理程序具有完整的上下文信息，并能够正确地执行所需的操作。

## （4）拓展练习 Challenge2：理解上下文切换机制
a. 在trapentry.S中汇编代码 csrw sscratch, sp；csrrw s0, sscratch, x0实现了什么操作，目的是什么？

答：汇编代码csrw sscratch, sp将寄存器sp的值存储到sscratch寄存器中；而csrrw s0, sscratch, x0则将sscratch寄存器的值存储到寄存器s0中，但由于源寄存器为x0，所以并不会并将寄存器x0的值写入sscratch寄存器。

在RISC-V架构中，sscratch寄存器是一个特殊的寄存器，用于保存异常处理程序的上下文。当异常发生时，处理器会将当前的上下文信息保存到sscratch寄存器中，然后跳转到异常处理程序。在异常处理程序执行完毕后，可以通过将sscratch寄存器的值恢复到s0寄存器中，来恢复之前保存的上下文信息。

所以这两个汇编代码的目的是保存和恢复异常处理程序的上下文。

b. save all里面保存了stval scause这些csr，而在restore all里面却不还原它们？那这样store的意义何在呢？

答：SAVE_ALL宏用于保存所有寄存器的状态，包括通用寄存器和一些特殊寄存器（如sscratch，sstatus，sepc，sbadaddr和scause）。这些寄存器的值被保存在栈上，以便在以后恢复它们。

未被恢复的特殊寄存器有三个：sscratch用于保存异常处理程序的上下文，sbadaddr用于存储错误或异常的地址，scause为S态原因寄存器。这些寄存器都存储与中断异常处理相关的值，被用于trap处理中断异常，不会对原来的程序状态造成影响，所以可以不恢复。

STORE他们是为了给trap传参，用于处理中断异常。

## （5）扩展练习Challenge3：完善异常中断
### 1.编写"Illegal instruction"和"ebreak"的异常处理代码
此部分代码位于kern/trap/trap.c中的exception_handler()函数中，两种异常的处理编写方式类似，分为三步：

（1）输出指令异常类型（Illegal instruction或ebreak）；

（2）输出异常指令地址（为上下文的结构体中epc所指的地址）；

（3）更新 tf->epc寄存器，使其指向下一个地址。

### 2.触发"Illegal instruction"和"ebreak"异常的代码
此部分代码位于kern/init/init.c中，只有两行代码，使用到了"asm volatile （"instruction"）"的语法形式。这是一个C/C++中的内联汇编表达式，会执行括号中的汇编指令，只需要写入能够触发上述两个异常的汇编指令即可。

![image-20230922125844904](C:\Users\复活少年\AppData\Roaming\Typora\typora-user-images\image-20230922125844904.png)

## （6）本实验中重要的知识点

### 1.riscv的中断机制：中断（interrupt）机制，就是不管CPU现在手里在干啥活，收到“中断”的时候，都先放下来去处理其他事情，处理完其他事情可能再回来干手头的活。在RISCV里，中断(interrupt)和异常(exception)统称为"trap"。
### 2.riscv的特权指令（与中断相关）：这里就包括在challeng 3中使用到的用于两个异常的汇编指令（ebreak和mret），此外还有ecall和sret。牵扯到riscv的权限模式，M-mode(机器模式，缩写为 M 模式)和S-mode(监管者模式，缩写为 S 模式)，的相互转换。
### 3.上下文切换：中断的处理需要“放下当前的事情但之后还能回来接着之前往下做”，对于CPU来说，实际上只需要把原先的寄存器保存下来，做完其他事情把寄存器恢复回来就可以了。这些寄存器也被叫做CPU的context(上下文，情境)。我们要用汇编实现上下文切换(context switch)机制，这包含两步：保存CPU的寄存器（上下文）到内存中（栈上），从内存中（栈上）恢复CPU的寄存器。在这个实验中，定义了一个结构体（trapframe）来保存这些寄存器。
