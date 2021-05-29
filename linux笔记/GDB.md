# gdb调试原理

## 一、gdb简介

　　gdb：GNU debugger
　　UNIX及UNIX-like下一个强大的命令行的调试工具
　　gdb调试的整体架构如下图所示：
　　![这里写图片描述](https://img-blog.csdn.net/20160415101242443)
　　可以发现gdb调试不管是本地调试还是远程调试，都是基于ptrace系统调用来实现的 　　

## 二、ptrace

　　ptrace系统调用的原型：

`lon`g ptrace(enum __ptrace_request request, \

 pid_t pid,void *addr,void *data);` 

　　ptrace系统调用提供了一种方法，让父进程可以观察和控制其它进程的执行，检查和改变其核心映像及寄存器。主要用来实现断点调试和系统调用跟踪
　　可通过man手册查看具体使用：man ptrace
　　
　　request参数的主要选项：
PTRACE_TRACEME：由子进程调用，表示本进程将被其父进程跟踪，交付给这个进程的所有信号，即使信号是忽略处理的（除SIGKILL之外），都将使其停止，父进程将通过wait()获知这一情况。

PTRACE_ATTACH： attach到一个指定的进程，使其成为当前进程跟踪的子进程，而子进程的行为等同于它进行了一次PTRACE_TRACEME操作。但是，需要注意的是，虽然当前进程成为被跟踪进程的父进程，但是子进程使用getppid()的到的仍将是其原始父进程的pid。
这下子gdb的attach功能也就明朗了。当你在gdb中使用attach命令来跟踪一个指定进程/线程的时候，gdb就自动成为改进程的父进程，而被跟踪的进程则使用了一次PTRACE_TRACEME，gdb也就顺理成章的接管了这个进程。

PTRACE_CONT：继续运行之前停止的子进程。可同时向子进程交付指定的信号。

## 三、gdb三种调试方式

1）attach并调试一个已经运行的进程：
确定需要进行调试的进程id
运行gdb，输入attch pid，如：gdb 12345。gdb将对指定进行执行如下操作：ptrace（PTRACE_ATTACH，pid，0,0）
2）运行并调试一个新的进程
运行gdb，通过命令行参数或file指定目标调试程序，如gdb ./test
输入run命令，gdb执行下述操作：
通过fork()系统调用创建一个新进程
在新创建的子进程中调用ptrace(PTRACE_TRACEME，0,0,0）
在子进程中通过execv（）系统调用加载用户指定的可执行文件
3）远程调试目标主机上新创建的进程
gdb运行在调试机，gdbserver运行在目标机，通过二者之间定义的数据格式进行通信

## 四、gdb调试的基础—信号

　　gdb调试的实现都是建立在信号的基础上的，在使用参数为PTRACE_TRACEME或PTRACE_ATTACH的ptrace系统调用建立调试关系后，交付给目标程序的任何信号首先都会被gdb截获。
　　因此gdb可以先行对信号进行相应处理，并根据信号的属性决定是否要将信号交付给目标程序。
　　
　　1、设置断点： 　　
　　信号是实现断点的基础，当用breakpoint 设置一个断点后，gdb会在=找到该位置对应的具体地址，然后向该地址写入断点指令INT3，即0xCC。
　　目标程序运行到这条指令时，就会触发SIGTRAP信号，gdb会首先捕获到这个信号。然后根据目标程序当前停止的位置在gdb维护的断点链表中查询，若存在，则可判定为命中断点。
　　gdb暂停目标程序运行的方法是想起发送SIGSTOP信号。
　　
　　2、next单步调试：
　　next指令可以实现单步调试，即每次只执行一行语句。一行语句可能对应多条及其指令，当执行next指令时，gdb会计算下一条语句对应的第一条指令的地址，然后控制目标程序走到该位置停止。
　　
![img](https://img-blog.csdn.net/20160415105521820)

# gdb调试方法

## 1.启动gdb

编译一个测试程序，-g表示可以调试，命令如下：

`gcc -g  test.c -o test`

启动gdb，命令如下：

`gdb test`

`gdb -q test//-q表示不打印版本信息`

## 2.查看源码

list(简写 l)： 查看源程序代码，默认显示10行，按回车键继续看余下的。
测试如下：

```
(gdb) list 
	#define MAX_SIZE	
int main()
{
    int i,fd,size1 ,size2 ,len;
    char *buf = "helo!I'm liujiangyong ";
    char buf_r[15];
    len = strlen(buf);
    fd = open("/home/hello.txt",O_CREAT | O_TRUNC | O_RDWR,0666);
    if (fd<0)
(gdb) 
     {
            perror("open :");
            exit(1);
      }
    else
      {
	    printf("open file:hello.txt %d\n",fd);       }
    size1 = write(fd,buf,len);
    if (fd<0)
(gdb) 
	{
		printf("writre erro;");	
    }
    else
    {
        printf("写入的长度：%d\n写入文本内容：%s\n",size1,buf);

    }
   lseek(fd,0,SEEK_SET);
(gdb) 
   size2 = read(fd,buf_r,12);
   if (size2 <0)
   {
   printf("read  erro\n");
   }
   else
   {
        printf("读取长度：%d\n 文本内容是：%s\n",size2,buf_r);
    }
    close(fd);    
(gdb) 	
	}
(gdb) 
Line number 52 out of range; write.c has 51 lines.
(gdb) 
```



## 3、运行程序

run(简写 r) ：运行程序直到遇到 结束或者遇到断点等待下一个命令；
测试如下：

```
(gdb) r
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /home/eit/c_test/test 
open file:hello.txt 3
写入的长度：22
写入文本内容：helo!I'm liujiangyong 
读取长度：12
 文本内容是：helo!I'm liu
[Inferior 1 (process 19987) exited normally]
(gdb) 
```



## 4、设置断点

break(简写 b) ：格式 b 行号，在某行设置断点；
info breakpoints ：显示断点信息
Num： 断点编号
Disp：断点执行一次之后是否有效 kep：有效 dis：无效
Enb： 当前断点是否有效 y：有效 n：无效
Address：内存地址
What：位置

```
(gdb) b 5
Breakpoint 3 at 0x400836: file write.c, line 5.
(gdb) b 26 
Breakpoint 4 at 0x4008a6: file write.c, line 26.
(gdb) b 30
Breakpoint 5 at 0x4008c6: file write.c, line 30.
(gdb) info breakpoints 
Num     Type           Disp Enb Address            What
3       breakpoint     keep y   0x0000000000400836 in main at write.c:5
4       breakpoint     keep y   0x00000000004008a6 in main at write.c:26
5       breakpoint     keep y   0x00000000004008c6 in main at write.c:30
(gdb) 
```



## 5、单步执行

使用 continue、step、next命令
测试如下：

```
(gdb) r
Starting program: /home/eit/c_test/test 

Breakpoint 3, main () at write.c:12
	{
(gdb) n
	    char *buf = "helo!I'm liujiangyong ";
(gdb) 
	    len = strlen(buf);
(gdb) 
	    fd = open("/home/hello.txt",O_CREAT | O_TRUNC | O_RDWR,0666);
(gdb) s
open64 () at ../sysdeps/unix/syscall-template.S:81
	../sysdeps/unix/syscall-template.S: No such file or directory.
(gdb) 
main () at write.c:18
	    if (fd<0)
(gdb) 
	        printf("open file:hello.txt %d\n",fd);
(gdb) 
__printf (format=0x400a26 "open file:hello.txt %d\n") at printf.c:28
	printf.c: No such file or directory.
(gdb) c
Continuing.
open file:hello.txt 3

Breakpoint 4, main () at write.c:27
	    size1 = write(fd,buf,len);
(gdb) 
Continuing.
写入的长度：22
写入文本内容：helo!I'm liujiangyong 
读取长度：12
 文本内容是：helo!I'm liu
[Inferior 1 (process 20737) exited normally]
(gdb) 
```



## 6、查看变量

使用print、whatis命令
测试如下：

```
main () at write.c:28
	    if (fd<0)
(gdb) 
	        printf("写入的长度：%d\n写入文本内容：%s\n",size1,buf);
(gdb) print fd
$10 = 3
(gdb) whatis fd
type = int
(gdb) 
```



## 7、退出gdb

用quit命令退出gdb：

```
(gdb) r
Starting program: /home/eit/c_test/test 
open file:hello.txt 3
写入的长度：22
写入文本内容：helo!I'm liujiangyong 
读取长度：12
 文本内容是：helo!I'm liu
[Inferior 1 (process 20815) exited normally]
(gdb) q
root@ubuntu:/home/eit/c_test# 
```


continue（简写 c)： 继续执行程序，直到下一个断点或者结束；
next（简写 n ）：单步执行程序，但是遇到函数时会直接跳过函数，不进入函数；
step(简写 s) ：单步执行程序，但是遇到函数会进入函数；
until：当你厌倦了在一个循环体内单步跟踪时，这个命令可以运行程序直到退出循环体；
until+行号： 运行至某行，不仅仅用来跳出循环；
finish： 运行程序，直到当前函数完成返回，并打印函数返回时的堆栈地址和返回值及参数值等信息；
call 函数(参数)：调用程序中可见的函数，并传递“参数”，如：call gdb_test(55)；
quit：简记为 q ，退出gdb；

