# 				操作系统试验（哈工大）



### ⛷三、进程运行轨迹的跟踪与统计

#### 1. 具体步骤

1. 修改`main.c`

   ```c
    // 下面过程通过在堆栈中设置的参数，利用中断返回指令启动任务0执行。
   	move_to_user_mode();                    // 移到用户模式下执行
   
   	/***************添加开始***************/
   	setup((void *) &drive_info);
   
   	// 建立文件描述符0和/dev/tty0的关联
   	(void) open("/dev/tty0",O_RDWR,0);
   
   	//文件描述符1也和/dev/tty0关联
   	(void) dup(0);
   
   	// 文件描述符2也和/dev/tty0关联
   	(void) dup(0);
   
   	(void) open("/var/process.log",O_CREAT|O_TRUNC|O_WRONLY,0666);
   
   	/***************添加结束***************/
   
   	if (!fork()) {		/* we count on this going ok */
   		init();                             // 在新建的子进程(任务1)中执行。
   	}
   ```

   目的是更早的记录进程的信息，即在创建子进程init之前就进行文件描述符的建立，并且把log文件的描述符关联到3。

   这样，文件描述符 0、1、2 和 3 就在进程 0 中建立了。根据 `fork()` 的原理，进程 1 会继承这些文件描述符，所以 `init()` 中就不必再 `open()` 它们。此后所有新建的进程都是进程 1 的子孙，也会继承它们。但实际上，`init()` 的后续代码和 `/bin/sh` 都会重新初始化它们。所以只有进程 0 和进程 1 的文件描述符肯定关联着 log 文件，这一点在接下来的写 log 中很重要。

2. 写log文件

   ```c
   #include "linux/sched.h"
   #include "sys/stat.h"
   
   static char logbuf[1024];
   int fprintk(int fd, const char *fmt, ...)
   {
       va_list args;
       int count;
       struct file * file;
       struct m_inode * inode;
   
       va_start(args, fmt);
       count=vsprintf(logbuf, fmt, args);
       va_end(args);
   /* 如果输出到stdout或stderr，直接调用sys_write即可 */
       if (fd < 3)
       {
           __asm__("push %%fs\n\t"
               "push %%ds\n\t"
               "pop %%fs\n\t"
               "pushl %0\n\t"
           /* 注意对于Windows环境来说，是_logbuf,下同 */
               "pushl $logbuf\n\t"
               "pushl %1\n\t"
           /* 注意对于Windows环境来说，是_sys_write,下同 */
               "call sys_write\n\t"
               "addl $8,%%esp\n\t"
               "popl %0\n\t"
               "pop %%fs"
               ::"r" (count),"r" (fd):"ax","cx","dx");
       }
       else
   /* 假定>=3的描述符都与文件关联。事实上，还存在很多其它情况，这里并没有考虑。*/
       {
       /* 从进程0的文件描述符表中得到文件句柄 */
           if (!(file=task[0]->filp[fd]))
               return 0;
           inode=file->f_inode;
   
           __asm__("push %%fs\n\t"
               "push %%ds\n\t"
               "pop %%fs\n\t"
               "pushl %0\n\t"
               "pushl $logbuf\n\t"
               "pushl %1\n\t"
               "pushl %2\n\t"
               "call file_write\n\t"
               "addl $12,%%esp\n\t"
               "popl %0\n\t"
               "pop %%fs"
               ::"r" (count),"r" (file),"r" (inode):"ax","cx","dx");
       }
       return count;
   }
   ```

   该函数放到`kernel/printk.c`中

