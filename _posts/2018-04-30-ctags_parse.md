---
layout: post
title:  "ctags自定义解析"
author: kevin
categories: vim vim
tags: ctags vim
excerpt: 
---
#### 基本使用
对于vim党，coding时比较常见使用ctags进行代码跳转。
一般生成tags的命令形如：
```
/usr/local/bin/ctags --languages=c++,proto --c++-kinds=+px  --fields=+iaKSz  --extra=+q "$@"
```
有些文件官方不支持解析([支持列表](http://ctags.sourceforge.net/languages.html))，这时候可以使用其自定义规则。
具体规则可参考[这里](http://ctags.sourceforge.net/ctags.html)。

#### 新增对protobuf的解析
新增 ~/.ctags，内容如下
```
--langdef=PROTO
--langmap=PROTO:.proto
--regex-PROTO=/^[ \t]*message[ \t]+([a-zA-Z0-9_\.]+)/\1/m,message/
--regex-PROTO=/^[ \t]*(required|repeated|optional)[ \t]+[a-zA-Z0-9_\.]+[ \t]+([a-zA-Z0-9_]+)[ \t]*=/\2/f,field/
```
这里定义了新的语言PROTO, 其后缀.proto。之后定义了二条规则解析message, field。对于项目中使用的enum定义的内容无法跳转，照葫芦画瓢写一个enum的解析。
核心主要是regex-XXX的模式匹配，文档说明已经挺清晰了:

![Alt text](/assets/1525081213113.png)

```
--regex-PROTO=/^[ \t]*([A-Z0-9_]+)[ \t]*=/\1/f,enum/
```

最后修改命令增加对于proto文件的感知和解析：
```
/usr/local/bin/ctags --languages=c++,proto --c++-kinds=+px  --fields=+iaKSz  --extra=+q "$@"
```

一切OK。