# gdb基本使用命令

## 1、运行命令

run：简记为 r ，其作用是运行程序，当遇到断点后，程序会在断点处停止运行，等待用户输入下一步的命令。
continue （简写c ）：继续执行，到下一个断点处（或运行结束）
next：（简写 n），单步跟踪程序，当遇到函数调用时，也不进入此函数体；此命令同 step 的主要区别是，step 遇到用户自定义的函数，将步进到函数中去运行，而 next 则直接调用函数，不会进入到函数体内。
step （简写s）：单步调试如果有函数调用，则进入函数；与命令n不同，n是不进入调用的函数的
until：当你厌倦了在一个循环体内单步跟踪时，这个命令可以运行程序直到退出循环体。
until+行号： 运行至某行，不仅仅用来跳出循环
finish： 运行程序，直到当前函数完成返回，并打印函数返回时的堆栈地址和返回值及参数值等信息。
call 函数(参数)：调用程序中可见的函数，并传递“参数”，如：call gdb_test(55)
quit：简记为 q ，退出gdb

## 2、设置断点

break n （简写b n）:在第n行处设置断点
（可以带上代码路径和代码名称： b OAGUPDATE.cpp:578）
b fn1 if a＞b：条件断点设置
break func（break缩写为b）：在函数func()的入口处设置断点，如：break cb_button
delete 断点号n：删除第n个断点
disable 断点号n：暂停第n个断点
enable 断点号n：开启第n个断点
clear 行号n：清除第n行的断点
info b （info breakpoints） ：显示当前程序的断点设置情况
delete breakpoints：清除所有断点：

