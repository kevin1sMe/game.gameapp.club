---
layout: post
title:  "git协同工作的几种方式"
author: kevin
categories: git
tags: git workflow
excerpt: 
---

* content
{:toc}
读耗子哥的专栏整理记录的一些东西。

#### 中心式协同工作流
* 定义：拉下远端代码，将本地代码rebase上去
* 优势：集中化管理代码
* 缺点：工作可能互相干扰

#### 功能分支协同工作流
* 定义：根据功能建立分支，先在分支上开发，最后完成后合入master。
* 优势：减少干扰

#### GitFlow协同工作流
* 定义：分五种分支，承担不同职责。

    * Master分支：主干分支，用作发布环境，每次提交都可以发布的。

    * Feature分支：功能分支，开发功能，对应开发环境。

    * Developer分支：开发分支，功能开发完成，合入此分支，并且删除功能分支。对应集成测试环境。

    * Release分支：Developer分支测试达到可发布状态，开出Release分支，做发布前准备。这样可释放Developer分支继续开发。开发和发布阻塞。当可上线状态，将此分支合入Master和Developer分支，保证代码一致性。并且删除Release。

    * Hotfix分支：修复线上bug。每个bug都需要开一个hotfix分支，修复完成后向Developer和Master合并。之后删除hotfix分支。

* 优点：通用
* 缺点：长期维护两个分支。分支合并也需要向两个进行。
* 备注：合并时–no-ff参数， no fast forward。使原本默认的直接版本指向的切换转变为在分支中重新建立一个个commit，这样版本线会清晰一些。

#### GitHub/GitLab协同工作流
* GitHub Flow
	* Fork=>开发=>同步自己库=>Pull request=>官方库
	* 缺点：没有和环境关联
* GitLab Flow
	* 添加环境分支

#### 参考
* Git协同工作流，你该怎样选（陈皓-左耳听风）
* Git分支管理策略 (http://www.ruanyifeng.com/blog/2012/07/git.html)
