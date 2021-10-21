---
layout: post
title: 2021-07-09-Linux Crontab SElinux daemons
date: 2021-07-09
categories: blog
tags: [Linux]
description: 2021-7-9-Linux SElinux daemons
---

## 1. Crontab  

### at命令  
在某一个时间点只执行一次的命令。  

### crontab命令  
循环在某个时间点执行，分为用户Crontab和系统crontab，用户crontab -u username，系统crontab需要编辑/etc/crontab，同时crontab.weekly目录下存放每周需要执行的脚本。  

### anacron  
处理非24小时启动的Linux，用来补充执行关机这段时间需要执行的命令，一般用默认参数即可。  

## 2. 进程操作  
&: 让这个进程放置在后台执行，在这一个终端还可以继续接受其他命令。若终端关闭则后台进程也会关闭，所以一般搭配nohup。例子：  
```shell
cp file1 file2 &
```  

ctrl-z：暂停进程  

列出目前的后台工作状态：  
```shell  
jobs [-lrs]  
-l: 列出后台job number的进程（run或者暂停的），同时列出PID  
-r: 仅列出正在后台run的工作  
-s: 仅列出正在后台中暂停的工作  
```  

将后台工作拿到前台来处理：  
```shell
fg %jobumber  

> jobs  
[1]- 10314 Stopped vim ~/.bashrc  
[2]+ 10833 Stopped find / -print  

> fg        # 默认取出+的那个工作，表示刚添加到后台进程  
> fg %1  
```  

让工作在后台下的状态变成运行中: bg  
因为[ctrl]+z让进程在后台变成暂停状态，此时可以用bg命令让进程在后台变成Run状态。  
```shell
bg %jobnumber  
# 或者直接bg，默认选择+那个进程  
```

管理后台当中的工作：kill  
列出后台能用的信号：  
```shell  
kill -l  
```  
发送信号：  
```shell  
kill -signal %jobnumber（直接接数字表示pid）  
# signal: 信号序号  
# -1：重新读取一次参数的配置文件，类似于reload  
# -2：代表与由键盘输入ctrl-c同样的操作  
# -9：立刻强制删除一个工作  
# -15：以正常的程序方式终止一项工作  
# -SIGTERM：与-15是一样的
```  
-9和-15区别：如果用vi打开一个文件，如果是-15退出则不会由.swap文件，如果是-9异常退出，则由.swap临时文件。  

### 17.2.3 脱机管理  
当关闭SSH连接后，前台和后台的进程都会被关闭，而nohup让进程脱机管理，配合&可以实现后台脱机管理。
```shell  
nohup xxxx &
```  
### 17.3.1 进程查看  
```shell  
# 查看用户的进程信息  
ps -l  
# 查看所有的进程信息  
ps -aux  
```
各个字段显示的百分比：  
|标识符|作用|
|:----:|:----:|  
|USER|该进程属于哪个用户账号|
|PID|进程的PID|
|%CPU|该进程使用掉的CPU资源百分比|  
|%MEM|该进程所占用的**物理内存**百分比|
|VSZ|该进程使用掉的虚拟内存量(KB)|
|RSS|该进程占用的固定的内存量|
|TTY|该进程是在哪个终端机上运行，若与终端机无关则显示?|  
|另外|tty1~tty6是本机上面的登录者程序, pts/0是网络连接进主机的进程|
|START|该进程被触发启动的时间|
|STAT|该进程目前的状态(R:runing, S:sleep, D: 不可被唤醒的睡眠状态)|
|T:停止状态|可能是工作控制（后台暂停或者除错）, Z:zombie：僵尸状态|

动态查看时间段内各个参数的变化：  
```shell  
# 按秒数更新
top [-d 秒数]  

# 在更新界面按下M就可以按照内存占比排序，按下P按照CPU占比排序，按下Q离开  

# 只查看一个进程  
top -d 2 -p PID  
```  

pstree: 查看进程的父子关系  
```shell 
# 列出目前系统上面所有的进程树的相关性   
pstree -A  

# 同时显示出PID和users  
pstree -Aup
```  

发送信号：  
kill -signal PID（或者%jobnumber也可以）  
常用信号：  
|代号|名称|内容|
|:----:|:----:|:----:|
|1|SIGHUP|启动被终止的进程，可让该PID重新读取自己的配置文件，类似重新启动|
|2|SIGINT|相当于用键盘输入[ctrl]+c来中断一个进程的进行|
|9|SIGKILL|强制中断一个进程，尚未完成的部分可能会有半产品产生，比如.swap文件|
|15|SIGTERM|以正常的结束进程来终止，如果进程已经发生问题，则输入signal没有用的|
|17|SIGSTOP|相当于[ctrl]+z来暂停一个进程的进行|

killall -signal command  









