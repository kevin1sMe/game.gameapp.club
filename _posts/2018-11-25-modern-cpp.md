---
layout: post
title:  "Modern C++实践"
author: kevin
categories: c++
tags: c++ c++11 
---
* content
{:toc}

> 更新自己C++的知识库，利用新标准C++11/14等去写一些用例，掌握基础语法。代码放在github中。[这里][1]可查看，后续持续更新，一作整理，二作备忘。我将已经在github中有所涉及的依据深度标注不等数量的`+`。


#### 主要特性
* lambda表达式 +
	* [capture] ( params) -\> ret { body };
* 线程支持
* 元组类型 +
* 正则表达式库
* 智能指针 +
* 右值引用，移动语义和完美转发


#### 类语法糖
* 区间迭代(range-based for loop) +
* 类型推导auto/decltype +
* 返回值型别尾序语法(后置返回值类型) +
* 显式重写override和final
* 空指针常量nullptr
* 变长参数模版
* 删除函数

#### 其它知识点
* enable\_if 模版类型推导相关 ++
* enable\_shared\_from\_this 智能指针传递相关 +
* enum class 类型安全的enum +



[1]:	https://github.com/kevin1sMe/ProgrammingPractice