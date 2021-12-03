

# 				操作系统试验（哈工大）



### ⛷环境搭建

#### 实验材料

1. [hit-oslab-linux-20110823.tar.gz](https://github.com/hoverwinter/HIT-OSLab/tree/master/Resources)	（含Linux-0.11源码以及bochs环境）
2. [gcc-3.4-ubuntu.tar.gz](https://github.com/hoverwinter/HIT-OSLab/tree/master/Resources)	（编译所需的低版本的gcc）

#### 安装

1. GitHub上下载包：

   ```bash
   git clone https://github.com/Wangzhike/HIT-Linux-0.11.git
   ```

2. 到配置环境文件夹：

   ```bash
   cd HIT-Linux-0.11/prepEnv/hit-oslab-qiuyu/
   ```

3. 运行安装脚本：

   ```bash
   ./setup.sh
   ```

   说明：运行完改脚本之后会在 */home/* 目录下创建一个 */oslab* 文件夹，该文件夹就是我们的环境，源代码以及解压好了，可以直接进入 */linux-0.11* 。

4. 编译内核，运行：

   ```bash
   cd ~/oslab/linux-0.11
   
   make all
   
   ./run
   ```

   bochs的界面就不截图显示了。



---

### ⛷一、操作系统的引导

#### 1.改写 <font color=purple>*bootsect.s*</font>

引导程序有BOIS加载并运行，这个文件里面主要使用BIOS中断

```
SETUPLEN=2
SETUPSEG=0x07e0
entry _start
_start:
    mov ah,#0x03			; 读光标位置，使用BIOS10号中断0x03号子功能，光标的返回值在dx中
    xor bh,bh
    int 0x10
    
    mov cx,#40				; cx里存放显示字符串的长度
    mov bx,#0x000c	  ; 字符属性，正常是0x0007，这里我的是0x000c，为红色
    mov bp,#msg1
    mov ax,#0x07c0    
    mov es,ax					
    mov ax,#0x1301
    int 0x10
    
;inf_loop:
 ;   jmp inf_loop     ;死循环
 
 
load_setup:						;加载setup.s
    mov dx,#0x0000    ;设置驱动器和磁头
    mov cx,#0x0002		;设置扇区和磁头号，磁头号为ch，扇区为cl
    mov bx,#0x0200		;设置读入的内存地址 512即bootsect.s执行完之的最后地址
    mov ax,#0x0200+SETUPLEN	;设置读入的扇区个数，Linux为4个，我们的setup.s比较小为2
    									;al需要读出的扇区数量，ah读出到的内存地址，即0x0200
    									;dh磁头号，dl驱动器号    es：bx指向数据缓冲区
    int 0x13					;磁盘读中断
    jnc ok_load_setup	;读入成功
    
    mov dx,#0x0000		;复位重新读入
    mov ax,#0x0000
    int 0x13
    jmp load_setup
ok_load_setup:
    jmpi    0,SETUPSEG
    
msg1:
    .byte   13,10     ; 回车换行的ASCII码
    .ascii  "Begin to learn OS! --MediMa"
    .byte   13,10,13,10
.org 510      ;源程序是508，这里不需要root_dev，所以设510，因为boot_flag要放最后2字节
boot_flag:
    .word   0xAA55
```

#### 2.编写<font color=purple>*setup.s*</font>

最终代码

```nasm
INITSEG  = 0x9000					
entry _start
_start:
! Print "NOW we are in SETUP"   和之前的bootsect.s类似
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#25
    mov bx,#0x000b    ;青色
    mov bp,#msg2
    mov ax,cs
    mov es,ax
    mov ax,#0x1301
    int 0x10

    mov ax,cs
    mov es,ax
! init ss:sp
    mov ax,#INITSEG
    mov ss,ax
    mov sp,#0xFF00

! Get Params			;获取基本参数,放到0x90000
    mov ax,#INITSEG
    mov ds,ax
    
    mov ah,#0x03		;BIOS 10号中断，3号子功能，读光标位置，bh:0,表示输入页号。返回：
    xor bh,bh				;ch扫码开始线，cl为扫码结束线，dh为行号，dl为列号
    int 0x10
    mov [0],dx			;放到9000：0000处
    
    mov ah,#0x88		;BIOS 15号中断，0x88号子功能，读扩展内存大小，保存在90002处
    int 0x15			
    mov [2],ax			;返回ax=0x100000（1M）处开始的内存大小（KB），出错CF置位
    
    mov ax,#0x0000		;从 0x41 处拷贝 16 个字节（磁盘参数表）
    mov ds,ax
    lds si,[4*0x41]				; 把低字(2B)置为偏移地址，高字(2B)置为段地址
    mov ax,#INITSEG
    mov es,ax
    mov di,#0x0004
    mov cx,#0x10
    rep
    movsb

! Be Ready to Print
    mov ax,cs
    mov es,ax
    mov ax,#INITSEG
    mov ds,ax

! Cursor Position
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#18
    mov bx,#0x0007
    mov bp,#msg_cursor
    mov ax,#0x1301
    int 0x10
    mov dx,[0]
    call    print_hex
! Memory Size
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#14
    mov bx,#0x0007
    mov bp,#msg_memory
    mov ax,#0x1301
    int 0x10
    mov dx,[2]
    call    print_hex
! Add KB
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#2
    mov bx,#0x0007
    mov bp,#msg_kb
    mov ax,#0x1301
    int 0x10
! Cyles
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#7
    mov bx,#0x0007
    mov bp,#msg_cyles
    mov ax,#0x1301
    int 0x10
    mov dx,[4]
    call    print_hex
! Heads
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#8
    mov bx,#0x0007
    mov bp,#msg_heads
    mov ax,#0x1301
    int 0x10
    mov dx,[6]
    call    print_hex
! Secotrs
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#10
    mov bx,#0x0007
    mov bp,#msg_sectors
    mov ax,#0x1301
    int 0x10
    mov dx,[12]
    call    print_hex

inf_loop:
    jmp inf_loop

print_hex:
    mov    cx,#4
print_digit:
    rol    dx,#4     ;循环移位，比如0x1234，操作一次之后就是 0x2341
    mov    ax,#0xe0f			;al为 0000 1111，半字节掩码，
    and    al,dl					
    add    al,#0x30		;0x30是数字0的ASCII码，如果是al为1，那么现在al就是1的ASCII码
    cmp    al,#0x3a		;有没有超过10？
    jl     outp				;小于跳转
    add    al,#0x07		;为a~f，原来基础上加0x07，因为a的ascii码为0x41
outp:
    int    0x10
    loop   print_digit
    ret
print_nl:
    mov    ax,#0xe0d     ! 10号中断的0x0e号子程序，al为显示字符的ASCII码，回车
    int    0x10
    mov    al,#0xa     ! LF，换行
    int    0x10
    ret

msg2:
    .byte 13,10
    .ascii "NOW we are in SETUP"
    .byte 13,10,13,10
msg_cursor:
    .byte 13,10
    .ascii "Cursor position:"
msg_memory:
    .byte 13,10
    .ascii "Memory Size:"
msg_cyles:
    .byte 13,10
    .ascii "Cyls:"
msg_heads:
    .byte 13,10
    .ascii "Heads:"
msg_sectors:
    .byte 13,10
    .ascii "Sectors:"
msg_kb:
    .ascii "KB"

.org 510
boot_flag:
    .word 0xAA55
```



