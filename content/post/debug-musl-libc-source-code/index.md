---
title: '使用VScode + Musl + Clangd + Native Debug探索libc源码'
description: 实现libc源代码的无缝外部跳转和调试
date: 2024-04-10T20:51:10+08:00
tags:
    - libc
categories:
    - 配置
---

## 使用musl作为libc

Musl作为轻量级的libc实现，相比臃肿的glibc，其代码简洁优雅，易于学习。可以使用下面的方式来静态连接musl libc（如果想使用动态链接的话，需要配置`LD_LIBRARY_PATH`环境变量）。

```
gcc -static -g a.c -nostdinc -nostdlib -I/usr/local/musl/include -L/usr/local/musl/lib /usr/local/musl/lib/crt1.o /usr/local/musl/lib/crti.o /usr/local/musl/lib/crtn.o -lc
```

- `-nostdinc`: 在编译过程中不使用标准系统目录中头文件，即`/usr/include`等目录中的头文件，确保只有通过`-I`选项指定的头文件目录被包含。
- `-nostdlib`: 编译时不自动链接系统的标准启动文件和库文件。
- `-I`: 指定头文件的搜索路径
- `-L`: 指定库文件的搜索路径，用于`-lc`的搜索
- `crt1`, `crti`, `crtn`: 提供C语言运行时环境，包含用于初始化和结束程序的代码。
  - `crt1.o`: 包含程序入口点（`_start`函数），其负责设置程序的运行环境，并调用`main`函数，在`main`返回后，调用`exit`结束程序
  - `crti.o`: 包含初始化代码的 "开头" 部分。它定义了一些特殊的符号，这些符号用于标记全局构造函数和全局析构函数的开始和结束位置。
  - `crtn.o`: 包含初始化代码的 "结尾" 部分。它也定义了一些特殊的符号，这些符号用于标记全局构造函数和全局析构函数的开始和结束位置。
- `-lc`: 链接libc，在Linux环境中，会在`-L`指定的目录中寻找`libc.a`。


## 使用Clangd作为语言服务器

> 为什么选择Clangd？
>
> 相比C/C++插件，Clangd拥有更快的跳转速度

### 配置compile_commands.json

一般来说，可以直接通过`bear -- make`生成`compile_commands.json`，但是这无法跳转到musl的源代码。因此要将musl的`compile_commands.json`与自己的进行合并，这里使用`jq`命令。

```
jq -s add compile_commands.json ~/src/musl/compile_commands.json > a.json
mv a.json compile_commands.json
```

## 使用Native Debug进行调试

Native Debug提供调试session，只需要配置`launch.json`就可以启动调试。Native Debug与Clangd兼容，且同时支持LLDB和GDB调试。而C/C++插件虽然也自带了GDB调试功能，但与Clangd不兼容。

> 在编译musl时，需要保留调试信息

```json
// launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug",
            "type": "gdb",
            "request": "launch",
            "target": "./a.out",
            "cwd": "${workspaceRoot}",
            "valuesFormatting": "parseText"
        }
    ]
}
```
## 最终效果

- VScode的go to definition可以跳转到musl源代码文件
- 使用F5可以启动调试，可以进入libc函数源代码进行调试