3. 寻找状态切换点。

   总的来说，Linux 0.11支持四种进程状态的转移：就绪到运行、运行到就绪、运行到睡眠和睡眠到就绪，此外还有新建和退出两种情况。其中就绪与运行间的状态转移是通过schedule()（它亦是调度算法所在）完成的；运行到睡眠依靠的是 `sleep_on()` 和 `interruptible_sleep_on()` ，还有进程主动睡觉的系统调用 `sys_pause()` 和 `sys_waitpid()` ；睡眠到就绪的转移依靠的是`wake_up()`。所以只要在这些函数的适当位置插入适当的处理语句就能完成进程运行轨迹的全面跟踪了。

   

   即进程的刚开始fork的时候和 sleep 的切换点，所以接下来的目的很明确，就是去看fork的代码和sleep的代码，fork在内核中的实现是`kernel/system_call.s`中实现的,查看源码可知主要功能进入了`copy_process`中，它在 `kernel/fork.c`中实现。

   + 所以现在在 `fork.c`中添加以下代码

   ```c
   int copy_process(int nr,……)
   {
       struct task_struct *p;
   		//    ......
      
       set_tss_desc(gdt+(nr<<1)+FIRST_TSS_ENTRY,&(p->tss));
   	  set_ldt_desc(gdt+(nr<<1)+FIRST_LDT_ENTRY,&(p->ldt));
   	  
   	  //需要添加的
   	  fprintk(3,"%ld\t%c\t%ld\n",p->pid,'N',jiffies);
       fprintk(3,"%ld\t%c\t%ld\n",p->pid,'J',jiffies);
       //
     
       p->state = TASK_RUNNING;
       return last_pid;
   }
   ```

   ---

   + 在`kernel/sched.c`中

   ```c
   //调度算法
   void schedule(void)
   {
   	int i,next,c;
   	struct task_struct ** p;
     for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
     if (*p) {
       //.......
       if (((*p)->signal & ~(_BLOCKABLE & (*p)->blocked)) &&
               (*p)->state==TASK_INTERRUPTIBLE){
         					//添加开始
                   fprintk(3,"%ld\t%c\t%ld\n",(*p)->pid,'J',jiffies);
         					//添加结束
                   (*p)->state=TASK_RUNNING;
               }
        }
       //.....
      if(task[next]->pid!=current->pid){
           if(current->state==TASK_RUNNING)
             //添加开始
               fprintk(3,"%ld\t%c\t%ld\n",current->pid,'J',jiffies);
           fprintk(3,"%ld\t%c\t%ld\n",task[next]->pid,'R',jiffies);
        			//添加结束
       }
       switch_to(next);
   }
   
   
   //转换当前任务状态为可中断的等待状态，并重新调度
   int sys_pause(void)
   {
   	current->state = TASK_INTERRUPTIBLE;
     //添加开始
     if(current->state != TASK_INTERRUPTIBLE)
        fprintk(3,"%ld\t%c\t%ld\n",current->pid,'W',jiffies);
     //添加结束
   	schedule();
   	return 0;
   }
   
   
   //不可中断的等待状态
   void sleep_on(struct task_struct **p)
   {
       struct task_struct *tmp;
       ……
       tmp = *p;
       *p = current;
     
     
     	//*********需要添加的地方**********//
     	if(current->state != TASK_UNINTERRUPTIBLE){
           fprintk(3,"%ld\t%c\t%ld\n",(**p).pid,'W',jiffies);    
       }
     	//*********结束位置***************//
     
     
       current->state = TASK_UNINTERRUPTIBLE; 
       schedule();  
       if (tmp){
         	//添加开始
           if(tmp->state!=0)
               fprintk(3,"%ld\t%c\t%ld\n",tmp->pid,'J',jiffies);        
         	//添加结束
           tmp->state=0;
       } 
   }
   
   //可中断的等待状态
   void interruptible_sleep_on(struct task_struct **p)
   {
       struct task_struct *tmp;
   
       if (!p)
           return;
       if (current == &(init_task.task))
           panic("task[0] trying to sleep");
       tmp=*p;
       *p=current;
   repeat:  //添加开始  
       if(current->state != TASK_INTERRUPTIBLE)
           fprintk(3,"%ld\t%c\t%ld\n",current->pid,'W',jiffies);
     			//添加结束
       current->state = TASK_INTERRUPTIBLE;
       schedule();
       if (*p && *p != current) {
         	//添加开始
           if((**p).state!=0)
               fprintk(3,"%ld\t%c\t%ld\n",(**p).pid,'J',jiffies);
         	//添加结束
           (**p).state=0;
           goto repeat;
       }
       *p=NULL;
       if (tmp){
         	//添加开始
           if(tmp->state!=0)
               fprintk(3,"%ld\t%c\t%ld\n",tmp->pid,'J',jiffies);        
         	//添加结束
           tmp->state=0;
       }
   }
   
   
   // 因此唤醒的是最后进入等待队列的任务。
   void wake_up(struct task_struct **p)
   {
   	if (p && *p) {
       //添加开始
       if((**p).state!=0)
               fprintk(3,"%ld\t%c\t%ld\n",(**p).pid,'J',jiffies);
       //添加结束
   		(**p).state=0;          // 置为就绪(可运行)状态TASK_RUNNING.
   		*p=NULL;
   	}
   }
   
   
   ```

   + 在`kernel/exit.c`中

   ```c
   //// 程序退出处理函数。
   // 该函数将把当前进程置为TASK_ZOMBIE状态，然后去执行调度函数schedule()，不再返回。
   // 参数code是退出状态码，或称为错误码。
   int do_exit(long code){
    //.......
   	tell_father(current->father);
     //添加开始
     fprintk(3,"%ld\t%c\t%ld\n",current->pid,'E',jiffies);
     //添加结束
   	schedule();
     
   }
   
   //系统调用waipid().挂起当前进程
   int sys_waitpid(pid_t pid,unsigned long * stat_addr, int options)
   {
     //.....
     if (flag) {
           if (options & WNOHANG)
               return 0;
       		//添加开始
           if(current->state!=TASK_INTERRUPTIBLE)
               fprintk(3,"%ld\t%c\t%ld\n",current->pid,'W',jiffies);
       		//添加结束
           current->state=TASK_INTERRUPTIBLE;
           schedule();
           if (!(current->signal &= ~(1<<(SIGCHLD-1))))
               goto repeat;
           else
               return -EINTR;
       }
   }
   ```

   

