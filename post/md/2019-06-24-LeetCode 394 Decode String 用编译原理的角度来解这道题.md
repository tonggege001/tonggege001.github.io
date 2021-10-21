---
layout: post
title: LeetCode 394 Decode String 用编译原理的角度来解这道题
date: 2019-06-24
categories: blog
tags: [技术,编译原理,leetcode,代码实现]
description: LeetCode 394 Decode String 用编译原理的角度来解这道题
---

## 题目如下：

>Given an encoded string, return its decoded string.
The encoding rule is: k[encoded_string], where the encoded_string inside the square brackets is being repeated exactly k times. Note that k is guaranteed to be a positive integer.
You may assume that the input string is always valid; No extra white spaces, square brackets are well-formed, etc.
Furthermore, you may assume that the original data does not contain any digits and that digits are only for those repeat numbers, k. For example, there won't be input like 3a or 2[4].

**Examples:**

>s = "3[a]2[bc]", return "aaabcbc".
s = "3[a2[c]]", return "accaccacc".
s = "2[abc]3[cd]ef", return "abcabccdcdcdef".

首先需要划分实体，什么是一个实体？如下：

1. 字母串  abc, b, c

2. 以数字开头并包含一个实体的实体比如:  num[E]

3.实体的连接是一个实体： EEE

那么我们采用递归下降的方法来解决这个问题。首先需要给出字母串和开始的位置找出这个整体的边界，比如s = 2[abc]cd，输入字符串s和第一个位置0，那么返回5也就是]的位置，0-5形成一个整体2[abc]。划分代码如下：

```C
int EndPos(string & s, int b){
    if(b >= s.size()) return -1;            //遇到边界返回-1
    int type = 0;                            //区分是字符串实体还是num[xxx]形式的实体
    type = (s[b] >='a' && s[b]<='z' || s[b]>='A' && s[b]<='Z')?0:1;
    if(!type){                                //字符串实体
        int endpos = b-1;
        for(int i = b;i<s.size();i++){
            if((s[i] >='a' && s[i]<='z' || s[i]>='A' && s[i]<='Z')){
                endpos++;
            }else
                break;
        }
        return endpos;
    }

    int cnt = 0;                            //num[xxx]实体
    for(int i = b;i<s.size();i++){
        if(s[i] != '[' && s[i] != ']') continue;
        if(s[i] == '['){
            cnt++;
        }
        else if(s[i] == ']'){
            cnt--;
        }
        if(cnt == 0){
            return i;
        }
    }
    return -1;
}

```
现在就需要对上面给出的三个推导式分情况讨论。

1. 如果DFS(s, begin)中，以beign为开头的实体是一个边界，那么返回空串，边界的类型包括]和begin >= s.size()两种类型。

可以计算FOLLOW(E) = { ] ,  $，E}三种类型（FOLLOW在编译原理里指的是跟随在非终结符后面的字符，简单来说就是实体E后面可以跟什么样的字符。（这里扩充一下FOLLOW的含义，实际上FOLLOW里只能跟终结符）

2. 如果DFS(s, begin)中，以beign为开头的实体是一个字符串，那么直接返回即可。

3. 如果DFS(s,begin)中，以begin开头的实体是一个num[xxx]型的实体，则首先分割出实体的边界endpos，然后获取num的数值，再调用num[xxx]中的xxx获得的字符串，循环相加获得(xxx)^num，最后返回res+DFS(s, endpos+1);

代码如下：

```C
string DFS(string & s, int beg){
    if(beg >= s.size()) return "";            //遇到边界
    if(s[beg] == ']') return "";
    string res;
    int endpos = EndPos(s,beg);                //划分实体的位置
    string orgstr = s.substr(beg,endpos-beg + 1);
    if(regex_match(orgstr, numpat)){            //如果是普通字符串则直接返回
        res = res + orgstr;
        res = res + DFS(s,endpos+1);            //这里是使用了推导式 E-> E E E...
        return res;
    }

    int bpos = beg;
    int cnt = 0;
    for(int i = bpos;i<=endpos;i++){
        if(s[i] >= '0' && s[i] <= '9'){            //计算num
            cnt = cnt * 10 + s[i] - '0';
            continue;
        }
        if(s[i] == '['){                            //对num[xxx]进行调用分析，获得xxx
            string ss = DFS(s,i+1);
            for(int i = 0;i<cnt;i++) res = res + ss;
            i = EndPos(s,i-1) + 1;                //这里进行跳出
        }
    }
    bpos = endpos + 1;
    res = res + DFS(s,bpos);                使用推导式E -> EEE...
    return res;
}
```

以下是完整的代码：

```c++
#include<cstdio>
#include<string>
#include<regex>
#include<iostream>
 
using namespace std;
 
regex numpat("[a-zA-Z]*");
 
 
 
smatch result;
 
class Solution {
public:
    string decodeString(string s) {
        string res = DFS(s,0);
        return res;
    }
 
    string DFS(string & s, int beg){
        if(beg >= s.size()) return "";
        if(s[beg] == ']') return "";
        string res;
        int endpos = EndPos(s,beg);
        string orgstr = s.substr(beg,endpos-beg + 1);
        if(regex_match(orgstr, numpat)){
            res = res + orgstr;
            res = res + DFS(s,endpos+1);
            return res;
        }
 
        int bpos = beg;
        int cnt = 0;
        for(int i = bpos;i<=endpos;i++){
            if(s[i] >= '0' && s[i] <= '9'){
                cnt = cnt * 10 + s[i] - '0';
                continue;
            }
            if(s[i] == '['){
                string ss = DFS(s,i+1);
                for(int i = 0;i<cnt;i++) res = res + ss;
                i = EndPos(s,i-1) + 1;
            }
        }
        bpos = endpos + 1;
        res = res + DFS(s,bpos);
        return res;
    }
 
    int EndPos(string & s, int b){
        if(b >= s.size()) return -1;
        int type = 0;
        type = (s[b] >='a' && s[b]<='z' || s[b]>='A' && s[b]<='Z')?0:1;
        if(!type){
            int endpos = b-1;
            for(int i = b;i<s.size();i++){
                if((s[i] >='a' && s[i]<='z' || s[i]>='A' && s[i]<='Z')){
                    endpos++;
                }else
                    break;
            }
            return endpos;
        }
 
        int cnt = 0;
        for(int i = b;i<s.size();i++){
            if(s[i] != '[' && s[i] != ']') continue;
            if(s[i] == '['){
                cnt++;
            }
            else if(s[i] == ']'){
                cnt--;
            }
            if(cnt == 0){
                return i;
            }
        }
        return -1;
    }
 
};
 
int main(){
    string ss;
    cin>>ss;
    Solution s;
    cout<<s.decodeString(ss)<<endl;
}

```
