title: 'nohup &与daemon进程'
date: 2015-12-09 11:16:27
tags: Linux
categories: Linux/Unix
description: Introduce nohup & daemon
feature: nohup.jpg
toc: true
---

我们先来了解下后台进程的定义，顾名思义，在后台运行的进程，那么什么叫前台进程呢？
举个例子：在Terminal运行./command -arg1=xxx，在进程没有结束之前，终端都挂起，不能做其它的事情，
这样的进程我理解为前台进程，对应的在后面加一个‘&’（./command -arg1=xxx &）
也就是后台进程了，暂时先介绍到这里。

<!-- more -->

## 前台进程 ##

在terminal运行前台进程，也就是父进程terminal进程fork了一个子进程，terminal进程等待，待子进程
执行结束，向父进程发出SIGCHLD信号，唤醒父进程，由于子进程共享父进程的文件描述符stdin、stdout、stderr，
所以在子进程的执行过程中，终端可以接受子进程的输出，子进程也可以接受terminal的输入，但是一旦
terminal进程退出，比如关闭ssh窗口呢，子进程就会被迫终止，原因是terminal进程退出时会发送SIGHUP信号
给所有子进程，子进程终止，解决这个问题，我们想到了后台进程。

## 后台进程 ##

前台进程会导致terminal挂起，必须等待子进程结束，那么我们可以在命令后面加上‘&’，
./command -arg1=xxx &
这样子进程就变成后台进程了，终端没有wait，而是继续执行其它命令，但是这样并不能解决ssh窗口关闭时
子进程终止的问题，你可以尝试一下，为了解决这个问题，引入了nohup，顾名思义（no hup），没有SIGHUP信号，
也就是加了nohup之后，fork出来的子进程忽略了SIGHUP信号（系统调用signal就可以做到）：
nohup ./command -arg1=xxx &
子进程的输出默认会在当前目录的nohup.out，如果不可写，会写在$HOME/nohup.out。
此时，ssh窗口的关闭，或者当前用户的退出都不会影响子进程的运行，该进程会由init进程接管。

## daemon进程 ##

也就是守护进程，又称精灵进程。这也是一种后台进程，一般系统服务都是守护进程，比如0号进程（系统调度进程）、
1号进程（init进程）等都是守护进程，这是一种完全脱离中终端控制的进程，那么问题来了，为什么不用
nohup+&呢？咱们先从daemon进程的实现说起：
其实，linux已经提供了一个api可以将进程变成守护进程，可以参考：http://man7.org/linux/man-pages/man3/daemon.3.html

int daemon ( int __nochdir, int __noclose);
如果__nochdir的值为0，则将切换工作目录为根目录；
如果__noclose为0，则将标准输入，输出和标准错误都重定向到/dev/null。
使用非常简单，举个例子：
``` c ```
#include <unistd.h>
#include <stdio.h>
int main()
{
    daemon(0,0);
    while (1)
    {
    	//Todo...
    
    	sleep(1);
    }
}
```
一般是自己实现守护进程的daemon函数（有时需要屏蔽一些特殊的信号），例如：

``` c ```
void daemon()
{ 
    int maxfile = 65535;

    pid_t pid;

    if ((pid = fork()) != 0)
    {   
	//父进程退出
        exit(0);
    }   
    //父进程退出后，此时子进程不再占用终端，变成后台进程


    //在子进程成为回话组长之前，屏蔽一些可以让子进程退出的信号：
    signal(SIGHUP, SIG_IGN);  //进程组长退出时向所有成员发出的
    signal(SIGTTOU, SIG_IGN); //表示后台进程写控制终端
    signal(SIGTTIN, SIG_IGN); //表示后台进程读控制终端
    signal(SIGTSTP, SIG_IGN); //表示终端挂起

    signal(SIGINT, SIG_IGN);
    signal(SIGQUIT, SIG_IGN);
    signal(SIGPIPE, SIG_IGN); //socket客户端退出，服务端还在写时会异常退出
    signal(SIGCHLD, SIG_IGN);
    signal(SIGTERM, SIG_IGN);

    //将此进程设置成会话组长和进程组组长，从此脱离了原来的进程组和会话
    //也就是彻底脱离了控制终端,安全了，很难被打断了。
    setsid();

    //为什么要再次fork呢？很多人认为这里没必要，其实还是有点用的，
    //当前进程已经是进程组长了，有权限再打开控制终端，为了防止这一点
    //还是使用孙子进程比较靠谱，孙子进程既脱离了终端，又不是组长，多安全！
    if((pid = fork()) != 0)
    {   
        //父进程结束,变成daemon
        exit(0);
    }   

    //重设文件创建掩模 
    //进程从创建它的父进程那里继承了文件创建掩模。
    //它可能修改守护进程所创建的文件的存取位。为防止这一点，将文件创建掩模清除
    umask(0);

    //这个很重要，如果不改变当前目录，那么当前目录所在的分区就无法卸载。
    chdir("/");
    
    //关闭文件描述符
    for(i=0;i<maxfile;i++)
    close(i);
}
```
咱们继续回到上面的问题，为什么不用nohup &呢？
> 1.显而易见，使用nohup启动的后台进程并没有脱离父进程的终端控制，会带来什么问题呢？
终端可以发信号使其终止，参见上面讲的信号。 而且，还会输出到终端，就算重定向了，只是多存了一份而已。
> 2.nohup启动的后台进程依然共享父进程的文件描述符，如果不关闭，则会浪费系统资源，造成
文件描述符对应文件所在的文件系统无法卸下以及引起其他无法预料的错误。

综上所述，个人认为与具体业务有关的进程使用nohup足矣，
需要开机自启动、与系统或者工程基础架构有关、需要长期运行对稳定性要求高的程序，
可以做成守护进程，比如异步日志组件的监控等之类的监控进程。

有关linux会话、进程组等知识，可以参考下《unix环境高级编程》。















