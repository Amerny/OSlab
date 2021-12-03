

# 					操作系统试验（哈工大）

[TOC]

## ⛷信号量的实现和应用

### 🤩一、基本内容

- 在 Ubuntu 下编写程序，用信号量解决生产者——消费者问题

  编写应用程序“pc.c”，完成下面的功能：

  - 建立一个生产者进程，N 个消费者进程（N>1）；
  - 用文件建立一个共享缓冲区；
  - 生产者进程依次向缓冲区写入整数 0,1,2,...,M，M>=500；
  - 消费者进程从缓冲区读数，每次读一个，并将读出的数字从缓冲区删除，然后将本进程 ID 和 + 数字输出到标准输出；
  - 缓冲区同时最多只能保存 10 个数。

  `pc.c` 中将会用到 `sem_open()`、`sem_close()`、`sem_wait()` 和 `sem_post()` 等信号量相关的系统调用，请查阅相关文档。

- 在 0.11 中实现信号量，用生产者—消费者程序检验之。

  实现一套山寨版的完全符合 POSIX 规范的信号量，它的函数原型和标准并不完全相同，而且只包含如下系统调用：

  ```c
  //创建一个信号量，或打开一个已经存在的信号量
  sem_t *sem_open(const char *name, unsigned int value);
  
  //信号量的P原子操作（检查信号量是不是为负值，如果是，则停下来睡眠等待，如果不是，则向下执行，0成功
  int sem_wait(sem_t *sem);
  
  //信号量的V原子操作（检查信号量的值是不是为0，如果是，表示有进程在睡眠等待，则唤醒队首进程，如果不是，   //向下执行）0 成功。
  int sem_post(sem_t *sem);
  
  //删除名为name的信号量。
  int sem_unlink(const char *name);
  ```

  - `sem_open()`的功能是创建一个信号量，或打开一个已经存在的信号量。
    - `sem_t` 是信号量类型，根据实现的需要自定义。
    - `name` 是信号量的名字。不同的进程可以通过提供同样的 name 而共享同一个信号量。如果该信号量不存在，就创建新的名为 name 的信号量；如果存在，就打开已经存在的名为 name 的信号量。
    - `value` 是信号量的初值，仅当新建信号量时，此参数才有效，其余情况下它被忽略。当成功时，返回值是该信号量的唯一标识（比如，在内核的地址、ID 等），由另两个系统调用使用。如失败，返回值是 NULL。
  - `sem_wait()` 就是信号量的 P 原子操作。如果继续运行的条件不满足，则令调用进程等待在信号量 sem 上。返回 0 表示成功，返回 -1 表示失败。
  - `sem_post()` 就是信号量的 V 原子操作。如果有等待 sem 的进程，它会唤醒其中的一个。返回 0 表示成功，返回 -1 表示失败。
  - `sem_unlink()` 的功能是删除名为 name 的信号量。返回 0 表示成功，返回 -1 表示失败。

  <font color=#00f0ffc>**在 `kernel` 目录下新建 `sem.c` 文件实现如上功能。然后将 pc.c 从 Ubuntu 移植到 0.11 下，测试自己实现的信号量。**</font>

### 🤩二、生产者-消费者问题

#### 1. 基本结构

```c
Producer()
{
    // 生产一个产品 item;

    // 空闲缓存资源
    P(Empty);

    // 互斥信号量
    P(Mutex);

    // 将item放到空闲缓存中;
    V(Mutex);

    // 产品资源
    V(Full);
}

Consumer()
{
    P(Full);
    P(Mutex);

    //从缓存区取出一个赋值给item;
    V(Mutex);

    // 消费产品item;
    V(Empty);
}
```

#### 2. 共享文件

在 Linux 下使用 C 语言，可以通过三种方法进行**文件的读写**：

- 使用标准 C 的 `fopen()`、`fread()`、`fwrite()`、`fseek()` 和 `fclose()` 等；

- 使用系统调用 `open()`、`read()`、`write()`、`lseek()` 和 `close()` 等；

- 通过内存镜像文件，使用 `mmap()` 系统调用。

  在 Linux 0.11 上只能使用前两种方法。

`fork()` 调用成功后，子进程会继承父进程拥有的大多数资源，包括父进程打开的文件。所以子进程可以直接使用父进程创建的文件指针/描述符/句柄，访问的是与父进程相同的文件。

