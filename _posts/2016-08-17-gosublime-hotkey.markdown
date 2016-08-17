---
layout: post
title:  "sublime text3快键键说明"
date:   2016-07-08 22:21:49
categories: sublime
tags: sublime
---
# 1.sublime text3有用的键记录：

```json
[ctrl+p]：sublime的go to anywhere，可见其强大之处，该命令可以在当前已经打开文档中跳转（如果打开文件夹，也可以在当前文件夹中跳转），配合:(行号跳转)，@(方法名跳转)，功能相当强大
[ctrl+g]：跳转到指定的行
[ctrl+r]：跳转到指定的方法
```

# 2.gosublime默认快捷键记录

在gosublime的默认键定义中，一般快键的定义如下：

```json
{
	"keys": ["ctrl+.", "ctrl+g"],
	"command": "gs_doc",
	"args": {"mode": "goto"},
	"context": [{ "key": "selector", "operator": "equal", "operand": "source.go" }]
}
```

这是gosublime跳转到声明的快键，键盘上的按法如下：  
1.按住ctrl键，然后按.号键，松开，然后再按g键  
2.按住ctrl键，同时按下.号键，然后再按下ctrl键，同时按下g键
（注意，一定要关闭输入法，否则所有的键都不灵）

有用的按键定义如下：

```
[ctrl+.,ctrl+.]：显示gosublime的功能菜单，显示gosublime的各种常用功能
[ctrl+.,ctrl+g]：跳转到相应的方法定义，不是很好用
[ctrl+shift+leftclick]：相当于[ctrl+.,ctrl+g]，跳转到方法定义
[ctrl+.,ctrl+O]：查看go的包定义
```


[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
