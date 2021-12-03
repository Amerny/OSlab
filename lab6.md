# 				操作系统试验（哈工大）



## 地址映射与共享

### 一、实验内容

#### 1. 跟踪地址翻译过程

​	首先以汇编级调试的方式启动 Bochs，引导 Linux 0.11，在 0.11 下编译和运行 test.c。它是一个无限循环的程序，永远不会主动退出。然后在调试器中通过查看各项系统参数，从逻辑地址、LDT 表、GDT 表、线性地址到页表，计算出变量 `i` 的物理地址。最后通过直接修改物理内存的方式让 test.c 退出运行。

```c
#include <stdio.h>

int i = 0x12345678;
int main(void)
{
    printf("The logical/virtual address of i is 0x%08x", &i);
    fflush(stdout);
    while (i)
        ;
    return 0;
}
```

#### 2. 基于共享内存的生产者—消费者程序

本项实验在 Ubuntu 下完成，与信号量实验中的 `pc.c` 的功能要求基本一致，仅有两点不同：

- 不用文件做缓冲区，而是使用共享内存；
- 生产者和消费者分别是不同的程序。生产者是 producer.c，消费者是 consumer.c。两个程序都是单进程的，通过信号量和缓冲区进行通信。

Linux 下，可以通过 `shmget()` 和 `shmat()` 两个系统调用使用共享内存。

#### 3. 共享内存的实现

本部分实验内容是在 Linux 0.11 上实现上述页面共享，并将上一部分实现的 producer.c 和 consumer.c 移植过来，验证页面共享的有效性。

具体要求在 `mm/shm.c` 中实现 `shmget()` 和 `shmat()` 两个系统调用。它们能支持 `producer.c` 和 `consumer.c` 的运行即可，不需要完整地实现 POSIX 所规定的功能。

+ **shmget()**

 ```c
 int shmget(key_t key, size_t size, int shmflg);
 ```

`shmget()` 会新建/打开一页内存，并返回该页共享内存的 shmid（该块共享内存在操作系统内部的 id）。

所有使用同一块共享内存的进程都要使用相同的 key 参数。

如果 key 所对应的共享内存已经建立，则直接返回 `shmid`。如果 size 超过一页内存的大小，返回 `-1`，并置 `errno` 为 `EINVAL`。如果系统无空闲内存，返回 -1，并置 `errno` 为 `ENOMEM`。

`shmflg`参数可以忽略

+ **shmat()**

```c
void *shmat(int shmid, const void *shmaddr, int shmflg);
```

`shmat()` 会将 `shmid` 指定的共享页面映射到当前进程的虚拟地址空间中，并将其首地址返回。

如果 `shmid` 非法，返回 `-1`，并置 `errno` 为 `EINVAL`。

`shmaddr` 和 `shmflg` 参数可忽略。

### 二、具体步骤

#### 1. 跟踪地址映射

##### 1）准备工作

+ 在Ubuntu终端中使用 进入汇编级调试模式

  ```bash
  cd ~/oslab
  
  ./dbg-asm	#进入调试,Bochs黑屏显示，此时在终端继续输入 c ,即可运行linux0.11
  	
  ```

+ 在Bochs中编译并运行`test.c`

  ```c
  #include <stdio.h>
  
  int i = 0x12345678;
  int main(void)
  {
      printf("The logical/virtual address of i is 0x%08x", &i);
      fflush(stdout);
      while (i)
          ;
      return 0;
  }
  ```

+ 通过使用 `c`和`ctrl+c`使得显示以下信息。

  ```c
  (0) [0x00fc8031] 000f:00000031 (unk. ctxt): cmp dword ptr ds:0x3004, 0x00000000 ; 833d0430000000
  ```

​		如果显示的下一条指令不是 `cmp ...`（这里指语句以 `cmp` 开头），就用 `n` 命令单步运行		几步，直到停在 `cmp ...`。

​		使用命令 `u /8`，显示从当前位置开始 8 条指令的反汇编代码。

```c
<bochs:3> u /8
10000063: (                    ): cmp dword ptr ds:0x3004, 0x00000000 ; 833d0430000000
1000006a: (                    ): jz .+0x00000004           ; 7404
1000006c: (                    ): jmp .+0xfffffff5          ; ebf5
1000006e: (                    ): add byte ptr ds:[eax], al ; 0000
10000070: (                    ): xor eax, eax              ; 31c0
10000072: (                    ): jmp .+0x00000000          ; eb00
10000074: (                    ): leave                     ; c9
10000075: (                    ): ret                       ; c3
```