4. 测试

   交换文件，将process.c拷贝到/hdc/usr/root/中，接着卸载hdc。使用以下命令

   ```shell
   cd ~/oslab
   sudo ./mount-hdc
   
   #拷贝到 /hdc/usr/root
   cp ./testcode/process.c ./hdc/usr/root
   
   #退出挂载
   sudo umount hdc
   ```

   ```c
   #include <stdio.h>
   #include <unistd.h>
   #include <time.h>
   #include <sys/times.h>
   #include <sys/wait.h>
   #include <sys/types.h>
   
   #define HZ	100
   
   void cpuio_bound(int last, int cpu_time, int io_time);
   
   int main(int argc, char * argv[])
   {
   	pid_t father,son1,son2,son3,tmp1,tmp2,tmp3;
   	tmp1=fork();
   	if(tmp1==0)			/* son1 */
   	{
   		son1=getpid();
   		printf("The son1's pid:%d\n",son1);
   		printf("I am son1\n");
   		cpuio_bound(10, 3, 2);
   		printf("Son1 is finished\n");
   	}
   	else if(tmp1>0)
   	{
   		son1=tmp1;
   		tmp2=fork();
   		if(tmp2==0)		/* son2 */
   		{
   			son2=getpid();
   			printf("The son2's pid:%d\n",son2);
   			printf("I am son2\n");
   			cpuio_bound(5, 1, 2);
   			printf("Son2 is finished\n");
   		}
   		else if(tmp2>0)		/* father */
   		{
   			son2=tmp2;
   			father=getpid();
   			printf("The father get son1's pid:%d\n",tmp1);
   			printf("The father get son2's pid:%d\n",tmp2);
   			wait((int *)NULL);
   			wait((int *)NULL);
   			printf("Now is the father's pid:%d\n",father);
   		}
   		else
   			printf("Creat son2 failed\n");
   	}
   	else
   		printf("Creat son1 failed\n");
   	return 0;
   }
   
   /*
    * 此函数按照参数占用CPU和I/O时间
    * last: 函数实际占用CPU和I/O的总时间，不含在就绪队列中的时间，>=0是必须的
    * cpu_time: 一次连续占用CPU的时间，>=0是必须的
    * io_time: 一次I/O消耗的时间，>=0是必须的
    * 如果last > cpu_time + io_time，则往复多次占用CPU和I/O
    * 所有时间的单位为秒
    */
   void cpuio_bound(int last, int cpu_time, int io_time)
   {
   	struct tms start_time, current_time;
   	clock_t utime, stime;
   	int sleep_time;
   
   	while (last > 0)
   	{
   		/* CPU Burst */
   		times(&start_time);
   		/* 其实只有t.tms_utime才是真正的CPU时间。但我们是在模拟一个
   		 * 只在用户状态运行的CPU大户，就像“for(;;);”。所以把t.tms_stime
   		 * 加上很合理。*/
   		do
   		{
   			times(&current_time);
   			utime = current_time.tms_utime - start_time.tms_utime;
   			stime = current_time.tms_stime - start_time.tms_stime;
   		} while ( ( (utime + stime) / HZ )  < cpu_time );
   		last -= cpu_time;
   
   		if (last <= 0 )
   			break;
   
   		/* IO Burst */
   		/* 用sleep(1)模拟1秒钟的I/O操作 */
   		sleep_time=0;
   		while (sleep_time < io_time)
   		{
   			sleep(1);
   			sleep_time++;
   		}
   		last -= sleep_time;
   	}
   }
   
   
   ```
   
   



