---
layout: post
title: 2021-06-05-Linux Bash命令
date: 2021-06-05
categories: blog
tags: [Linux]
description: Linux Bash命令
---

## sort

```shell
sort [-fbMnrtuk] [file or srdin]  
-f: 忽略大小写差异   
-b: 忽略最前面的空格符部分  
-M: 以月份的名字来排序  
-n: 使用纯数字进行排序  
-r: 反向排序  
-u: 相同数据只出现一行代表  
-t: 分隔符，默认是[TAB]键分割  
  
eg: cat /etc/passed | sort  
```  

## wc  
```shell
wc [-lwm]  
-l: 仅列出行  
-w: 仅列出多少字  
-m: 多少字符  

eg: ls -l | wc -l
```

## tee  
```shell
tee [-a] file  

last | tee last.list | cut -d " " -f1  
功能：让标准输出转存一份到文件last.list中
```

## tr
```shell
tr [-ds] SET1 ...  
-d: 删除信息当中的SET1这个字符串  
-s: 替换掉重复的字符串  

将输出中的小写字母转成大写字母  
eg: last | tr '[a-z]' '[A-Z]  


```

## expand  
将[tab]转换成空格


# Shell Script  
1. 一行命令太多，可以\[Enter]扩展命令  
2. \# 是注释  
3. 读取到一个[Enter]就开始执行该行  

## 格式  
1. 第一行： #!/bin/bash  
2. 指明PATH和LANG
3. exit 0是退出值  

## 例子  
1. 获取用户输入
```shell
read -p "Please input your first name: " firstname      # 提示用户输入  
read -p "Please input your last name: " lastname        # 提示用户输入  
echo "\n Your full name is: $firstname $lastname"       # 结果由屏幕输出  
exit 0  
```

2. 创建按照日期的文件
```shell
#!/bin/bash  

PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:~/bin  
export PATH  

echo "I will use 'touch' command to create 3 files."  
read -p "Please input your filename: " fileuser  

filename=${fileuser:-"filename"}        # 鸟哥书P315也，变量名设置，若变量名为空则设置成filename  
  
# 利用date设置文件名  

echo $filename  

date1=$(date --date='2 days ago' +%Y%m%d)       # 前两天日期 $(command) 是用来取得输出信息的  
date2=$(date --date='1 days ago' +%Y%m%d)       # 前一天日期  
date3=$(date +%Y%m%d)  

echo $date1  

file1=${filename}${date1}  
file2=${filename}${date2}  
file3=${filename}${date3}  

touch "$file1"  
touch "$file2"  
touch "$file3"  
exit 0  

```


## script 执行方式  

1. 直接 sh xxx.sh 进行执行： script都会使用一个新的bash环境执行脚本内的内容  

2. 利用source 或者 . 执行： script在原本的bash环境内执行，那么其中声明的变量还会生效，  

## script 判断式  

### 1 test 命令判断文件  
```shell
test -e /dmtsai && echo "Exist" || echo "Not Exist"
```  

test 其他测试功能：  
| flag | 意义 |
|:---- |:---- |
|测试文件名| |
|-e | 该文件名是否存在,常用 |
|-f | 该文件名是否为文件(file),常用 |
|-d | 该文件名是否为目录(directory),常用 |  
|-b | 该文件名是否是个块设备 |
|-S | 该文件名是否是Socket文件 |
|-L | 该文件名是否是一个连接文件  
|文件权限检测| |
|-r | 检测文件名是否存在且具有可读权限 |
|-w | 检测文件名是否存在且具有可写权限 |
|-x | 检测文件名是否存在且具有可执行权限|
|-u | 检测文件名是否存在且具有SUID属性 |
|-g | 检测文件名是否存在且具有SGID属性 |
|-k | 检测文件名是否存在且具有SBIT属性 |
|-s | 检测文件名是否存在且为非空白文件 |
|两个文件之间的比较| test file1 -nt file2 |
|-nt | 判断file1是否比file2新 |
|-ot | 判断file1是否比file2旧 |
|-ef | 判断file1和file2是否为同一个文件，主要判断hard link的判定。判断是否均指向同一个inode |  
|两个整数的判定 | test n1 -eq n2 |
|-eq | 两数值相等(equal) |  
|-ne | 两数值不等(not equal) |
|-gt | n1 > n2 | 
|-lt | n1 < n2 |
|-ge | n1 >= n2 | 
|-le | n1 <= n2 |
|字符串判定| test -z string|
| test -z string | 判断字符串是否为0，若为空字符串则返回true |  
| test -n string | 判断字符串是否非0，若字符串不为空则返回true |
| test str1 = str2 | 判断两个字符串是否相等 |  
| test str1 != str2 | 判断两个字符串是否不相等 |  
| 多重条件判定 |  |  
| -a | 两个条件同时成立！例如 test -r file -a -x file, 文件必须同时具有可读可执行权限才为true |
| -o | 任何一个条件成立都返回true |  
| ! | 反向状态， 例如 test ! -x file , 当file不具有x权限时返回
true |  

