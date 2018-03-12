---
layout: post
title:  "程序中的错误处理"
author: kevin
categories: backend
tags: error code
excerpt: 
---

* content
{:toc}
读耗子哥的专栏整理记录的一些东西。

#### 返回值
##### 传统错误检查
* 返回值 + 错误码
	* errno+errstr
	* atoi() 无errno设置，返回值不能携带错误信息，不能转换时的行为undefine
	* strtol()解决atoi()问题，设置了errno


##### Windows的参数返回
* 错误统一有返回值返回， HRESULT类型
* 入参+出参

##### 多返回值
* Go等语言
	* 常见的返回result, err两个值
* 正常逻辑被大量错误处理打得凌乱

#### 资源清理
* goto fail
* C++ RAII
* Go defer

#### 异常捕捉处理
* try-catch-finally
	* 优势
		* 业务代码，异常处理代码，资源清理代码分门别类，代码干净
		* 不需要判断返回值，链式调用
    * 缺陷
	    * 性能略有影响
	    * 异步运行时，多线程下抛出的异常无法被捕捉


#### 返回码和异常的使用场景
* 错误归类
	* 资源错误
	* 程序错误
	* 用户错误
* 逻辑分类
	* 不期望发生的事， 可以使用异常
	* 觉得可能发生的事，使用返回码