这就是 test.c 中从 while 开始一直到 return 的汇编代码。变量 i 保存在 `ds:0x3004` 这个地址，并不停地和 0 进行比较，直到它为 0，才会跳出循环。

##### 2）段表

+ 在终端输入`sreg`

  ```c
  <bochs:4> sreg
  cs:s=0x000f, dl=0x00000002, dh=0x10c0fa00, valid=1
  ds:s=0x0017, dl=0x00003fff, dh=0x10c0f300, valid=3
  ss:s=0x0017, dl=0x00003fff, dh=0x10c0f300, valid=1
  es:s=0x0017, dl=0x00003fff, dh=0x10c0f300, valid=1
  fs:s=0x0017, dl=0x00003fff, dh=0x10c0f300, valid=1
  gs:s=0x0017, dl=0x00003fff, dh=0x10c0f300, valid=1
  ldtr:s=0x0068, dl=0xa2d00068, dh=0x000082fa, valid=1
  tr:s=0x0060, dl=0xa2e80068, dh=0x00008bfa, valid=1
  gdtr:base=0x00005cb8, limit=0x7ff
  idtr:base=0x000054b8, limit=0x7ff
  ```

  可以看到 ldtr 的值是 `0x0068=0000000001101000`（二进制）**段选择符**，**低三位为TI和RPL**表示 LDT 表存放在 GDT 表的 1101（二进制）=13（十进制）号位置（每位数据的意义参考后文叙述的段选择子）。（**从0开始编号**）

  而 GDT 的位置已经由 gdtr 明确给出，在物理地址的 `0x00005cb8`。<font color=#10aa>**GDTR一共48位，低16位为表的界限，高32位为基地址**。</font>

+ 用 `xp /32w 0x00005cb8` 查看从该地址开始，32 个字的内容，及 GDT 表的前 16 项，如下：（32位地址，一个字32位，即4个字节）。

  ```c
  <bochs:7> xp /32w 0x00005cb8
  [bochs]:
  0x00005cb8 <bogus+       0>:	0x00000000	0x00000000	0x00000fff	0x00c09a00
  0x00005cc8 <bogus+      16>:	0x00000fff	0x00c09300	0x00000000	0x00000000
  0x00005cd8 <bogus+      32>:	0xb8880068	0x00008901	0xb8700068	0x00008201
  0x00005ce8 <bogus+      48>:	0xf2e80068	0x000089ff	0xf2d00068	0x000082ff
  0x00005cf8 <bogus+      64>:	0xd2e80068	0x000089ff	0xd2d00068	0x000082ff
  0x00005d08 <bogus+      80>:	0x12e80068	0x000089fc	0x12d00068	0x000082fc
  0x00005d18 <bogus+      96>:	0x52e80068	0x00008bfd	0x52d00068	0x000082fd
  0x00005d28 <bogus+     112>:	0xe2e80068	0x000089f8	0xe2d00068	0x000082f8
  
  
  ```

  GDT 表中的每一项占 64 位（8 个字节），所以我们要查找的项的地址是 `0x00005cb8+13*8`。

  输入 `xp /2w 0x00005cb8+13*8`，得到：

  ```c
  <bochs:8> xp /2w 0x00005cb8+13*8
  [bochs]:
  0x00005d20 <bogus+       0>:	0x52d00068	0x000082fd
  ```

  “0x**52d0**0068 0x**00**0082**fd**” 将其中的加粗数字组合为“0x00fd52d0”，这就是 **LDT 表的物理地址**（低32位的 0x52d00068中的高16位➕高32位的0x000082fd的低8位和高8位）**<font color=green>段描述符为64位，其中包括32位基地址和20位段界限以及其他属性，其中若粒度位G=0，则段界限以字节为单位；若G=1，则段界限以4KB为单位，此时段的扩展范围为4KB~4GB</font>。**是段界限的单位。

  `xp /8w 0x00fd52d0`，得到：

  ```c
  <bochs:9> xp /8w 0x00fd52d0
  [bochs]:
  0x00fd52d0 <bogus+       0>:	0x00000000	0x00000000	0x00000002	0x10c0fa00
  0x00fd52e0 <bogus+      16>:	0x00003fff	0x10c0f300	0x00000000	0x00fd6000
  ```

  这就是 LDT 表的前 4 项内容了。

