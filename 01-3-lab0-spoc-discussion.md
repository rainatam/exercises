- 你理解的对于类似ucore这样需要进程/虚存/文件系统的操作系统，在硬件设计上至少需要有哪些直接的支持？至少应该提供哪些功能的特权指令？

  答：能探測系統物理內存布局、、分配系统资源、修改虚存的段表和页表，修改用户的访问权限、允许和禁止中断、清理内存、停止一个中央处理器的工作、执行I/O操作、建立存储键等。



- 你理解的x86的实模式和保护模式有什么区别？物理地址、线性地址、逻辑地址的含义分别是什么？

  答：可访问的物理内存空间大小不同，实模式下可访问的物理内存空间不超过1MB，在保护方式下，全部32条地址线有效，可寻址高达4G字节的物理地址空间。

  物理地址：是处理器提交到总线上用于访问计算机系统中的内存和外设的最终地址。一个计算机系统中只有一个物理地址空间。 逻辑地址：在有地址变换功能的计算机中,访问指令给出的地址叫逻辑地址。 线性地址：线性地址是逻辑地址到物理地址变换之间的中间层，处理器通过段(Segment)机制控制下的形成的地址空间。



- 理解list_entry双向链表数据结构及其4个基本操作函数和ucore中一些基于它的代码实现（此题不用填写内容）



- 对于如下的代码段，请说明":"后面的数字是什么含义

```
 /* Gate descriptors for interrupts and traps */
 struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
 };

```

答：每一个filed(域，成员变量)在struct(结构)中所占的位数; 也称“位域”，用于表示这个成员变量占多少位(bit)。



```
 /* Gate descriptors for interrupts and traps */
 struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
 };
```

```
 /* Gate descriptors for interrupts and traps */
 struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
 };
```

- 对于如下的代码段，

```
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}

```

如果在其他代码段中有如下语句，

```
unsigned intr;
intr=8;
SETGATE(intr, 1,2,3,0);

```

请问执行上述指令后， intr的值是多少？

答： 0x10002



### 课堂实践练习

#### 练习一

请在ucore中找一段你认为难度适当的AT&T格式X86汇编代码，尝试解释其含义。

boot/bootasm.S的代码流程解释如下：

禁止中斷，設置段寄存器 Data Segment, Extra Segment, Stack Segment：

```assembly
.code16
	cli
    cld 
    xorw %ax, %ax 
    movw %ax, %ds 
    movw %ax, %es
    movw %ax, %ss
```

使能 A20 地址線：先等待 8042 输入缓存为空，然后发送写 8042 输出端口的指令，再等待输入缓存空，使能 A20

```assembly
seta20.1:
    inb $0x64, %al
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al 
    outb %al, $0x64
    
seta20.2:
    inb $0x64, %al
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al 
    outb %al, $0x60 
```

初始化 GDT 表：GDT 表是靜態設定好的，在汇编代码的最后先通过 gdt 标签定义了一个段描述符，之后通过 gdtdesc 定义了一个段表，数组的第0位存储段描述符。

```assembly
	lgdt gdtdesc
```

進入保護模式：將 CR0 寄存器的 PE 位置 1，開啟保護模式。長跳转到 32 位地址，完成到保护模式的转换。

```assembly
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
    
    ljmp $PROT_MODE_CSEG, $protcseg
```

设置保护模式的段寄存器，准备C程序栈，调用bootmain函数。

```assembly
.code32 
protcseg:
    movw $PROT_MODE_DSEG, %ax 
    movw %ax, %ds 
    movw %ax, %es 
    movw %ax, %fs 
    movw %ax, %gs 
    movw %ax, %ss

    movl $0x0, %ebp
    movl $start, %esp
    call bootmain
```

#### 练习二

宏定义和引用在内核代码中很常用。请枚举ucore中宏定义的用途，并举例描述其含义。

ROUNDDOWN(a, n)：向下取整為最近的 n 的倍數。

SetPageReserved(page)：把 Page 標記為保留的