#### 2. 一些思考☺️

1. 在`kernel/sched.c`中，包含了核心的进程切换和调度程序。先看 `schedule()`，主要任务就是，找到下一个可执行的任务并且`switch_to()`。进行这个操作之前还对定时任务进行了处理

```c
// 从任务数组中最后一个任务开始循环检测alarm。在循环时跳过空指针项。
	for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
		if (*p) {
            // 如果设置过任务的定时值alarm，并且已经过期(alarm<jiffies)，则在
            // 信号位图中置SIGALRM信号，即向任务发送SIGALARM信号。然后清alarm。
            // 该信号的默认操作是终止进程。jiffies是系统从开机开始算起的滴答数(10ms/滴答)。
			if ((*p)->alarm && (*p)->alarm < jiffies) {
					(*p)->signal |= (1<<(SIGALRM-1));
					(*p)->alarm = 0;
				}
            // 如果信号位图中除被阻塞的信号外还有其他信号，并且任务处于可中断状态，则
            // 置任务为就绪状态。其中'~(_BLOCKABLE & (*p)->blocked)'用于忽略被阻塞的信号，但
            // SIGKILL 和SIGSTOP不能呗阻塞。
			if (((*p)->signal & ~(_BLOCKABLE & (*p)->blocked)) &&
			(*p)->state==TASK_INTERRUPTIBLE){
				//这里添加状态切换点
				(*p)->state=TASK_RUNNING;

			}
		}
//.......
```



找出下一个可运行的任务，主要操作如下

