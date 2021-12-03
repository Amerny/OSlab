

# 				操作系统试验（哈工大）

​	

### ⛷四、基于内核栈切换的进程切换

#### 1. 基本知识

在现在的 Linux 0.11 中，真正完成进程切换是依靠任务状态段（Task State Segment，简称 TSS）的切换来完成的。

具体的说，在设计“Intel 架构”（即 x86 系统结构）时，每个任务（进程或线程）都对应一个独立的 TSS，TSS 就是内存中的一个结构体，里面包含了几乎所有的 CPU 寄存器的映像。有一个任务寄存器（Task Register，简称 TR）指向当前进程对应的 TSS 结构体，所谓的 TSS 切换就将 CPU 中几乎所有的寄存器都复制到 TR 指向的那个 TSS 结构体中保存起来，同时找到一个目标 TSS，即要切换到的下一个进程对应的 TSS，将其中存放的寄存器映像“扣在” CPU 上，就完成了执行现场的切换，这部分很好理解。

具体的工作过程是：

- （1）首先用 <font color=purple>TR 中存取的段选择符在 GDT 表中找到当前 TSS 的内存位置</font>，由于 TSS 是一个段，所以需要用段表中的一个描述符来表示这个段，和在系统启动时论述的内核代码段是一样的，那个段用 GDT 中的某个表项来描述，还记得是哪项吗？是 8 对应的第 1 项。此处的 TSS 也是用 GDT 中的某个表项描述，而 TR 寄存器是用来表示这个段用 GDT 表中的哪一项来描述，所以 TR 和 CS、DS 等寄存器的功能是完全类似的。
- （2）找到了当前的 TSS 段（就是一段内存区域）以后，将 CPU 中的寄存器映像存放到这段内存区域中，即拍了一个<font color=purple>快照。</font>
- （3）存放了当前进程的执行现场以后，接下来要找到目标进程的现场，并将其扣在 CPU 上，找目标 TSS 段的方法也是一样的，因为找段都要从一个描述符表中找，描述 TSS 的描述符放在 GDT 表中，所以找目标 TSS 段也要靠 GDT 表，当然只要给出目标 TSS 段对应的描述符在 GDT 表中存放的位置——段选择子就可以了，仔细想想系统启动时那条著名的 `jmpi 0, 8` 指令，这个段选择子就放在 ljmp 的参数中，实际上就 `jmpi 0, 8` 中的 8。
- （4）一旦将目标 TSS 中的全部寄存器映像扣在 CPU 上，就相当于切换到了目标进程的执行现场了，因为那里有目标进程停下时的 `CS:EIP`，所以此时就开始从目标进程停下时的那个 `CS:EIP` 处开始执行，现在目标进程就变成了当前进程，所以 TR 需要修改为目标 TSS 段在 GDT 表中的段描述符所在的位置，因为 TR 总是指向当前 TSS 段的段描述符所在的位置。

所以基于 TSS 进行进程/线程切换的 `switch_to` 实际上就是一句 `ljmp` 指令：

```c
#define switch_to(n) {
    struct{long a,b;} tmp;
    __asm__(
        "movw %%dx,%1"
        "ljmp %0" ::"m"(*&tmp.a), "m"(*&tmp.b), "d"(TSS(n)
    )
 }
#define FIRST_TSS_ENTRY 4
#define TSS(n) (((unsigned long) n) << 4) + (FIRST_TSS_ENTRY << 3))
```

在此继续搬一下GDT，LDT，保护模式的知识吧

##### 1）80x86的系统寄存器

1. 标志寄存器，微机原理讲过，在此就放个图。

   ![EFlags](/Users/medima/Desktop/图片/EFlags.png)

