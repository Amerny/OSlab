# 				操作系统试验（哈工大）



## ⛷二、系统调用

#### 1.系统调用的过程

1. 应用程序调用库函数（API）；
2. API 将系统调用号存入 EAX，然后通过中断调用使系统进入内核态；
3. 内核中的中断处理函数根据系统调用号，调用对应的内核函数（系统调用）；
4. 系统调用完成相应功能，将返回值存入 EAX，返回到中断处理函数；
5. 中断处理函数返回到 API 中；
6. API 将 EAX 返回给应用程序。

通常情况下，**调用系统调用和调用一个普通的自定义函数在代码上并没有什么区别**，但调用后发生的事情有很大不同。调用自定义函数是通过 call 指令直接跳转到该函数的地址，继续运行。而调用系统调用，是调用系统库中为该系统调用编写的一个接口函数，叫 API（Application Programming Interface）。API 并不能完成系统调用的真正功能，它要做的是去调用真正的系统调用，过程是：

- 把系统调用的编号存入 EAX；
- 把函数参数存入其它通用寄存器；
- 触发 0x80 号中断（int 0x80）。

#### 2.目录说明

1.  Linux  的库文件在 :file_folder:`lib` 文件夹中，编译时供内核使用的一些内核常用函数的集合，主要包括退出函数_exit() ，关闭文件函数close()，复制文件描述符函数dup()， 文件打开函数open() ，写文件函数 write() ，执行程序函数 execve() ，内存分配函数 malloc() ，等待子进程状态函数 wait() ，创建会话系统调用 setsid() ，以及包括在include/string.h 的所有字符操作函数

2.  Linux的系统调用申明:file_folder:`linux-0.11/include/linux/sys.h` sys.h列出了内核中所有系统调用函数的原型，以及系统调用函数指针表。

3.  Linux的系统调用实现:file_folder: `system_call.s`，对于所有系统调用的实现函数，内核把它们按照系统调用功能号顺序排列成一张函数指针表，在sys.h中，然后int 0x80的处理过程中根据用户提供的功能号调用对应的系统调用函数进行处理。

#### 3. 具体步骤

1. 在:file_folder:`include/linux/sys.h`中加入我们要实现的 iam 和 whoami 调用以及加到函数指针表 fn_ptr sys_call_table[]中

   ```c
   extern int sys_whoami();
   extern int sys_iam();
   ....
   ....
   .  
   .
   .
   fn_ptr sys_call_table[] = { sys_setup, sys_exit, sys_fork, sys_read,...sys_whoami,sys_iam}
   ```

   

2. 在 :file_folder:`kernel/system_call.s`中，增加了系统调用还要修改系统调用总数

   ```asm
   !……
   ! # 这是系统调用总数。如果增删了系统调用，必须做相应修改
   nr_system_calls = 72  # 增加两个改成74
   !……
   
   .globl system_call
   .align 2
   system_call:
   
   ```

   

3. 在:file_folder:`/kernel`中添加系统调用函数 who.c并实现，可以参考其他系统调用的实现

   用户态和内核态之间的传递讲点可以参考[实验楼的系统调用实验6.7](https://www.lanqiao.cn/courses/115/learning/?id=569)

   ```c
   #include <asm/segment.h>
   #include <errno.h>
   #include <string.h>
   
   char _myname[24];
   
   int sys_iam(const char *name)
   {
       char str[25];
       int i = 0;
   
       do
       {
           // get char from user input
           str[i] = get_fs_byte(name + i);
       } while (i <= 25 && str[i++] != '\0');
   
       if (i > 24)
       {
           errno = EINVAL;
           i = -1;
       }
       else
       {
           // copy from user mode to kernel mode
           strcpy(_myname, str);
       }
   
       return i;
   }
   
   int sys_whoami(char *name, unsigned int size)
   {
       int length = strlen(_myname);
       printk("%s\n", _myname);
   
       if (size < length)
       {
           errno = EINVAL;
           length = -1;
       }
       else
       {
           int i = 0;
           for (i = 0; i < length; i++)
           {
               // copy from kernel mode to user mode
               put_fs_byte(_myname[i], name + i);
           }
       }
   
       return length;
   }
   ```

   

4. 编译测试，修改:file_folder:`kernel/Makefile`，共两处.

   ```makefile
   ### 第一处
   OBJS  = sched.o system_call.o traps.o asm.o fork.o \
           panic.o printk.o vsprintf.o sys.o exit.o \
           signal.o mktime.o
   ```

   改为

   ```makefile
   OBJS  = sched.o system_call.o traps.o asm.o fork.o \
           panic.o printk.o vsprintf.o sys.o exit.o \
           signal.o mktime.o who.o
   ```

   ---

   ```makefile
   ### Dependencies:
   exit.s exit.o: exit.c ../include/errno.h ../include/signal.h \
     ../include/sys/types.h ../include/sys/wait.h ../include/linux/sched.h \
     ../include/linux/head.h ../include/linux/fs.h ../include/linux/mm.h \
     ../include/linux/kernel.h ../include/linux/tty.h ../include/termios.h \
     ../include/asm/segment.h
   ```

   改为

   ```makefile
   ### Dependencies:
   who.s who.o: who.c ../include/linux/kernel.h ../include/unistd.h
   exit.s exit.o: exit.c ../include/errno.h ../include/signal.h \
     ../include/sys/types.h ../include/sys/wait.h ../include/linux/sched.h \
     ../include/linux/head.h ../include/linux/fs.h ../include/linux/mm.h \
     ../include/linux/kernel.h ../include/linux/tty.h ../include/termios.h \
     ../include/asm/segment.h
   ```

   此时，可以执行 `make all`，将这两个调用加入到内核中，`./run`之后接着如下的步骤测试

5. 在:file_folder:`/usr/root/include/unistd.h`中添加

   ```c
   #define __NR_whoami 72
   #define __NR_iam  73
   ```

6. 编写测试程序

   `~/whoami.c`

   ```c
   /* 有它，_syscall1 等才有效。详见unistd.h */
   #define __LIBRARY__
   
   /* 有它，编译器才能获知自定义的系统调用的编号 */
   #include "unistd.h"
   
   #include <stdio.h>
   #include <errno.h>
   
   /* whoami()在用户空间的接口函数 */
   _syscall2(int, whoami,char*,name,unsigned int,size);
   
   int main(char *arg) {
       char myname[50];
       whoami(myname, 50);
       return 0;
   }
   ```

   ---

   `~/iam.c`

   ```c
   /* 有它，_syscall1 等才有效。详见unistd.h */
   #define __LIBRARY__
   
   /* 有它，编译器才能获知自定义的系统调用的编号 */
   #include "unistd.h"
   
   #include <stdio.h>
   #include <errno.h>
   
   /* iam()在用户空间的接口函数 */
   _syscall1(int, iam, const char*, name);
   
   int main(int argc, char* argv[]) {
       iam(argv[1]);
       return 0;
   }
   ```

7. 编译并进行测试

   ```shell
   # 编译
   gcc -o iam iam.c
   gcc -o whoami whoami.c
   
   #测试
   ./iam hello
    ./whoami
   
   ```

   