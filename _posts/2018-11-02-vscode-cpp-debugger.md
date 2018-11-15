---
layout: post
title:  "OSX中VSCode C++编译调试环境搭建"
author: kevin
categories: tools
tags: vscode c++ debugger
---
* content
{:toc}

#### 说明
* 系统： OSX 10.13.6
* VSCode: 1.28.0

#### 插件
安装以下插件。版本号仅供参考。
* C/C++ cpptools （0.20.1）
	* C/C++ IntelliSense, debugging, and code browsing.
* C/C++ Clang (0.2.3)
	* Completion and Diagnostic for C/C++/Objective-C using Clang command.
* Native Debug (0.22.0)
	* C/C++ IntelliSense, debugging, and code browsing.
	* 只安装上面两个时，提供的lldb-mi似乎有bug

#### 配置
* 全局C/CPP配置。F1命令模式输入：`C/Cpp: Edit Configurations`
编辑文件，如下：
```js
{
    "configurations": [
        {
            "name": "Mac",
            "includePath": [
                "${workspaceFolder}/**",
                "/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1",
                "/usr/include"
            ],
            "defines": [],
            "macFrameworkPath": [
                "/System/Library/Frameworks",
                "/Library/Frameworks"
            ],
            "compilerPath": "/usr/bin/clang",
            "cStandard": "c11",
            "cppStandard": "c++17",
            "intelliSenseMode": "clang-x64", 
            "compileCommands:": "THE_CMAKE_COMPILE_COMMANDS.json"

        }
    ],
    "version": 4
}
```
注意，以上includePath用于头文件查找跳转等，看具体情况可修改。
其中compileCommands用于帮助VSCode在源码中各种合理跳转。可以由CMake在编译时添加参数`CMAKE_EXPORT_COMPILE_COMMANDS`来生成对应的json文件，用实际路径替换它。

* 配置tasks.json。 命令模式下，`Tasks: Configure Task`
```js
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
        {
            "label": "c++",
            "command": "clang++",
            "type": "shell",
            "args": [
                "main.cpp", //这些就是编译时的一些参数。单文件编译可以用${file}
                // "${file}"
                "-std=c++11",
                "-g"
            ],
            "presentation": {
                "echo": true,
                "reveal": "always",
                "focus": false,
                "panel": "shared"
            }
        }
    ]
}
```

* 配置launch.json。这里使用lldb进行调试。
```js
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "c/c++ Launch",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/a.out",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": true,
            "miDebuggerPath": "/usr/local/bin/lldb-mi",
            "MIMode": "lldb",
            "preLaunchTask": "c++",
        }
    ]
}
```
注意比默认配置添加了`miDebuggerPath`，原因是发现cpptools使用默认的lldb-mi（在`~/.vscode/extensions/ms-vscode.cpptools-0.20.1/debugAdapters/lldb/bin/`下）对于vector等STL的大小一直显示0。对应版本如下：
```
[~/.vscode/extensions/ms-vscode.cpptools-0.20.1/debugAdapters/lldb/bin]$ lldb-mi --version
Version: GNU gdb (GDB) 7.4
(This is a MI stub on top of LLDB and not GDB)
All rights reserved.
```
所以这是为什么安装第三个插件原因，安装后
`ln -s /Applications/Xcode.app/Contents/Developer/usr/bin/lldb-mi /usr/local/bin/lldb-mi`
然后配置上述`miDebuggerPath` 编译及调试功能即OK。

#### 示例图
![Alt text](/assets/201811/vscode-cpp-debugger.png)



#### 参考
* [在macOS下使用Visual Studio Code进行C/C++开发](https://zhuanlan.zhihu.com/p/25006796)
* [Mac在VSCode中搭建C/C++环境](https://www.jianshu.com/p/050fa455bc74)