## 3、查看源码

list ：简记为 l ，其作用就是列出程序的源代码，默认每次显示10行。
list 行号：将显示当前文件以“行号”为中心的前后10行代码，如：list 12
list 函数名：将显示“函数名”所在函数的源代码，如：list main
list ：不带参数，将接着上一次 list 命令的，输出下边的内容。

## 4、打印表达式

print 表达式：简记为 p ，其中“表达式”可以是任何当前正在被测试程序的有效表达式，比如当前正在调试C语言的程序，那么“表达式”可以是任何C语言的有效表达式，包括数字，变量甚至是函数调用。
print a：将显示整数 a 的值
print ++a：将把 a 中的值加1,并显示出来
print name：将显示字符串 name 的值
print gdb_test(22)：将以整数22作为参数调用 gdb_test() 函数
print gdb_test(a)：将以变量 a 作为参数调用 gdb_test() 函数
display 表达式：在单步运行时将非常有用，使用display命令设置一个表达式后，它将在每次单步进行指令后，紧接着输出被设置的表达式及值。如： display a
watch 表达式：设置一个监视点，一旦被监视的“表达式”的值改变，gdb将强行终止正在被调试的程序。如： watch a
whatis ：查询变量或函数
info function： 查询函数
扩展info locals： 显示当前堆栈页的所有变量

## 5、查看运行信息

where/bt ：当前运行的堆栈列表；
bt backtrace 显示当前调用堆栈
up/down 改变堆栈显示的深度
set args 参数:指定运行时的参数
show args：查看设置好的参数
info program： 来查看程序的是否在运行，进程号，被暂停的原因。

## 6、分割窗口

layout：用于分割窗口，可以一边查看代码，一边测试：
layout src：显示源代码窗口
layout asm：显示反汇编窗口
layout regs：显示源代码/反汇编和CPU寄存器窗口
layout split：显示源代码和反汇编窗口
Ctrl + L：刷新窗口

## 7、cgdb强大工具

cgdb主要功能是在调试时进行代码的同步显示，这无疑增加了调试的方便性，提高了调试效率。界面类似vi，符合unix/linux下开发人员习惯;如果熟悉gdb和vi，几乎可以立即使用cgdb。