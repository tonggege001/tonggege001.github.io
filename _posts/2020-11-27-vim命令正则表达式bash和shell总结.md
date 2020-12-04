---
layout: post
title: 2020-11-27-vim命令正则表达式bash和shell总结
date: 2020-11-27
categories: blog
tags: [linux,操作系统]
description: 文件系统
---


# vim  


# bash  

查看系统支持的shell：
```shell
cat /etc/passwd
```

查看用户登陆时取得的shell：
```shell
cat /etc/passwd
```

文件 ~/.bash_history 里面记录了上一次登陆以前

命令设置别名：  
alias lm='ls -al'

bash可以使用类型通配符 * ? .

type命令：查看命令的类型
type command  
type -f command  

变量设置  
1. 变量设置用=连接，=左右不能有空格
2. 变量内容若有空格则可以用双引号或者单引号括起来。
双引号内的特殊字符具有特殊含义，如$。  
单引号内的特殊字符只表示纯文本。  
3. 可用转移\将特殊字符转换为纯文本  
4. 在一串命令中，还需要通过其他命令提供的信息，可以使用返单引号``或者$(命令)，例如version=$(uname -r)
5. 若需要对变量增加内容时，则可用"$变量名称"或${变量}累加内容，如： PATH="$PATH":/home/bin  
6. 若变量需要在其他紫禁城执行，则需要以export来使变量变成环境变量： export PATH  
7. 通常大写字符为系统默认变量，自行设置变量可以使用小写字符，方比那判断，不强制。  
8. 取消变量方法： unset 变量名称  

不同shell不共享变量，但是共享环境变量，也就是说进程1 export了一个变量后，再打开新的shell还是可以得到这个变量的值。

查看所有环境变量： export 或者 env  

查看所有变量： set

几个重要的变量： 
```shell
环境变量：  
HOME: 主文件夹  
PATH: 命令执行路径  
HITSIZE：历史命令记录条数最大值设置  
LANG：语系数据，中文一般是zh_CN.UTF-8或者zh_CN.gb2312  
RANDOM：随机数变量，$RANDOM 可以获得一个随机数（0-32767之间）  如：echo $RANDOM  

bash自定义变量  
PS1： 命令提示符设置
  \d：显示星期月日，如Mon Feb 2  
  \H：显示完整的主机名
  \u：目前用户的账号名称  
  \#：执行的第几个命令
  \$：提示符，root是#，一般用户是$  

$：本身也是个变量，表示目前这个Shell的线程代号（PID），例如echo $$  
?：本身也是格变量，表示上个执行的命令的返回值，正常情况下是0，如果执行发生错误则返回错误代码。  例如echo $?，返回上一个命令的执行码  

OSTYPE：操作系统名称  
HOSTTYPE：CPU类型
MACHTYPE：全称  


```