+ 看看 ds 选择子的内容，还是用 `sreg` 命令：

  ds 的值是 `0x0017=0000000000010111`（二进制），所以 RPL=11，可见是在最低的特权级（因为在应用程序中执行），TI=1，表示查找 LDT 表，索引值为 10（二进制）= 2（十进制），表示找 LDT 表中的第 3 个段描述符（从 0 开始编号）。

+ 费了很大的劲，实际上我们需要的只有段基址一项数据，即段描述符 “0x**0000**3fff 0x**10**c0f3**00**” 中加粗部分组合成的 “0x10000000”。这就是 ds 段在线性地址空间中的起始地址。用同样的方法也可以算算其它段的基址，都是这个数。

  LDT 和 GDT 的结构一样，每项占 8 个字节。所以第 3 项 `0x00003fff 0x10c0f300`（上一步骤的最后一个输出结果中） 就是搜寻好久的 ds 的段描述符了。

+ 费了很大的劲，实际上我们需要的只有段基址一项数据，即段描述符 “0x00003fff 0x10c0f300” 中加粗部分组合成的 “0x10000000”。这就是 ds 段在线性地址空间中的起始地址。用同样的方法也可以算算其它段的基址，都是这个数。

  段基址+段内偏移，就是线性地址了。所以 ds:0x3004 的线性地址就是： 

  ```c
  0x10000000 + 0x3004 = 0x10003004
  ```

  用 `calc ds:0x3004` 命令可以验证这个结果。