2. 内存管理寄存器

   有四个分别是 `gdtr,ldtr,tr,idtr`，以下分别看下结构

   ![gdtr](/Users/medima/Desktop/图片/gdtr.png)

   + 全局描述符表寄存器 *GDTR* 

     32位基地址，16位长度。基地址用来指定 GDT 表中字节0 在线性地址空间中的地址，表长度用来指定表的长度。指令 `lgdt  sgdt`用来加载和保存寄存器的值

   + 中断描述符表寄存器 *IDTR* 

     32位基地址，16位长度。基地址用来指定 IDT 表中字节0在线性地址空间中的地址，表长度用来指定表的长度。指令 `lidt  sidt`用来加载和保存寄存器的值

   + 局部描述符表寄存器 *LDTR* 

     同样有指令 `lldt  sldt`。<font color=red>包含*LDT* 表 的段必须在 GDT 中有一个段描述符项。</font> 当 `lldt`的时候会把含有 LDT的段描述符加载进 LDTR，LDT段描述符的段基地址，段限长度和描述符属性会自动加到 LDTR。进行任务切换的时候，处理器会把新任务的段描述符以及段选择符自动加载到 LDTR中。

   + 任务寄存器 *TR*

     同样有指令 `ltr  str`。用于存放当前任务 TSS（任务状态段） 段的16位段选择符，32位基地址，16位长度和描述符属性值。当使用LTR指令把选择符加载进任务寄存器时，TSS描述符中的段基地址、段限长度以及描述符属性会被自动加载到任务寄存器中。当执行任务切换时，处理器会把新任务的TSS的段选择符和段描述符自动加载进任务寄存器TR中。

3. 控制寄存器

   ![cr](/Users/medima/Desktop/图片/cr.png)

##### 2）保护模式

![截屏2021-11-21 上午1.40.40](/Users/medima/Desktop/图片/trans.png)

##### 3）分段机制

<font color=red>保护模式下的段描述符。</font>

![prote](/Users/medima/Desktop/图片/prote.png)

<font color=red>段选择符</font>

![](/Users/medima/Desktop/图片/select.png)

TI为描述符索引，为0从GDT中查找；1从LDT中查找

RPL为请求访问者所用的访问权限，0~3，0的权限最大。通常情况下，CS 和 SS 中 RPL 就组成了 CPL（当前权限级别），所以常常是 RPL=CPL，进而 CPL 就表示发起访问者要以什么权限去访问目标段，当 CPL 大于目标段 DPL 时，则 CPU 禁止访问，只有 CPL 小于等于目标段 DPL 时才能访问。

<font color=red>逻辑地址到线性地址的转换</font>

1. 段选择符中的偏移值在GDT或者LDT中定位相应的段描述符（*仅当一个新的段描述符加载到段寄存器中时才需要这样）*

2. 利用段描述符检查段的访问权限和范围

3. 将段描述符中的段基地址加到偏移量上。用图形如下描述。

   ![](/Users/medima/Desktop/图片/findaddr.png)

<font color=red>保护模式下的中断</font>

保护模式下的<font color=purple>中断要权限检查，还有特权级的切换</font>，所以就需要扩展中断向量表的信息，即每个中断用一个中断门描述符来表示，也可以简称为中断门，中断门描述符依然有自己的格式，如下图所示。

