

# <center>LAB 0.5 比麻雀更小的麻雀</center>

#### 实验目的

- 了解操作系统的最小执行内核和启动流程

#### 实验内容

- **练习** **1:** **使用** **GDB** **验证启动流程**

​				硬件加电之后的几条代码是：

![image-20230921084123391](C:\Users\复活少年\AppData\Roaming\Typora\typora-user-images\image-20230921084123391.png)

​				

```c
=> 0x1000:	auipc	t0,0x0	//将当前地址的上半部分存储在t0中。设置一个全局偏移量。
   0x1004:	addi	a1,t0,32	//a1 = t0 + 32
   0x1008:	csrr	a0,mhartid	//获取当前硬件线程的ID
   0x100c:	ld	t0,24(t0)	//从t0+24处获取双字值到t0
   0x1010:	jr	t0	//跳转到t0寄存器中存储的地址，通常是用于启动程序
   0x1014:	unimp	//未定义的指令
   0x1016:	unimp	//
   0x1018:	unimp	//
   0x101a:	0x8000	//未使用的值
   0x101c:	unimp	//
```

​		主要作用是初始化一些寄存器和执行一些基本的启动操作，例如设置全局地址、加载数据、获取线程ID，并尝试跳转到某个地址。

- 跟随程序运行，跳转至0x80000000处

  ![image-20230921085040636](C:\Users\复活少年\AppData\Roaming\Typora\typora-user-images\image-20230921085040636.png)

​		我们继续跟随程序运行,发现程序通过0x8000000c处的代码跳转至0x800006a0,之后程序会通过ret返回至0x80000010处。

<img src="C:\Users\复活少年\AppData\Roaming\Typora\typora-user-images\image-20230921085430085.png" alt="image-20230921085430085" style="zoom: 80%;" />

<img src="C:\Users\复活少年\AppData\Roaming\Typora\typora-user-images\image-20230921085501879.png" alt="image-20230921085501879" style="zoom:80%;" />

总结：Risc-V加电过程中，完成了哪些事情？

答：

1. **复位操作**：这将把处理器的内部状态和寄存器初始化为预定的初始状态。复位操作确保处理器处于一个已知的状态，以便开始执行程序。

   内置rom0x1000存放复位代码，CPU上电时调到此处执行复位代码，然后指定PC=0x80000000，再运行bootloader。

2. **初始化寄存器**：对一些特殊寄存器进行初始化，例如程序计数器（PC）和栈指针寄存器。

3. **加载引导代码**：通常，在RISC-V处理器中，加电后需要执行引导代码，这是一段特殊的代码，用于启动系统。引导代码的任务包括初始化硬件设备、加载操作系统或其他应用程序，以及设置系统的初始状态。

4. **初始化外设**：如果系统中有外设（例如内存控制器、时钟控制器、中断控制器等），那么这些外设也需要在加电过程中进行初始化。这通常包括配置外设的寄存器、设置时钟频率和中断向量表等。

5. **执行引导代码**

6. **处理异常和中断**

7. **启动操作系统，交还控制权**

#### 从机器启动到操作系统运行的过程

1. ##### OpenSBI，bin，ELF

   ​		QEMU自带的bootloader: OpenSBI固件负责开机，并且加载OS到内存里。在 Qemu 开始执行任何指令之前，首先两个文件将被加载到 Qemu 的物理内存中：即作为 bootloader 的 OpenSBI.bin 被加载到物理内存以物理地址 0x80000000 开头的区域上，同时内核镜像 os.bin 被加载到以物理地址 0x80200000 开头的区域上。

   ​		elf文件和bin文件为两种不同的可执行文件格式。其中bin文件头会简单解释自己应该被加载到什么起始位置，而ELF文件包含冗余的调试信息，指定程序每个section的内存布局。因此，我们可以将内存布局合适的elf文件转化为bin文件，然后再加载到qemu中运行，就可以节省部分内存。

2. **内存布局，链接脚本，入口点**

   ​		.text 段，即代码段，存放汇编代码； .rodata 段，即只读数据段，；.data 段，存放被初始化的可读写数据，通常保存程序中的全局变量；.bss 段，存放被初始化为 00 的可读写数据。stack栈负责函数调用和局部变量的控制，heap堆负责动态内存的分配。

   ​		链接器的作用是把输入文件(往往是 .o文件)链接成输出文件(往往是elf文件)。一般来说，输入文件和输出文件都有很多section, 链接脚本(linker script)的作用，就是描述怎样把输入文件的section映射到输出文件的section, 同时规定这些section的内存布局。

   ​		链接脚本中，`entry.S`中的汇编代码, 作为整个内核的入口点。程序入口点定义为`kern_entry`，作用是分配好内核栈，随后跳转到真正的入口点`kern_init`。

3. **"真正的"入口点**

```
kern/init/init.c
```