+ 线性地址到物理地址转换

  首先需要算出线性地址中的页目录号、页表号和页内偏移，它们分别对应了 32 位线性地址的 10 位 + 10 位 + 12 位，所以 0x10003004 的页目录号是 64，页号 3，页内偏移是 4。

  IA-32 下，页目录表的位置由 CR3 寄存器指引。“creg”命令可以看到：

  ```c
  <bochs:11> creg
  CR0=0x8000001b: PG cd nw ac wp ne ET TS em MP PE
  CR2=page fault laddr=0x10002fac
  CR3=0x00000000
      PCD=page-level cache disable=0
      PWT=page-level writes transparent=0
  CR4=0x00000000: osxmmexcpt osfxsr pce pge mce pae pse de tsd pvi vme
  ```

  说明页目录表的基址为 0。![page](http://image.amerny.top/img/pc/page.png)

  页目录表和页表中的内容很简单，是 1024 个 32 位（正好是 4K）数。这 32 位中前 20 位是物理页框号，后面是一些属性信息（其中最重要的是最后一位 P）。其中第 65 个页目录项就是我们要找的内容，用“`xp /w 0+64*4`”查看：

  ```c
  <bochs:14> xp /w 0+64*4
  [bochs]:
  0x00000100 <bogus+       0>:	0x00fa5027
  ```

  其中的 027 是属性，显然 P=1。

  页表所在物理页框号为 `0x00fa5`，即页表在物理内存的 `0x00fa5000` 位置。从该位置开始查找 3 号页表项，得到（`xp /w 0x00fa5000+3*4`）：

  ```c
  <bochs:15> xp /w 0x00fa5000+3*4
  [bochs]:
  0x00fa500c <bogus+       0>:	0x00fa2067
  ```

  ![transform](http://image.amerny.top/img/pc/transform.png)

+ 物理地址

  线性地址 `0x10003004` 对应的物理页框号为 0x00fa2，和页内偏移 0x004 接到一起，得到 `0x00fa2004`，这就是变量 i 的物理地址。可以通过两种方法验证。

  第一种方法是用命令 `page 0x10003004`，可以得到信息：

  ```c
  <bochs:16> page 0x10003004
  linear page 0x10003000 maps to physical page 0x00fa2000
  ```

  第二种方法是用命令 `xp /w 0x00fa7004`，可以看到：

  ```c
  <bochs:20> xp /w 0x00fa2004
  [bochs]:
  0x00fa2004 <bogus+       0>:	0x12345678
  ```

  现在，通过直接修改内存来改变 i 的值为 0，命令是： `setpmem 0x00fa7004 4 0`，表示从 0x00fa7004 地址开始的 4 个字节都设为 0。然后再用“c”命令继续 Bochs 的运行，可以看到 test 退出了，说明 i 的修改成功了，此项实验结束。

  这项实验主要是加深对地址转换的理解以及一些细节掌握，建议这块理解不清楚的可以跟着做一下

#### 2. 基于共享内存的生产者—消费者程序

##### 1）编写`shm.c`

```c
#include <sys/shm.h>
#include <linux/kernel.h>
#include <error.h>
#include <linux/mm.h>
#include <linux/sched.h>

shm_t shms[SHM_SIZE] = {{0, 0, 0}};

/**
 * 该文件用于实现对共享物理内存空间的操作
 * 1.创建 
 * 2.返回逻辑地址，以便于生产者-消费者读写 
*/

/**
 * 该系统调用会新建或者打开一页物理内存作为共享内存，返回该共享内存的shmid，
 * 即该页共享内存在操作系统中的标识，如果多个进程使用相同的key调用shmget()，
 * 则这些进程就会获得相同的shmid，即得到了同一块共享内存的标识。
 * 在shmget()实现时，
 * 如果key所对应的内存已经建立，直接返回shmid，否则新建然后再返回。
 * 如果size超过一页内存大小，返回-1，并置errno为EINVAL。
 * 如果系统没有空闲内存了，返回-1，并置errno为ENOMEM。
 * 参数shmflg在本次实验中可以直接忽略。
*/
int shmget(key_t key, size_t size/*, int shmflg*/)
{
    int i;

    //得到原来已经创建好的共享内存(在生产者已经创建好了的共享内存，消费者拿着相同的key对应的是同一块内存)
    for(i = 0; i < SHMS_SIZE; i++)
    {
        if(shms[i].key == key)
        {
            return i;
        }
    }

    //创建新的共享内存
    for(i = 0; i < SHMS_SIZE; i++);
    if(i == SHMS_SIZE)
    {
        printk("no shm place");
        errno = ENOMEM;
        return -1;
    }
    if(size > PAGE_SIZE) 
    {
        errno = EINVAL;
        return -1;
    }
    shms[i].size = size;
    shms[i].key = key;
    if(!(shms[i].addr = get_free_page()))
    {
        return ENOMEM;
    }
    return i;
}

/**
 * 该系统调用会将shmid对应的共享内存页面映射到当前进程的虚拟地址空间中，并返回一个逻辑地址p，
 * 调用进程可以读写逻辑地址p来读写这一页共享内存。
 * 两个进程都调用shmat可以关联到同一页物理内存上，此时两个进程读写p指针就是在读写同一页内存，
 * 从而实现了基于共享内存的进程间通信。
 * 如果shmid不正确，返回-1，并置errno为EINVAL。
 * 参数shmaddr和shmflg在本次实验中可以直接忽略。
*/
void * shmat(int shmid/*, const void * shmaddr, int shmflg*/);
{
    unsigned long logical_addr;

    if(shmid < 0 || shmid >= SHMS_SIZE)
    {
        //shmid对应的物理内存不存在
        errno = EINVAL;
        return -1;
    }
    logival_addr = current->brk;
    current->brk += PAGE_SIZE;
    if(!put_page(shms[shmid].addr,current->start_code+logival_addr))
    {
        //分页分不出来
        errno = EINVAL;
        return -1;
    }

    //返回共享物理空间对应的逻辑地址
    return (void)*logival_addr;
}

```

##### 2）编写`produce.c`和`consumer.c`

```c
#include <unistd.h>
#include <syscall.h>
#include <sys/shm.h>
#include <semaphore.h>
#include <fcntl.h>
#include <stdio.h>

#define BUF_SIZE 10
#define COUNT 500
#define KEY 183
#define SHM_SIZE (BUF_SIZE+1)*sizeof(short)

int main(int argc,char ** argv)
{
    int pid;//该进程的id
    unsigned short count = 0;//生产资源的个数
    int shm_id;//共享物理内存空间的id
    short *shmp;//操作共享内存的逻辑地址
    sem_t *empty;//三个实现进程间的同步
    sem_t *full;
    sem_t *mutex;

    //关闭原来的信号量
    sem_unlink("empty");
    sum_unlink("full");
    sum_unlink("mutex");

    //新打开三个信号量
    empty = sem_open("empty",0_CREAT|O_EXCL,0666,10);
    full = sem_open("full",0_CREAT|O_EXCL,0666,0);
    mutex = sem_open("mutex",0_CREAT|O_EXCL,0666,1);
    if(empty == SEM_FAILED || full == SEM_FAILED || mutex == SEM_FAILED)
    {
        //申请信号量失败
        printf("sem_open error!\n");
        return -1;
    }

    //使用KEY值申请一块共享物理内存
    shm_id = shmget(KEY,SHM_SIZE,IPC_CREAT|0666);
    if(shm_id == -1)
    {
        //申请共享内存失败
        printf("shmget error!\n");
        return -1;
    }
    shmp = (short*)shmat(shm_id,NULL,0);//返回共享物理内存的逻辑地址


    pid = syscall(SYS_getpid);//得到进程的pid
    
    //生产者生产出资源
    while(count <= COUNT)
    {
        sem_wait(empty);//P(empty)
        sem_wait(mutex);//P(mutex)

        printf("Producer 1 process %d : %d\n",pid,count);
        fflush(stdout);
        *(shmp++) = count++;
        if(!(count % BUF_SIZE))
        {
            shmp -= 10;
        }          
        
        sem_post(mutex);//V(mutex)
        sem_post(full);//V(full)
    }
    return 0;
}

```

---

```c
#include <unistd.h>
#include <syscall.h>
#include <sys/shm.h>
#include <semaphore.h>
#include <fcntl.h>
#include <stdio.h>

#define BUF_SIZE 10
#define KEY 183

int main()
{
    int pid;
    int shm_id;
    short *shmp;
    short *index;
    sem_t *empty;
    sem_t *full;
    sem_t *mutex;

    shm_id = shmget(KEY,0,0);//使用和生产者同一个KEY值，会返回同一个shm_id(指向同一个内存空间)
    if(shm_id == -1)
    {
        //申请共享内存失败
        printf("shmget error!\n");
        return -1;
    }
    shmp = (short*)shmat(shm_id,NULL,0);//返回共享物理内存的逻辑地址
    index = shmp + BUF_SIZE;
    *index = 0;

    //打开生产者那里创建的三个信号量
    empty = sem_open("empty",0);
    full = sem_open("full",0);
    mutex = sem_open("mutex",0);
    if(empty == SEM_FAILED || full == SEM_FAILED || mutex == SEM_FAILED)
    {
        //申请信号量失败
        printf("sem_open error!\n");
        return -1;
    }

    if(!sysvall(SYS_fork))
    {
        pid = syscall(SYS_getpid);//得到进程的pid

        //消费者1开始消费资源
        while(1)
        {
            sem_wait(full);//P(full)
            sem_wait(mutex);//P(mutex)

            printf("Consumer 1 process %d : %d\n",pid,shem[*index]);
            fflush(stdout);
            if(*index == 9)
            {
                *index = 0;
            }
            else
            {
                (*index)++;
            }

            sem_post(mutex);//V(mutex)
            sem_post(empty);//V(empry)
        }
        return 0;
    }

    if(!sysvall(SYS_fork))
    {
        pid = syscall(SYS_getpid);//得到进程的pid

        //消费者2开始消费资源
        while(1)
        {
            sem_wait(full);
            sem_wait(mutex);

            printf("Consumer 2 process %d : %d\n",pid,shem[*index]);
            fflush(stdout);
            if(*index == 9)
            {
                *index = 0;
            }
            else
            {
                (*index)++;
            }

            sem_post(mutex);
            sem_post(empty);
        }
        return 0;
    }

    if(!sysvall(SYS_fork))
    {
        pid = syscall(SYS_getpid);//得到进程的pid

        //消费者3开始消费资源
        while(1)
        {
            sem_wait(full);
            sem_wait(mutex);

            printf("Consumer 3 process %d : %d\n",pid,shem[*index]);
            fflush(stdout);
            if(*index == 9)
            {
                *index = 0;
            }
            else
            {
                (*index)++;
            }

            sem_post(mutex);
            sem_post(empty);
        }
        return 0;
    }
    return 0;
}

```



>​                         最后就是编写运行和测试了。