![img](https://static001.geekbang.org/resource/image/e1/0b/e11b9de930a09fb41bd6ded9bf12620b.jpg?wh=3809*2105)

![img](https://static001.geekbang.org/resource/image/ff/5b/ff5c25c85a7fa28b17f386848f19fb5b.jpg?wh=2696*1780)

产生中断后，CPU 首先会检查中断号是否大于**最后一个中断门描述符**，x86 CPU 最大支持 256 个中断源（即中断号：0~255），然后检查描述符类型（是否是中断门或者陷阱门）、**是否为系统描述符**，是不是存在于内存中。接着，检查中断门描述符中的段选择子指向的段描述符。**最后做权限检查**，如果 CPL 小于等于中断门的 DPL，并且 CPL 大于等于中断门中的段选择子所指向的段描述符的 DPL，就指向段描述符的 DPL。进一步的，CPL 等于中断门中的段选择子指向段描述符的 DPL，则为同级权限不进行栈切换，否则进行栈切换。如果进行栈切换，还需要从 TSS 中加载具体权限的 SS、ESP，当然也要对 SS 中段选择子指向的段描述符进行检查。做完这一系列检查之后，CPU 才会加载中断门描述符中目标代码段选择子到 CS 寄存器中，把目标代码段偏移加载到 EIP 寄存器中。

##### 4）分页机制

通过设置CR0的PG位可以启用分页机制。

![](/Users/medima/Desktop/图片/page.png)

通过分段机制得到的线性地址经过分页机制中的页机制变化就可以得到最终的物理地址。具体知识在内存管理。

##### 5）TSS切换

GDT 表的结构如下图所示，所以第一个 TSS 表项，即 0 号进程的 TSS 表项在第 4 个位置上，4<<3，即 `4 * 8`，相当于 TSS 在 GDT 表中开始的位置，TSS（n）找到的是进程 n 的 TSS 位置，所以还要再加上 n<<4，即 `n * 16`，因为每个进程对应有 1 个 TSS 和 1 个 LDT，每个描述符的长度都是 8 个字节，所以是乘以 16，其中 LDT 的作用就是上面论述的那个映射表，关于这个表的详细论述要等到内存管理一章。`TSS(n) = n * 16 + 4 * 8`，得到就是进程 n（切换到的目标进程）的 TSS 选择子，将这个值放到 dx 寄存器中，并且又放置到结构体 tmp 中 32 位长整数 b 的前 16 位，现在 64 位 tmp 中的内容是前 32 位为空，这个 32 位数字是段内偏移，就是 `jmpi 0, 8` 中的 0；接下来的 16 位是 `n * 16 + 4 * 8`，这个数字是段选择子，就是 `jmpi 0, 8` 中的 8，再接下来的 16 位也为空。所以 swith_to 的核心实际上就是 `ljmp 空, n*16+4*8`，现在和前面给出的基于 TSS 的进程切换联系在一起了。

<img src="/Users/medima/Desktop/图片/123.png" alt="图片描述信息"  />

##### 6）本次实验内容

虽然用一条指令就能完成任务切换，但这指令的执行时间却很长，这条 ljmp 指令在实现任务切换时大概需要 200 多个时钟周期。而通过堆栈实现任务切换可能要更快，而且采用堆栈的切换还可以使用指令流水的并行优化技术，同时又使得 CPU 的设计变得简单。**所以无论是 Linux 还是 Windows，进程/线程的切换都没有使用 Intel 提供的这种 TSS 切换手段，而都是通过堆栈实现的。**

要实现基于内核栈的任务切换，主要完成如下三件工作：

- （1）重写 `switch_to`；
- （2）将重写的 `switch_to` 和 `schedule()` 函数接在一起；
- （3）修改现在的 `fork()`。

#### 2. 具体步骤

我们不用 TSS 进行切换，而是采用切换**内核栈的方式来完成进程切换**，所以在新的 switch_to 中将用到当前进程的 PCB、目标进程的 PCB、当前进程的内核栈、目标进程的内核栈等信息。由于 Linux 0.11 **进程的内核栈和该进程的 PCB 在同一页内存上**（一块 4KB 大小的内存），其中 PCB 位于这页内存的低地址，栈位于这页内存的高地址；另外，由于当前进程的 PCB 是用一个全局变量 current 指向的，所以只要**告诉新 switch_to()函数一个指向目标进程 PCB 的指针就可以了**。同时还要将 next 也传递进去，虽然 TSS(next)不再需要了，但是 **LDT(next)仍然是需要的**，也就是说，现在每个进程不用有自己的 TSS 了，因为已经不采用 TSS 进程切换了，但是每个进程需要有自己的 LDT，地址分离地址还是必须要有的，而进程切换必然要涉及到 LDT 的切换

##### 1）修改 `schedule`

在文件 `kernel/sched.c`中

原来的调用 `switch_to`宏在函数最后一句。

```c
switch_to(next);     // 切换到Next任务并运行。

//改为
switch_to(pnext, _LDT(next));

if ((*p)->state == TASK_RUNNING && (*p)->counter > c)
				c = (*p)->counter, next = i, pnext = *p;

//并且在前面加入下面这一行
struct tss_struct *tss = &(init_task.task.tss);
```

既然 `switch_to`的调用参数都已经修改了，那么接下来也要去修改 `switch_to`定义了。

##### 2）修改 `switch_to`

由于要对内核栈进行精细的操作，所以需要用汇编代码来完成函数 `switch_to` 的编写。

这个函数依次主要完成如下功能：由于是 C 语言调用汇编，所以需要首先在汇编中处理栈帧，即处理 `ebp` 寄存器；接下来要取出表示下一个进程 PCB 的参数，**并和 `current` 做一个比较**，如果等于 current，则什么也不用做；如果不等于 current，就开始进程切换，依次**完成 PCB 的切换、TSS 中的内核栈指针的重写、内核栈的切换、LDT 的切换以及 PC 指针（即 CS:EIP）的切换。**

原来的 `switch_to`在 `include/linux/sched.h`中定义，所以这里将原来的定义删除

```c
#define switch_to(n) {\
struct {long a,b;} __tmp; \
__asm__("cmpl %%ecx,current\n\t" \
	"je 1f\n\t" \
	"movw %%dx,%1\n\t" \
	"xchgl %%ecx,current\n\t" \
	"ljmp *%0\n\t" \
	"cmpl %%ecx,last_task_used_math\n\t" \
	"jne 1f\n\t" \
	"clts\n" \
	"1:" \
	::"m" (*&__tmp.a),"m" (*&__tmp.b), \
	"d" (_TSS(n)),"c" ((long) task[n])); \
}
//这是基于TSS的进程切换
```

并且在 `kernel/system_call.s`中重新编写如下

```c
.align 2
switch_to:
# 压栈操作
    pushl %ebp
    movl %esp,%ebp
    pushl %ecx
    pushl %ebx
    pushl %eax
    movl 8(%ebp),%ebx					 # 调用switch_to的第一个参数，即pnext——目标进程的PCB
# 和当前进程做比较
    cmpl %ebx,current
    je 1f
# 切换PCB
    # ...
  	movl %ebx,%eax    # ebx --> eax 
    xchgl %eax,current  # eax 现在是当前进程， current拿的 eax 也就是 ebx即下一进程
    
# TSS中的内核栈指针的重写
    # ...
  	movl tss,%ecx
		addl $4096,%ebx
		movl %ebx,ESP0(%ecx)
# 切换内核栈
    # ...
  	KERNEL_STACK = 12
		movl %esp,KERNEL_STACK(%eax)		# 保存当前进程的内核栈
# 再取一下 ebx，因为前面修改过 ebx 的值
		movl 8(%ebp),%ebx
		movl KERNEL_STACK(%ebx),%esp		# 加载下一个进程的内核栈
# 切换LDT
    # ...
  	movl 12(%ebp),%ecx 							# 对应传入的参数 LDT（pnext）
  	lldt %cx 
# 重置一下用户态内存空间指针的选择符fs
    movl $0x17,%ecx
    mov %cx,%fs
# 和后面的 clts 配合来处理协处理器，由于和主题关系不大，此处不做论述
    cmpl %eax,last_task_used_math
    jne 1f
    clts

1:    popl %eax
    popl %ebx
    popl %ecx
    popl %ebp
ret

```

前面已经详细论述过，在中断的时候，要找到内核栈位置，并将用户态下的 `SS:ESP`，`CS:EIP` 以及 `EFLAGS` 这五个寄存器压到内核栈中，这是沟通用户栈（用户态）和内核栈（内核态）的关键桥梁，而找到内核栈位置就依靠 **TR 指向的当前 TSS。**

现在虽然不使用 TSS 进行任务切换了，但是 Intel 的这态中断处理机制还要保持，所以仍然需要有一个当前 TSS，这个 TSS 就是我们定义的那个全局变量 tss，即 0 号进程的 tss，<font color=red>所有进程都共用这个 tss，</font>任务切换时不再发生变化。定义 `ESP0 = 4` 是因为 TSS 中**内核栈底指针 esp0** 就放在偏移为 4 的地方，看一看 tss 的结构体定义就明白了。这里具体说明一下这段代码。`%ecx`里面存的是tss段的首地址，在后面我们会知道，<font color=red>tss段的首地址就是进程0的tss的首地址</font>，根据这个tss段里面的内核栈指针找到内核栈，所以在切换时就要更新这个内核栈指针。也就是说，<u>任何正在运行的进程内核栈都被进程0的tss段里的某个指针指向，我们把该指针叫做内核栈指针</u>。`addl $4096,%ebx`未加4KB前，ebx指向下一个进程的PCB首地址，加4096后，相当于为该进程开辟了一个“进程页”，ebx此时指向进程页的最高地址。`movl %ebx,ESP0(%ecx)`将内核栈底指针放进tss段的偏移为ESP0（=4）的地方，作为寻找当前进程的内核栈的依据。 由上面一段代码可以知道们的“进程页”是这样的，PCB由低地址向上扩展，栈由上向下扩展。也可以这样理解，一个进程页就是PCB，我们把内核栈放在最高地址，其它的task_struct从最低地址开始扩展

```c
struct tss_struct {
	long	back_link;	/* 16 high bits zero */
	long	esp0;
	long	ss0;		/* 16 high bits zero */
  //.....
}
```



完成<font color=red>内核栈的切换</font>也非常简单，和我们前面给出的论述完全一致，**将寄存器 esp（内核栈使用到当前情况时的栈顶位置）的值保存到当前 PCB 中，再从下一个 PCB 中的对应位置上取出保存的内核栈栈顶放入 esp 寄存器**，这样处理完以后，再使用内核栈时使用的就是下一个进程的内核栈了。

由于现在的 Linux 0.11 的 PCB 定义中没有保存内核栈指针这个域（kernelstack），所以需要加上，而宏 `KERNEL_STACK` 就是你加的那个位置，当然将 kernelstack 域加在 task_struct 中的哪个位置都可以，但是在某些汇编文件中（主要是在 `kernal/system_call.s` 中）有些关于操作这个结构一些汇编硬编码，所以一旦增加了 kernelstack，这些硬编码需要跟着修改，由于第一个位置，即 long state 出现的汇编硬编码很多，所以 kernelstack 千万不要放置在 task_struct 中的第一个位置，当放在其他位置时，修改 `kernal/system_call.s` 中的那些硬编码就可以了。

```c
// 在 include/linux/sched.h 中
struct task_struct {
    long state;
    long counter;
    long priority;
    long kernelstack;
//......
```

此时还需要修改`INIT_STACK`

```c
#define INIT_TASK \
/* state etc */	{ 0,15,15, \
/* signals */	0,{{},},0, \
/* ec,brk... */	0,0,0,0,0,0, \
  
//改为
  
#define INIT_TASK \
								{ 0,15,15, \
								PAGE_SIZE+(long)&init_task, 0,{{},},0, \
```

以及 `kernel/system_call.s`中的声明

```c
/*在system_call.s下*/
ESP0 = 4    #此处新添
KERNEL_STACK = 12   #此处新添
//全局函数调用
 .globl switch_to
state   = 0     # these are offsets into the task-struct.
counter = 4
priority = 8
signal  = 16    #此处修改
sigaction = 20  #此处修改
blocked = (37*16)   #此处修改
```

和 `sched.c`中的声明

```c
/*添加在文件头部附近*/
extern long switch_to(struct task_struct *p, unsigned long address);
```



再下一个切换就是<font color=red> LDT 的切换了</font>，指令 `movl 12(%ebp),%ecx` 负责取出对应 LDT(next)的那个参数，指令 `lldt %cx` 负责修改 LDTR 寄存器，一旦完成了修改，下一个进程在执行用户态程序时使用的映射表就是自己的 LDT 表了，地址空间实现了分离。

最后一个切换是关于<font color=red> PC 的切换</font>，和前面论述的一致，依靠的就是 `switch_to` 的最后一句指令 ret，虽然简单，但背后发生的事却很多：`schedule()` 函数的最后调用了这个 `switch_to` 函数，所以这句指令 ret 就返回到下一个进程（目标进程）的 `schedule()` 函数的末尾，遇到的是}，继续 ret 回到调用的 `schedule()` 地方，是在中断处理中调用的，所以回到了中断处理中，就到了中断返回的地址，再调用 iret 就到了目标进程的用户态程序去执行，和书中论述的内核态线程切换的五段论是完全一致的。

switch_to 代码中在切换完 LDT 后的两句，即：

```c
! 切换 LDT 之后
movl $0x17,%ecx
mov %cx,%fs
```

这两句代码的含义是重新取一下段寄存器 fs 的值，这两句话必须要加、也必须要出现在切换完 LDT 之后，这是因为在实践项目 2 中曾经看到过 fs 的作用——通过 fs 访问进程的用户态内存，LDT 切换完成就意味着切换了分配给进程的用户态内存地址空间，所以前一个 fs 指向的是上一个进程的用户态内存，而现在需要执行下一个进程的用户态内存，所以就需要用这两条指令来重取 fs。

不过，细心的读者可能会发现：fs 是一个选择子，即 fs 是一个指向描述符表项的指针，这个描述符才是指向实际的用户态内存的指针，所以上一个进程和下一个进程的 fs 实际上都是 0x17，真正找到不同的用户态内存是因为两个进程查的 LDT 表不一样，所以这样重置一下 `fs=0x17` 有用吗，有什么用？要回答这个问题就需要对段寄存器有更深刻的认识，实际上段寄存器包含两个部分：显式部分和隐式部分，如下图给出实例所示，就是那个著名的 `jmpi 0, 8`，虽然我们的指令是让 `cs=8`，但在执行这条指令时，会在段表（GDT）中找到 8 对应的那个描述符表项，取出基地址和段限长，除了完成和 eip 的累加算出 PC 以外，还会将取出的基地址和段限长放在 cs 的隐藏部分，即图中的基地址 0 和段限长 7FF。为什么要这样做？下次执行 `jmp 100` 时，由于 cs 没有改过，仍然是 8，所以可以不再去查 GDT 表，而是直接用其隐藏部分中的基地址 0 和 100 累加直接得到 PC，增加了执行指令的效率。现在想必明白了为什么重新设置 fs=0x17 了吧？而且为什么要出现在切换完 LDT 之后？加载即通知我变了。

<img src="https://doc.shiyanlou.com/userid19614labid571time1424053856897" alt="图片描述信息" style="zoom:150%;" />

<mark>切换五段论</mark>

首先列出这种切换机制下PCB的结构。ESP0即栈底，esp即栈顶。

![在这里插入图片描述](https://i2.wp.com/img-blog.csdnimg.cn/20201212170452179.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2x5ajE1OTczNzQwMzQ=,size_16,color_FFFFFF,t_70)

1. 中断进入阶段

   中断进入，是`switch_to`的第一阶段，就是int指令或其它硬件中断的中断处理入口，核心工作就是记录当前程序在用户态执行时的信息，如当前使用的用户栈、当前程序的执行位置、当前执行的现场信息等。

   ![在这里插入图片描述](https://i2.wp.com/img-blog.csdnimg.cn/20201212194148326.png)

   当执行系统调用`int 0x80`从而执行到`system_call()`，又会把当前执行的现场信息压入内核栈，一些用户态的寄存器信息。

   `system_call`中执行完相应的系统调用`sys_call_xx`后，又将函数的返回值eax压栈。

   若引起进程调度，则跳转执行reschedule。否则则执行`ret_from_sys_call`。

   ![在这里插入图片描述](https://i2.wp.com/img-blog.csdnimg.cn/20201212214657327.png)

2. 调用`schedule`引起PCB切换

   内核栈会压入参数_LDT(next)、pnext，然后压入它的下一条指令的位置，也就是 “}”此时内核栈如下：

   ![在这里插入图片描述](https://i2.wp.com/img-blog.csdnimg.cn/20201212214619938.png)

   进入`switch_to`后，将一些调用者寄存器压栈，以便后续使用这些寄存器。

   ![在这里插入图片描述](https://i2.wp.com/img-blog.csdnimg.cn/20201212214552877.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2x5ajE1OTczNzQwMzQ=,size_16,color_FFFFFF,t_70)

3. 内核栈切换

   这个阶段内核栈变化了。一个进程被挂起，也意味着其内核栈被挂起，这时切换到一个之前被挂起的进程，并将其内核栈的栈顶地址加载到esp。但是，我们要注意一点：任何进程被挂起，都要经过上面的过程，也就意味着内核栈的结构没变，但是**里面的内容变了**。

4. 中断返回前

   ①切换内核栈后，后续经过一些和内核栈无关的操作后，最后在switch_to弹出eax、ebx、ecx、ebp。

   ![在这里插入图片描述](https://i2.wp.com/img-blog.csdnimg.cn/2020121222022726.png)

   ②switch_to的ret

   ![在这里插入图片描述](https://i2.wp.com/img-blog.csdnimg.cn/20201212221139991.png)

   ret_from_sys_call的最后弹出了一系列用户态寄存器信息，此时内核栈信息如下

   ![在这里插入图片描述](https://i2.wp.com/img-blog.csdnimg.cn/20201212221504182.png)

5. 中断返回

   然后再执行iret指令，该指令就可以设置好用户栈(SS:ESP)、EFLAGS、用户指令执行位置(CS:EIP)到相应寄存器，然后就直接跳转回用户态执行。此时**用户态执行的是目标进程**也就是所谓的 next 这条硬件指令和int 0x80对应来看就很容易理解了。

##### 3）修改 `fork.c`

 fork() 这个叉子的含义就是要**让父子进程共用同一个代码、数据和堆栈**，现在虽然是使用内核栈完成任务切换，但 fork() 的基本含义不会发生变化。修改 fork() 的核心工作就是要形成如下图所示的子进程内核栈结构。

![图片描述信息](/Users/medima/Desktop/图片/1323.png)

1. **<font color=red>对 fork() 的修改就是对子进程的内核栈的初始化</font>**，在 fork() 的核心实现 `copy_process` 中，`p = (struct task_struct *) get_free_page();`用来完成申请一页内存作为子进程的 PCB，而 p 指针加上页面大小就是子进程的内核栈位置，所以语句 `krnstack = (long *) (PAGE_SIZE + (long) p);` 就可以找到子进程的内核栈位置，接下来就是初始化 krnstack 中的内容了。

```c
*(--krnstack) = ss & 0xffff;
*(--krnstack) = esp;
*(--krnstack) = eflags;
*(--krnstack) = cs & 0xffff;
*(--krnstack) = eip;
```

这五条语句就完成了上图所示的那个重要的关联，因为其中 ss,esp 等内容都是 `copy_proces()` 函数的参数，这些参数来自调用 `copy_proces()` 的进程的内核栈中，就是父进程的内核栈中，所以上面给出的指令不就是将父进程内核栈中的前五个内容拷贝到子进程的内核栈中，图中所示的关联不也就是一个拷贝吗？

因此需要改动的是在 `fork.c`中修改

```c
/*在fork.c的copy_process()添加以下代码即可*/
long *krnstack;
p = (struct task_struct *) get_free_page();
krnstack = (long)(PAGE_SIZE +(long)p);
 *(--krnstack) = ss & 0xffff;
 *(--krnstack) = esp;
  *(--krnstack) = eflags;
 *(--krnstack) = cs & 0xffff;
 *(--krnstack) = eip;
 *(--krnstack) = ds & 0xffff;
 *(--krnstack) = es & 0xffff;
 *(--krnstack) = fs & 0xffff;
 *(--krnstack) = gs & 0xffff;
 *(--krnstack) = esi;
 *(--krnstack) = edi;
 *(--krnstack) = edx;
 *(--krnstack) = (long)first_return_from_kernel;
 *(--krnstack) = ebp;
 *(--krnstack) = ecx;
 *(--krnstack) = ebx;
 *(--krnstack) = 0;//eax，到最后作为fork()的返回值，即子进程的pid
 p->kernelstack = krnstack;	//将存放在 PCB 中的内核栈指针修改到初始化完成时内核栈的栈顶
 //......
```

**子进程的内核栈如下:** 和之前的内核栈结构一毛一样哈哈😄

![在这里插入图片描述](https://i2.wp.com/img-blog.csdnimg.cn/20201212233556570.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2x5ajE1OTczNzQwMzQ=,size_16,color_FFFFFF,t_70)

2. 现在到了 ret 指令了，这条指令要从内核栈中弹出一个 32 位数作为 EIP 跳去执行，所以需要弄一个函数地址（仍然是一段汇编程序，所以这个地址是这段汇编程序开始处的标号）并将其初始化到栈中。我们弄的一个名为 `first_return_from_kernel` 的汇编标号，然后可以用语句 `*(--krnstack) = (long) first_return_from_kernel;` 将这个地址初始化到子进程的内核栈中，现在执行 ret 以后就会跳转到 `first_return_from_kernel` 去执行了。

`first_return_from_kernel` 要完成什么工作？PCB 切换完成、内核栈切换完成、LDT 切换完成，接下来应该那个“内核级线程切换五段论”中的最后一段切换了，即完成用户栈和用户代码的切换，依靠的核心指令就是 iret，当然在切换之前应该回复一下执行现场，主要就是 `eax,ebx,ecx,edx,esi,edi,gs,fs,es,ds` 等寄存器的恢复.

编写 `first_return_from_kernel` 

```c
/*在system_call.s下*/
.align 2
first_return_from_kernel:
    popl %edx
    popl %edi
    popl %esi
    pop  %gs
    pop  %fs
    pop  %es
    pop  %ds
    iret
```

```c
/*在system_call.s头部附近*/
.globl first_return_from_kernel
  
 /*在fork.c头部附近*/
extern void first_return_from_kernel(void);
```

这次实验比较乱，但内容其实不多，`make all`并且之后 `../run`就可以了🤩