使用**标准 C 的文件操作函数要注意，它们使用的是进程空间内的文件缓冲区，父进程和子进程之间不共享这个缓冲区。因此，任何一个进程做完写操作后，必须 `fflush()` 一下，将数据强制更新到磁盘，其它进程才能读到所需数据。**

建议直接使用系统调用进行文件操作。

#### 3. 文件操作（c语言）

1. `fopen(char *filename,char *type)`

   \*filename是要打开文件的文件名指针，一般用**双引号括起来的文件名表示**，也可使用双反斜杠隔开的路径名。而*type参数表示了对打开文件的操作方式。其可采用的**操作方式**如下： <u>方式 含义 "r" 打开，只读； "w" 打开，文件指针指到头，只写； "a" 打开，指向文件尾，在已存在文件中追加； "rb" 打开一个二进制文件，只读； "wb" 打开一个二进制文件，只写； "ab" 打开一个二进制文件，进行追加 ；"r+" 以读/写方式打开一个已存在的文件； "w+" 以读/写方式建立一个新的文本文件 ；"a+" 以读/写方式打开一个文件文件进行追加 ；"rb+" 以读/写方式打开一个二进制文件； "wb+" 以读/写方式建立一个新的二进制文件 ；"ab+" 以读/写方式打开一个二进制文件进行追加</u> ；当用fopen()成功的打开一个文件时，该函数将返回一个FILE指针，如果文件打开失败，将返回一个NULL指针。

2. `fclose()`

   关闭文件函数，例如 `fclose(FILE *stream); `

3. 读写文件字符的函数

   ```c
   int fgetc(FILE *stream);
   
   int getchar(void);
   
   int fputc(int ch,FILE *stream);
   
   int putchar(int ch);
   
   int getc(FILE *stream);
   
   int putc(int ch,FILE *stream);
   ```

4. `int fread(void *ptr,int size,int nitems,FILE *stream)`

   从流指针指定的文件中读取nitems个数据项，每个数据项的长度为size个字节，读取的nitems数据项存入由ptr指针指向的内存缓冲区中，在执行fread()函数时，文件指针随着读取的字节数而向后移动，最后移动结束的位置等于实际读出的字节数。该函数执行结束后，将返回实际读出的数据项数，这个数据项数不一定等于设置的nitems，因为若文件中没有足够的数据项，或读中间出错，都会导致返回的数据项数少于设置的nitems。当返回数不等于nitems时，可以用`feof()`或`ferror()`函数进行检查。

