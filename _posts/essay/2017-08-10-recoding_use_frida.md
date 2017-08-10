---
title: Frida使用过程记录
comments: true
verifid: 2017081001
layout: default
category: 随笔
---

> 使用环境`Ubuntu16.04 with Python3.5 / Win7 with Python3.6`<br>https://www.frida.re/docs/

* TOC
{:toc}

## 1. 安装

需要通过`sudo easy_install3 -i http://pypi.douban.com/simple/ Frida`安装；不知道为什么`pip`安装一直卡在运行`setup.py`那里

## 2. 安装完成测试

> linux 在运行脚本前需要执行这条命令:`sudo sysctl kernel.yama.ptrace_scope=0`（该命令的作用是临时的），否则必须通过`root`权限运行，这样不便于脚本的调试

命令 `frida-trace -i "recv*" -i "read*" *calc*`

- 通过进程名注入：最后一个参数为`*xxx*`

- 通过`PID`注入：最后一个参数为`PID`

## 3. 官方例子`Examples-Windows`练习

> https://www.frida.re/docs/examples/windows/<br>这是一个在目标进程中寻找`jvm.dll`的模块地址的例子，所以被注入的进程应该是个java开发的程序，例如`eclipse`、`xmind`

发现`JavaScript`的注入代码是作为字符串传递给`python API`的，所以最好单独建立`.js`文件来写注入代码来获得语法高亮和补全，但对于`Frida`的`JavaScript API`是没有高亮和补全的，这点比较令人难受

独立`.js`脚本使用方法

```python
import codecs
import frida

session = frida.attach('xxx`)
with codecs.open('./xxx.js', 'r', 'utf-8') as f:
    source = f.read()

script = session.create_script(source)

...
```

## 4. 文档`Basic Usage`练习

```python
# python2 需要引入下面的这个包
# from __future__ import print_function

import frida
import sys

def main(target_process):
    # 获取进程会话\打开进程句柄
    session = frida.attach(target_process)

    # 实验代码

    # 枚举进程模块
    print(session.enumerate_modules())
    # 枚举内存范围
    print(session.enumerate_ranges('rw-'))
    # 写内存 write_bytes(address, data)
    # 没有写成功，错误:frida.core.RPCException: Error: expected an integer
    session.write_bytes(0x663000, "\1\2\3\4")
    # 读内存 read_bytes(address, bytes)
    print(session.read_bytes(0x663000, 4))

if __name__ == '__main__':
    # 进程PID
    target_process = 4536
    main(target_process)
```

## API

### Python

Python API 较为简单，重点是和JavaScript通信的消息回调

### JavaScript

对进程的主要操作都在js脚本中

### 小总结

- 牛逼的一点：可以hook任何API或程序内部函数，调用其任意函数（对于thiscall和寄存器传参的函数还没有验证）并获取及改变其参数

- 对于注入的js脚本在宿主进程中不能执行太多操作或运算，否则会影响宿主进程的效率或运行（猜测），而通过`send(...)/recv(...)`的方式和py脚本进行`异步`通信，py脚本操作或运算完再将结果发送给js脚本，此时js脚本根据结果再执行响应操作或直接将结果写入宿主进程内存中

## Tools

### frida-trace

动态跟踪API调用

```shell
# Trace recv* and send* APIs in Safari
$ frida-trace -i "recv*" -i "send*" Safari

# Trace ObjC method calls in Safari
$ frida-trace -m "-[NSView drawRect:]" Safari

# Launch SnapChat on your iPhone and trace crypto API calls
$ frida-trace -U -f com.toyopagroup.picaboo -I "libcommonCrypto*"
```

### frida-discover

用于发现程序内部函数 - 对于没有逆向经验的来说是好东西

```shell
$ frida-discover -n name
$ frida-discover -p pid
```

> 在linux下一开起程序进程就被终止了：`segmentation fault (core dumped)`（分段故障）<br><br>在win下只分析到了线程数量（有可能是分析时间太短）；分析期间目标程序非常卡，并且无法正常停止分析，除非关闭目标进程
