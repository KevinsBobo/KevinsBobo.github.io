---
title: Frida使用过程记录
comments: true
verifid: 2017081001
layout: default
category: 随笔
---

> 使用环境`Ubuntu16.04 with Python3.5 / Win7 with Python3.6`<br>[官方文档 - https://www.frida.re/docs/](https://www.frida.re/docs/)<br>[练习代码 - https://github.com/KevinsBobo/frida_usage_record](https://github.com/KevinsBobo/frida_usage_record)

* TOC
{:toc}

## 1. 安装

需要通过`sudo easy_install3 -i http://pypi.douban.com/simple/ Frida`安装；不知道为什么`pip`安装一直卡在运行`setup.py`那里

## 2. 安装完成测试

> linux 在以普通用户身份运行脚本前需要执行这条命令:`sudo sysctl kernel.yama.ptrace_scope=0`（该命令的作用是临时的），否则必须通过`root`身份运行，这样不便于后期脚本的调试

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

> `frida/invincible.py` <br>[https://github.com/KevinsBobo/frida_usage_record/blob/master/frida/invincible.py](https://github.com/KevinsBobo/frida_usage_record/blob/master/frida/invincible.py)

```python
'''
example: 修改game.exe为无敌模式
'''

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
    # 读内存 read_bytes(address, bytes)
    print(hex(session.read_bytes(0x00403616, 1)[0]))
    # 写内存 write_bytes(address, data)
    # 错误 frida.core.RPCException: Error: access violation accessing 0x403616
    # 用 js 脚本的方式写内存也是同样地错误
    # 但 js 脚本在写内存前调用 Memory.protect(ptr("0x00403616"), 8, 'rw-'); 就没问题
    # 而 py 脚本没有这个 API
    session.write_bytes(0x00403616, b'0xeb')    

    with codecs.open('./invincible.js', 'r', 'utf-8') as f:
        source = f.read()

    script = session.create_script(source)
    script.on('message', on_message)
    script.load()
    session.detach()


if __name__ == '__main__':
    # 进程PID或进程名
    target_process = 'game.exe'
    main(target_process)
```

## API

### Python

Python API 较为简单，重点是和JavaScript通信的消息回调

### JavaScript

对进程的主要操作都在js脚本中

- 写内存前要先修改内存属性 `Memory.protect(ptr("0x00403616"), 8, 'rw-');`，参见`frida/invincible.js`<br>[https://github.com/KevinsBobo/frida_usage_record/blob/master/frida/invincible.js](https://github.com/KevinsBobo/frida_usage_record/blob/master/frida/invincible.js)

调用函数：

- 通过`NativeFunction(...)`来绑定函数

    ```javascript
    // 绑定
    var thiscall_func = new NativeFunction(ptr("0x0041153C"), // 函数地址
                                       'int',             // 返回值类型
                                       ['pointer', 'int'],// 函数参数（__thiscall的第一个参数为this指针）
                                       'thiscall'         // 调用约定
                                       );

    // 调用并打印返回值
    console.log(thiscall_func( ptr('0x00421360'), 0xb ));
    ```

- 通过`Interceptor.replace` + `NativeCallback(...)`来替换函数

    ```javascript
    Interceptor.replace(ptr("0x0041153C"), new NativeCallback(function (ecx, stack1) {
        console.log(ecx);
        console.log(stack1);
        return 1; // thiscall_func(ecx, stack1);
    }, 'int', ['pointer', 'int'], 'thiscall'));
    ```

    - `NativeCallback(...)`的参数和`NativeFunction(...)`的相同，只是在使用时直接把一个匿名函数写在了第一个参数的位置，相当于函数地址，也可以在其他地方写好一个函数将函数名写在这里

    - 在替换之后要不要继续调用原函数或将调用原函数的返回值返回由自己决定

- 更多调用例子参见GitHub仓库中的`frida/example.js`, `target/example.cpp`, `frida/call_plant.js`, `testcall.js`, `target/target.cpp`

- 支持的参数类型和调用约定

    ```python
    ### Supported Types
    -  void
    -  pointer
    -  int
    -  uint
    -  long
    -  ulong
    -  char
    -  uchar
    -  float
    -  double
    -  int8
    -  uint8
    -  int16
    -  uint16
    -  int32
    -  uint32
    -  int64
    -  uint64

    ### Supported ABIs
    -  default

    -  Windows 32-bit:
        -  sysv
        -  stdcall
        -  thiscall
        -  fastcall
        -  mscdecl
    - Windows 64-bit:
        -  win64
    - UNIX x86:
   -  sysv
        -  unix64
    - UNIX ARM:
        -  sysv
        -  vfp
    ```

### 小总结

- 牛逼的一点：可以hook任何API或程序内部函数，调用其任意函数并获取及改变其参数

- 在`win`下，注入的`DLL`文件是一个命名为`frida-agent-xx.dll`，在`Linux`下注入的`so`文件则命名为`frida-angent-xx.so`，这些注入文件虽然是临时的，但二进制内容是相同的，所以这里猜测`js`脚本最终是通过这个代理模块来执行相应操作的

- 猜测`js`脚本实际上并没有注入到目标进程中，通过代理来进行读写数据和函数调用，最终以回调的方式触发，然后再通过`send(...)/recv(...)`的方式和py脚本进行`异步`通信，py脚本操作或运算完再将结果发送给js脚本，此时js脚本根据结果再通过代理执行相应操作或直接将结果写入宿主进程内存中

- 根据网上找到的资料了解到，注入到目标进程中的是`JavaScript 引擎`，在版本9之前前用的是谷歌的v8引擎，在之后改用了自己研发的`Duktape`引擎

## Tools

### frida CLI

`frida`命令行工具，在这里可以执行所有`JavaScript API`并且会有提示和补全

```shell
# 打开进程
# win
> frida *calc*
# linux
$ frida '*calc*'

# 打开进程并加载js代码
# win
> frida -l xxx.js *calc*
# linux
$ frida -l xxx.js '*calc*'
```

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

> 在linux下一开起程序进程就被终止了：`segmentation fault (core dumped)`（分段故障）<br><br>在win下只分析到了线程数量（有可能是分析时间太短）；分析期间目标程序非常卡，并且无法正常停止分析，除非关闭目标进程<br><br>所以这个工具应该是用于分析移动端的