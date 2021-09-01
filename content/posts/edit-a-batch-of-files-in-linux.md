---
title: 在Linux下面批量修改文件名字以及内容
date: 2013-10-27 15:39:41
tags:
 - Linux
---
众所周知，sed和awk被称为Linux系统维护的两大瑞士军刀，但是一直没学会……最近遇到批量修改文件的场合，用到了sed，所以记录一下。

### 批量修改文件名字

	for i in `ls`; do mv -f $i `echo $i | sed 's/oldstring/newstring/'`;  done
	
其实上面已经不算是一个语句了，而是通过for循环一个个的取出所有文件名达到遍历的效果。
基本原理是使用echo取得文件名，交给sed替换字符串然后把结果（也就是新的文件名）作为参数回传给mv。
需要注意的是，在shell里面把命令运行结果当作参数引用时，是需要用到 `` （键盘esc下面那个键）把需要引用的部分包围起来的。

### 批量替换文件内容

	sed -i "s/oldstring/newstring/g" `grep -rl oldstring /path/to/dir`
	
这个其实蛮简单的。说下sed的几个参数。

	-i 表示直接把文件编辑掉（默认不会去修改文件内容）。
	双引号里面第一个s 的意思是替换，这个应该已经比较明确了。
	而最后那个g表示全部替换。

同样的，grep的结果是作为参数传给sed的，所以也需要用 `` 包围起来。其实个人感觉直接来个ls应该也能达到目的。