### [] 判断  
```shell
例如： [ "$HOME" == "$MALL" ]  # 这里[]两边必加空格，变量用双引号括起来
```

一个让用户选择Y/N的例子： 
```shell
read -p "Please input (Y/N) : " yn

[ "$yn" == "Y" -o "$yn" == "y" ] && echo "OK, continue" && exit 0  
[ "$yn" == "N" -o "$yn" == "n" ] && echo "Oh, interrupt!" && exit 0
echo "I don't know what your choice is" && exit 0

```

## Linux 各类符号  
1. 纯{}：扩展功能(例如：ls {name1,name2}.sh ；touch {1..3}.sh)、代码组织功能({ cmd1;cmd2;cmd3 })  

2. ${} ：取变量的值  

3. 单中括号[]：用于条件测试，f  [ $num1 -eq $num2 ] ; then 等价于 if  /usr/bin/[ $num1 -eq $num2 ] ; then   
还等价于if  test $num1 -eq $num2 ; then  
单中括号用于数组：例如：输出数组全部元素echo ${array[@]}  

4. 双中括号：单中括号加强版 [[是 bash 程序语言的关键字，并不是一个命令，在单括号中是不可以用 ||  &&等关系运算符号的，只能用-a -o等等，例如[[ $arch_com = i386 || $ARCH = i*86 ]]  

5. 小括号()：此处命令块会在**子shell**中运行，例如：(cmd1;cmd2;cmd3) 这三条命令的执行不会影响当前的shell环境。单括号数组初始化功能：例如：array=(1 2 3 4 5)双括号算术运算比较功能：例如：for((i=0;i<9;i++)) 如果不采用双括号可用for i in `seq 0 9`或者for i in {0..9}代替。  ((表达式))用来扩展Shell中的算术运算,以及赋值运算,扩展for,while,if条件测试运算.

## Linux bash 默认变量  

|符号|描述|
|:----:|:----:|
|$0|脚本名称|
|$1|第一个参数|
|$2|第二个参数|
|$3|第三个参数|
|$4|第四个参数|
|$#|后面接的参数个数|
|$@|代表$1，$2，$3，$4|
|$*|代表"$1c$2c$3c$4"，其中c为分隔符，本例为:"$1 $2 $3 $4"

偏移命令：shift num，拿掉前面几个参数使$1~$4的内容改变  

## Bash 条件命令  
```shell
if [条件判断式]; then
    XXX 
fi
```

多个条件用多个中括号连接，然后中括号用&&和||连接  ，例如：
```shell
if [条件判断式] || [条件判断式]; then  
    echo OK  
fi
```

其他结构：
```shell
if [条件判断式]; then

else

fi

if [条件判断式]; then  
    XXX
elif [条件判断式2]; then
    XXX
else  
    XXXX
fi
```


case的用例：
```shell
case $1 in  
    "hello")  
    echo "Hello, how are you?"  
    ;;  
    "")  
    echo "You MUST input parameter, ex> {$0 someword}"  
    ;;  
    *)          # 相当于通配符
    echo "Usage $0 {hello}"
    ;;
esac
```

## Bash function 功能 
function的定义一定要在最前面，这样才能够在被执行时找到可用的程序段。 函数的参数也用\$0~\$3表示。
```shell
function fname() {
    程序段
}
```
一个例子：
```shell
function printit () {
    echo "Your choice is $1"  
}  

printit 2           # 参数时2  
```

## Bash 循环  
### 不定循环 while do done, until do down  

```shell
while [condition]
do
    程序段落  
done

```

```shell
until [condition]       # 当condition成立时就终止循环
do  
    程序段落  
done
```
一个例子：  
```shell
s=0  
i=0  
while [ "$i" != "100" ]
do  
    i=$(($i+1))
    s=$(($s+$i))
done
echo "The result of '1+2+...+100' is ==> $s"
```  

### 固定循环  
第一次循环时， $var的内容是con1，第二次是con2，第三次是con3...
```shell
for var in con1 con2 con3 ...
do
    程序段  
done
```  

(seq 1 100)：产生1~100的整数序列  

第二种方法：
```shell
for ((初始值; 限制值; 执行步长))  
do  
    程序段  
done
```

例子：  
```shell
for (( i=1; i<=$nu; i=i+1  ))
do
    s=$(($s + $i))
```

## shell script 的追踪与调试  
```shell
sh [-nvx] script.sh  
-n: 不执行script.sh，仅查询语法问题  
-x：将使用到的内容显示到屏幕上

```