```c
		// 这段代码也是从任务数组的最后一个任务开始循环处理，并跳过不含任务的数组槽。比较
  	// 每个就绪状态任务的counter(任务运行时间的递减滴答计数)值，哪一个值大，运行时间还
    // 不长，next就值向哪个的任务号。
{
  i = NR_TASKS;
  next = 0;
  p = &task[i];
  c = -1;
  while (--i) {
        if (!*--p)
          continue;
        if ((*p)->state == TASK_RUNNING && (*p)->counter > c)
          c = (*p)->counter, next = i;
      }
    // 如果比较得出有counter值不等于0的结果，或者系统中没有一个可运行的任务存在(此时c
    // 仍然为-1，next=0),则退出while(1)_的循环，执行switch任务切换操作。否则就根据每个
    // 任务的优先权值，更新每一个任务的counter值，然后回到while(1)循环。counter值的计算
    // 方式counter＝counter/2 + priority.注意：这里计算过程不考虑进程的状态。
      if (c) break;
      for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
        if (*p)
          (*p)->counter = ((*p)->counter >> 1) +
              (*p)->priority;
}

  // 用下面的宏把当前任务指针current指向任务号Next的任务，并切换到该任务中运行。上面Next
  // 被初始化为0。此时任务0仅执行pause()系统调用，并又会调用本函数。
	switch_to(next);     // 切换到Next任务并运行。


```

`sys_pause()`函数中，是当前任务状态变为可中断的等待状态，并且会<font color=red>重新调度</font>,系统调用将导致进程进入睡眠状态，直到收到一个信号。该信号用于终止进程或者使进程调用一个信号捕获函数。只有当捕获了一个信号，并且信号捕获处理函数返回，`pause()`才会返回。

`sleep_on`函数中，是当前状态变为不可中断的等待状态。该函数的主要功能是生成一个 tmp 指针，传递的参数是 **p指针，所以 *p是等待队列头指针，一系列操作之后会使得此时的tmp指针指向原来的等待任务头，而 *p会指向新的等待任务。所以，`sleep_on()`是当一个进程所请求的资源正忙时或者不在内存中时，让这个进程切换出去等待一段时间。切换回来后继续运行。<font color=red>刚开始比如是这样，current在当前任务，\*p在等待任务，没有切换出去之前，加入了 tmp 使得tmp替代了 *p,\*p替代了current，然后进行 shedule，选出next并切换，current又指向了新的当前任务。tmp一直是进程私有的，所以这样就形成了一条链</font>。个人理解就是把 tmp ,\*p,current整体向右移，当前进程重新执行的时候会把它之前的一个等待进程置为就绪，这样，相当于递归回去，每一个唤醒的时候都是去置它之前进入等待的进程为就绪。<font color=red>资源互斥问题。</font>

`interruptible_sleep_on()`为可中断睡眠，基本类似于不可中断睡眠

```c
if (*p && *p != current) {
		if((**p).state!=0) //添加
            fprintk(3,"%ld\t%c\t%ld\n",(**p).pid,'J',jiffies);

		(**p).state=0;
		goto repeat;
	}
/*这几句是schedule之后的语句，表面只要队列头指针和队列头指针的任务不是当前任务的时候，置
 *任务为就绪，然后重新调度。队列头指针的任务不是当前任务的时候说明当前任务被放入等待队列后
 *又有新的任务被插入等待队列前部，因此先唤醒他们
 */
```

`wake_up()`，唤醒的是最后进入等待队列的任务。

2. 在`kernel/exit.c`中主要是退出函数的code，`do_exit()`,退出子进程，并且告诉父进程，这里添加状态切换点，为退出标志。 `sys_waitpid`系统调用挂起当前进程，直到pid指定的子进程退出(终止)或收到要求终止该进程的信号， 或者是需要调用一个信号句柄(信号处理程序)。如果pid所指向的子进程早已退出(已成所谓的僵死进程)， 则本调用将立刻返回。子进程使用的所有资源将释放。

<font color=red>注：</font> 可中断睡眠状态和不可中断睡眠的状态。可中断的睡眠状态的进程会睡眠直到某个条件变为真，如产生一个硬件中断、释放进程正在等待的系统资源或是传递一个信号都可以是唤醒进程的条件。不可中断睡眠状态与可中断睡眠状态类似，但是它有一个例外，那就是把信号传递到这种睡眠状态的进程不能改变它的状态，也就是说它不响应信号的唤醒。

