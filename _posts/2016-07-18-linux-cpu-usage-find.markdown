---
layout: post
title:  "linux下查找java进程占用CPU过高的线程"
date:   2016-07-17 22:21:49
categories: 性能问题
tags: Java
---
1.首先查找占用CPU最高的java进程，在linux下，可以使用top等命令。，比如查找到的进程ID为1422

2，运行命令top -H -p 1422，该命令的-H参数表示显示所有该进程的线程，-p代表进程号，显示如下：
![cpu top image](/imgs/20160718/top1.png)

3.在top命令中，按照CPU使用进行排序，把使用CPU最高的线程找出来，假设该线程为40102

4.把40102转化为16进制，40102的16进制表示为9CA6

5.使用jstack命令打印出1422进程的线程的调用堆栈信息，查找出nid=9CA6的线程，如下图![cpu top image](/imgs/20160718/jstack1.png)

6.经过以上步骤，我们已经找出了在这个进程中使用CPU最高的线程信息。

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