5. `int fwrite(void *ptr,int size,int nitems,FILE *stream)`

   fwrite()函数从ptr指向的缓冲区中取出长度为size字节的nitems个数据项，写入到流指针stream指向的文件中，执行该操作后，文件指针将向后移动，移动的字节数等于写入文件的字节数目。该函数操作完成后，也将返回写入的数据项数。
   
   [可以看下这篇博客。](https://wangdoc.com/clang/file.html)

### 🤩三、信号量

#### 1. 信号量组成

+ 需要有一个整形变量value，用作进程同步。

+ 需要有一个PCB指针，指向睡眠的进程队列。

+ 需要有一个名字来表示这个结构的信号量。

同时，由于该value的值是所有进程都可以看到和访问的共享变量，所以必须在内核中定义；同样，这个名字的信号量也是可供所有进程访问的，必须在内核中定义；同时，又要操作内核中的数据结构：进程控制块PCB，所以信号量一定要在内核中定义，而且必须是全局变量。由于信号量要定义在内核中，所以和信号量相关的操作函数也必须做成系统调用，还是那句话：**系统调用是应用程序访问内核的唯一方法。**

#### 2. 内核信号量代码实现

##### 1）编写 `sem.h`，放在 `include/linux`中

根据 `sem_t *sem_open(const char *name, unsigned int value)`，需要三个变量

```c
#ifndef _SEM_H
#define _SEM_H

#include <linux/sched.h>		//进程调度

#define SEMTABLE_LEN    20
#define SEM_NAME_LEN    20


typedef struct semaphore{
    char name[SEM_NAME_LEN];		//信号量的name
    int value;									//信号量的初值
    struct task_struct *queue;	//指向睡眠的进程队列
} sem_t;
extern sem_t semtable[SEMTABLE_LEN];

#endif
```

由于`sem_open()`的第一个参数name，传入的是应用程序所在地址空间的逻辑地址，在内核中如果直接访问这个地址，访问到的是内核空间中的数据，不会是用户空间的。所以要用`get_fs_byte()`函数获取用户空间的数据。`get_fs_byte()`函数的功能是获得一个字节的用户空间中的数据。同样，`sem_unlink()`函数的参数name也要进行相同的处理。

编写之前看下队列的形成

```c
void sleep_on(struct task_struct **p)	//这里是二级指针，因为里面是需要修改*p的，而c语言是值传递
{
    struct task_struct *tmp;

    if (!p)
        return;
    if (current == &(init_task.task))
        panic("task[0] trying to sleep");
    tmp = *p;					//*p是等待队列头指针
    *p = current;			//当前进程睡眠到等待队列的头部
    current->state = TASK_UNINTERRUPTIBLE;
    schedule();
    if (tmp)
        tmp->state=0;
}

/*
 *从中我们可以看到唤醒函数wake_up()负责唤醒的是等待队列队首的进程。
 *唤醒之后，从schedule中退出执行 if语句，会将接下来的进程唤醒。
 *只要执行一次wake_up()操作，就可以依次将所有等待在信号量sem处的睡眠进程唤醒。
 */
void wake_up(struct task_struct **p)
{
    if (p && *p) {
        (**p).state=0;
        *p=NULL;
    }
}
```

##### 2）编写 `sem_wait()`

```c
int sys_sem_wait(sem_t *sem){
  cli();		//关中断，确保原子操作
  sem->value--;		//相当于 P操作，-1
 	while(sem->value<0)		// <0，当前进程睡眠在等待队列头
    sleep_on(&(sem->queue))
  
  sti();		//开中断
}
```

<font color=red>注意：</font>根据网上的博客看的这个代码有问题，比如现在有这样一种情况，有一个信号量初值为1，此时该进程正在对文件操作，按照上述代码来说，此时 `sem->value=0`，假如现在有另外两个进程 P1和P2，P1试图读写，此时`sem->value=-1`，P1进入等待队列头，P2试图读写，此时`sem->value=-2`，P2进入等待队列头，原来进程结束，置`sem->value=-1`唤醒P2，好像还是不能读写，然后事实是文件已经空闲了。所以这就是问题所在。

必须是`while`循环，因此当生产者生产一个数的时候不应该是把所有进程唤醒，只有while条件判断失败之后才会跳出循环。 

改动代码如下

```c
int sys_sem_wait(sem_t *sem){
  cli();		//关中断，确保原子操作
  
 	while(sem->value<0)		// <0，当前进程睡眠在等待队列头
    sleep_on(&(sem->queue));
    
	sem->value--;				// P操作， -1，等你能活着出来再使用
  sti();		//开中断
  return 0;
}
```

##### 3)编写 `sem_post()`

同理，`sem_wait()`是先睡眠，后-1，那么`sem_post()`需要先+1，后唤醒

```c
int sys_sem_post(sem_t *sem){
  cli();
  sem->value++;			//先生产，再唤醒，不能是还没有生产就唤醒。
  if(sem->value<=1)	//唤醒一次全部唤醒，第二次唤醒的时候已经队列空了，参见wake_up()*p=NULL,
    wake_up(&(sem->queue));
  sti();
  return 0;
}
```

借助`wake_up()`函数可以理解，现在缓冲区的value=1，消费者C1消费一个，value=0；消费者C2消费的时候没有东西了，睡眠在这个信号量队列上；消费者C3同理，也睡眠在这个信号量队列上。此时，等待队列如下，C2,C3。然后生产者开始生产，value=1，唤醒C3，此时等待队列头指向为空，C3醒了之后会从之前睡眠的位置重新开始执行，退出 `schedule()`之后将该进程下的 `tmp`头置为可执行状态，而tmp的指向就是C2,因此，<font color=red>判读条件为<=1,并且用if判断就行。</font>

##### 4)编写`sem_open `和`sem_unlink`

将上面两个函数和这个组合在一起组成 `sem.c`放在`kernel`中

```c
#include <linux/sem.h>
#include <linux/sched.h>
#include <unistd.h>
#include <asm/segment.h>
#include <linux/tty.h>
#include <linux/kernel.h>
#include <linux/fdreg.h>
#include <asm/system.h>
#include <asm/io.h>
//#include <string.h>

sem_t semtable[SEMTABLE_LEN];
int cnt = 0;			//信号量的数量

//创建或者返回一个已经存在的信号量
sem_t *sys_sem_open(const char *name,unsigned int value)
{
    char kernelname[100];   /* 应该足够大了 ，内核中的信号量的name */
    int isExist = 0;				//默认不存在
    int i=0;
    int name_cnt=0;					//记录name的长度
    while( get_fs_byte(name+name_cnt) != '\0')		//从用户空间获得字符
    		name_cnt++;
    if(name_cnt>SEM_NAME_LEN)		//超出最大长度
    		return NULL;
    for(i=0;i<name_cnt;i++)
    		kernelname[i]=get_fs_byte(name+i);		//将用户空间获得name拷贝到内核中信号量的名称
    int name_len = strlen(kernelname);		//获得长度
    int sem_name_len =0;		
    sem_t *p=NULL;
    for(i=0;i<cnt;i++)
    {
        sem_name_len = strlen(semtable[i].name);		//
        if(sem_name_len == name_len)
        {
                if( !strcmp(kernelname,semtable[i].name) )	//相等返回0
                {
                    isExist = 1;
                    break;
                }
        }
    }
    if(isExist == 1)		//信号量已经存在
    {
        p=(sem_t*)(&semtable[i]);
        //printk("find previous name!\n");
    }
    else
    {
        i=0;
        for(i=0;i<name_len;i++)
        {
            semtable[cnt].name[i]=kernelname[i];
        }
        semtable[cnt].value = value;
        p=(sem_t*)(&semtable[cnt]);
         //printk("creat name!\n");
        cnt++;
     }
    return p;
}

//信号量 P操作，相当于 -1
int sys_sem_wait(sem_t *sem)
{
    cli();
    while( sem->value <= 0 )        //
        sleep_on(&(sem->queue));    //这两条语句顺序不能颠倒，很重要，是关于互斥信号量能不
    sem->value--;               //能正确工作的！！！
    sti();
    return 0;
}
//信号量 V操作 ，相当于 +1
int sys_sem_post(sem_t *sem)
{
    cli();
    sem->value++;
    if( (sem->value) <= 1)
        wake_up(&(sem->queue));
    sti();
    return 0;
}
//删除信号量
int sys_sem_unlink(const char *name)
{
    char kernelname[100];   /* 应该足够大了 */
    int isExist = 0;
    int i=0;
    int name_cnt=0;
    while( get_fs_byte(name+name_cnt) != '\0')
            name_cnt++;
    if(name_cnt>SEM_NAME_LEN)
            return NULL;
    for(i=0;i<name_cnt;i++)
            kernelname[i]=get_fs_byte(name+i);
    int name_len = strlen(name);
    int sem_name_len =0;
    for(i=0;i<cnt;i++)
    {
        sem_name_len = strlen(semtable[i].name);
        if(sem_name_len == name_len)
        {
                if( !strcmp(kernelname,semtable[i].name))
                {
                        isExist = 1;
                        break;
                }
        }
    }
    if(isExist == 1)
    {
        int tmp=0;
        for(tmp=i;tmp<=cnt;tmp++)
        {
            semtable[tmp]=semtable[tmp+1];
        }
        cnt = cnt-1;
        return 0;
    }
    else
        return -1;
}
```

##### 5）其他调整和修改

+ 程序头文件

  由于**系统调用是借助内嵌汇编_syscall实现的**，而_syscall的内嵌汇编实现是在linux-0.11/include/unistd.h中，所以必须包含`#include <unistd.h>`这个头文件，另外由于_syscall的内嵌汇编实现是包含在一个条件编译里面，所以必须包含这样一个宏定义`#define __LIBRARY__`。

+ 修改 `include/unistd.h`

```c
 #define __NR_sem_open   72  /* !!! */
 #define __NR_sem_wait   73
 #define __NR_sem_post   74
 #define __NR_sem_unlink 75
//类比系统调用实验的操作
```

+ 修改`kernel/system_call.s`

```c
nr_system_calls = 76  //原来值的基础上+4
```

+ 修改`include/linux/sys.h`

```c
extern int sys_sem_open();
extern int sys_sem_wait();
extern int sys_sem_post();
extern int sys_sem_unlink();

fn_ptr sys_call_table[] = { ....,sys_sem_open,sys_sem_wait,sys_sem_post,sys_sem_unlink };
```

+ 修改`kernel/Makefile`

```makefile
### 添加的 sem.o
......
OBJS  = sched.o system_call.o traps.o asm.o fork.o \
panic.o printk.o vsprintf.o sys.o exit.o \
signal.o mktime.o sem.o	
......

### Dependencies:
### 添加如下
sem.s sem.o: sem.c ../include/linux/sem.h ../include/linux/kernel.h \	
../include/unistd.h
......
```

+ 在运行之后，将修改过的 `include/unistd.h`移动到 0.11环境下的`usr/include`中覆盖。

#### 3. 测试程序编写

##### 1）基本要求

>1. 建立一个生产者进程，N个消费者进程（ N>1 ）
>2. 用文件建立一个共享缓冲区
>3. 生产者进程依次向缓冲区写入整数0，1，2，…，M，M>=500
>4. 消费者进程从缓冲区读数，每次读一个，并将读出的数字从缓冲区删除，然后将本进程ID和+ 数字输出到标准输出
>5. 缓冲区同时最多只能保存10个数
>     一种可能的输出效果是：
>     10: 0

##### 2） 文件IO函数

由于要用文件建立一个共享缓冲区，同时生产者要往文件中写数，消费者要从文件中读数，所以要用到open()、read()、write()、lseek()、close()这些文件IO系统调用。
应用程序实现的难点在于，消费者进程每次读一个数，要将读出的数字从缓冲区删除，这几个文件IO系统调用函数中，并没有可以删除一个数字的函数。解决办法是，当消费者进程要从缓冲区读数时，首先调用lseek()系统调用获取到目前文件指针的位置，保存生产者目前写文件的位置。由于被消费者进程读过的数都被删除了，所以同时最多只能保存10个数的缓冲区已有的数，一定是消费者进程未读的，也就是说每次消费者要从缓冲区读数时，要读的数一定是缓冲区的第一个数。这样，让消费者进程每次都从缓冲区读10个数出来，取读出的10个数中的第一个数送标准输出显示，再将后面的9个数再次写入到缓冲区中，这样，就可以做到删除读出的那个数。最后，再调用lseek()系统调用将文件指针定位到之前保存的文件指针减1的位置，这样，生产者进程再次写缓冲区时，也能正确定位删除了一个数字的缓冲区的写位置。

+ `open()`

  ```c
  int open(const char *pathname,int flags[,mode_t mode]);
  ```

  `open()`函数的第一个参数通常为待打开文件的文件路径及名称；第二个参数为文件的访问模式，一般使用定义在函数库fcntl.h中的一组宏来表示，常用如下

  | 编号 | 宏       | 说明                                                |
  | ---- | -------- | --------------------------------------------------- |
  | 1    | O_RDONLY | 以只读方式打开文件                                  |
  | 2    | O_WRONLY | 以只写方式打开文件                                  |
  | 3    | O_RDWR   | 以读写方式打开文件                                  |
  | 4    | O_CREAT  | 创建一个文件并打开，若文件已存在则会出错            |
  | 5    | O_TRUNC  | 当以只写或读写方式成功打开文件时，将文件长度截断为0 |

  ```c
  fi = open(filename, O_CREAT| O_TRUNC| O_WRONLY, 0222);	//返回文件描述符
  ```

+ `read()`

  ```c
  ssize_t read(int fd, void *buf, size_t count);
  ```

  `read()`函数基于文件描述符对文件进行操作，其中第一个参数为从open()函数或creat()函数获取的文件描述符；第二个参数为缓冲区；第三个参数为计划读取的字节数。调用read()函数后，该函数会从文件描述符fd对应的文件中读取count个字节的数据，存储到缓冲区buf中，并重新记录文件偏移量。

+ `write()`

  ```c
  ssize_t write(int fd, const void *buf, size_t count);
  ```

  `write()`函数的第一个参数为文件描述符；第二个参数为需要输出的缓冲区；第三个参数为最大输出字节数。当write()函数调用成功时返回写入的字节数；否则返回-1，并设置errno。

+ `lssek()`

  ```c
  off_t lseek(int fd, off_t offset, int whence);
  ```

  lseek()函数中第一个参数fd为文件描述符；第二个参数offset用于对文件偏移量的设置，该参数值可正可负；第三个参数whence用于控制设置当前文件偏移量的方法，该参数有3个取值：

  (1) 若whence为SEEK_SET，文件偏移量将被设置为offset；

  (2) 若whence为SEEK_CUR，文件偏移量的值将会在当前文件偏移量的基础上加上offset；

  (3) 若whence为SEEK_END，文件偏移量的值将会被设置为文件长度加上offset。

+ `close`

  ```c
  int close(int fd);
  ```

  打开的文件在操作结束后应该主动关闭，Linux系统调用中用于关闭文件的函数为close()函数，该函数的使用方法很简单，只要在函数中传入文件描述符，便可关闭文件。

##### 3）最终代码实现

```c
#define __LIBRARY__
#include <unistd.h>
#include <linux/sem.h>
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <linux/sched.h>

_syscall2(sem_t *,sem_open,const char *,name,unsigned int,value)
_syscall1(int,sem_wait,sem_t *,sem)
_syscall1(int,sem_post,sem_t *,sem)
_syscall1(int,sem_unlink,const char *,name)

const char *FILENAME = "/usr/root/buffer_file";    /* 消费生产的产品存放的缓冲文件的路径 */
const int NR_CONSUMERS = 5;                        /* 消费者的数量 */
const int NR_ITEMS = 50;                        /* 产品的最大量 */
const int BUFFER_SIZE = 10;                        /* 缓冲区大小，表示可同时存在的产品数量 */
sem_t *metux, *full, *empty;                    /* 3个信号量 */
unsigned int item_pro, item_used;                /* 刚生产的产品号；刚消费的产品号 */
int fi, fo;                                        /* 供生产者写入或消费者读取的缓冲文件的句柄 */


int main(int argc, char *argv[])
{
    char *filename;
    int pid;
    int i;

    filename = argc > 1 ? argv[1] : FILENAME;
    /* O_TRUNC 表示：当文件以只读或只写打开时，若文件存在，则将其长度截为0（即清空文件）
     * 0222 和 0444 分别表示文件只写和只读（前面的0是八进制标识）
     */
    fi = open(filename, O_CREAT| O_TRUNC| O_WRONLY, 0222);    /* 以只写方式打开文件给生产者写入产品编号 */
    fo = open(filename, O_TRUNC| O_RDONLY, 0444);            /* 以只读方式打开文件给消费者读出产品编号 */

    metux = sem_open("METUX", 1);    /* 互斥信号量，防止生产消费同时进行 */
    full = sem_open("FULL", 0);        /* 产品剩余信号量，大于0则可消费 */
    empty = sem_open("EMPTY", BUFFER_SIZE);    /* 空信号量，它与产品剩余信号量此消彼长，大于0时生产者才能继续生产 */

    item_pro = 0;

    if ((pid = fork()))    /* 父进程用来执行消费者动作 */
    {
        printf("pid %d:\tproducer created....\n", pid);
        /* printf()输出的信息会先保存到输出缓冲区，并没有马上输出到标准输出（通常为终端控制台）。
         * 为避免偶然因素的影响，我们每次printf()都调用一下stdio.h中的fflush(stdout)
         * 来确保将输出立刻输出到标准输出。
         */
        fflush(stdout);

        while (item_pro <= NR_ITEMS)    /* 生产完所需产品 */
        {
            sem_wait(empty);
            sem_wait(metux);

            /* 生产完一轮产品（文件缓冲区只能容纳BUFFER_SIZE个产品编号）后
             * 将缓冲文件的位置指针重新定位到文件首部。
             */
            if(!(item_pro % BUFFER_SIZE))
                lseek(fi, 0, 0);

            write(fi, (char *) &item_pro, sizeof(item_pro));        /* 写入产品编号 */
            printf("pid %d:\tproduces item %d\n", pid, item_pro);
            fflush(stdout);
            item_pro++;

            sem_post(full);        /* 唤醒消费者进程 */
            sem_post(metux);
        }
    }
    else    /* 子进程来创建消费者 */
    {
        i = NR_CONSUMERS;
        while(i--)
        {
            if(!(pid=fork()))    /* 创建i个消费者进程 */
            {
                pid = getpid();
                printf("pid %d:\tconsumer %d created....\n", pid, NR_CONSUMERS-i);
                fflush(stdout);

                while(1)
                {
                    sem_wait(full);
                    sem_wait(metux);

                    /* read()读到文件末尾时返回0，将文件的位置指针重新定位到文件首部 */
                    if(!read(fo, (char *)&item_used, sizeof(item_used)))
                    {
                        lseek(fo, 0, 0);
                        read(fo, (char *)&item_used, sizeof(item_used));
                    }

                    printf("pid %d:\tconsumer %d consumes item %d\n", pid, NR_CONSUMERS-i+1, item_used);
                    fflush(stdout);

                    sem_post(empty);    /* 唤醒生产者进程 */
                    sem_post(metux);

                    if(item_used == NR_ITEMS)    /* 如果已经消费完最后一个商品，则结束 */
                        goto OK;
                }
            }
        }
    }
OK:
    close(fi);
    close(fo);
    return 0;
}
```

#### 4. 结果运行

