---
title: 32位环境下VC/VS除法的8种汇编形态及传说中的MagicNumber
comments: true
verifid: 201711101203
layout: default
category: C/C++原理探索
---

## 

* TOC
{:toc}

### 1. 8 / n 常数除变量

> 直接上除法指令

```asm
...
mov    eax, 8          // 8 / nVar
mov    esi, ds:printf
cdq
idiv    [esp+14h+nVar_4]
...
```

### 2. 有符号变量除正负2的幂

#### 正

```asm
...
mov    eax, [esp+1Ch+nVar_4] // nVar / 8
cdq
and    edx, 7
add    eax, edx
sar    eax, 3
...
```

##### 公式

> 向零取整：正数除法向下取整，负数除法向上取整；移位属于向下取整

![173957526](/assets/img/win32_div_optimization.assets/173957526-1585136079570.png)

##### 特征

```asm
mem -> nVar
c   -> 8
m   -> 3 -> 2^3 == 8

...
mov eax, mem       // 将数据移动到eax寄存器
cdq                // cdq符号位扩展到edx
and edx, c - 1     // and后正数edx结果为0，负数edx结果为c - 1
add eax, edx       // 最终，正数不变，负数 + (c-1)
sar eax, m         // 有符号右移替代除法
...
```

##### 还原

不用在意变量为负数的情况，直接拿出移位数，转化为`nVar / 2^m`

#### 负

```asm
...
mov    eax, [esp+0Ch+nVar_4]  // nVar_4 / -8
cdq
and    edx, 7
add    eax, edx
sar    eax, 3
neg    eax                    // 只多了个求补，相当于符号取反
...
```

### 3. 无符号变量除正2的幂

> 直接右移

```asm
...
mov    edi, [esp+24h+nArgc] ; nArgc / 16
mov    ecx, edi
shr    ecx, 4
...
```


### 4. 无符号变量除正非2的幂（且非7及其倍数）

```asm
mov    eax, 0AAAAAAABh // uArgc / 3
mul    edi
shr    edx, 1
```

#### 公式

> a为无符号正数，c为正数常量，n由编译器决定、越大误差越小，m就是传说中的MagicNumber（关于这个最后再议）

![174651309](/assets/img/win32_div_optimization.assets/174651309.jpg)

#### 特征

```asm
M -> MagicNumber -> 0AAAAAAABh
n -> 移位数
N -> 再次移位数（如果mul的操作数为edi，则已经右移了32为，后面shr是再次移位，还原时需注意）

mov eax, M
mul reg/mem  // a * m
shr edx, N   // a * m >> 32+N -> a * m >> n
```

#### 还原

> `m = 2^n / c  =>  2^n / m = c`<br>以上面的`uArgc / 3`为例

- 首先拿到魔法数`M`->`0AAAAAAABh`
- 然后分析移位数`n`（mul操作数为edi，已经移了32位，因此`n = 32 + N`）-> `32 + 1 = 33`
- 套用`m / 2^n = c`还原->`2^33 / 0AAAAAAABh = c`得到结果为非整数，向上取整后`c = `

#### 无符号变量除负非2的幂

```asm
mov    eax, 40000001h   // uArgc / -3
mul    [esp+0Ch+uArgc]
shr    edx, 1Eh
```

和无符号变量除正非2的幂的特征一样（因为有无符号混除会被视为无符号数，在运算时`uArgc / -3`相当于`uArgc / FFFFFFFDh`）

所以反推回来的c值：无符号为：`4294967293/FFFFFFFDh`，有符号位：`-3`


### 5. 无符号变量除7

```asm
...
mov    ecx, [esp+0Ch+uArgc]  // uArgc / 7
mov    eax, 24924925h
mul    ecx
sub    ecx, edx
shr    ecx, 1
add    ecx, edx
shr    ecx, 2
...
```

#### 公式

首先根据上面的汇编可以推出（假定24924925h为MagicNumber）：

- `mul ecx`这一句执行后`ecx`和`eax`相乘，结果在`edx.eax`，而直接拿出`edx`进行后面的运算，相当于将`ecx`和`eax`相乘的结果`右移32位`（除`2^32`）
  
  ![0.7687497044655083](/assets/img/win32_div_optimization.assets/0.7687497044655083.png)
  
- `sub ecx, edx`
  
  ![0.477840983128653](/assets/img/win32_div_optimization.assets/0.477840983128653.png)
  
- `shr ecx, 1`
  
  ![0.7721911646786184](/assets/img/win32_div_optimization.assets/0.7721911646786184.png)
  
- `add ecx, edx`
  
  ![0.0036086899529590433](/assets/img/win32_div_optimization.assets/0.0036086899529590433.png)
  
- `shr ecx, 2`
  
![177238570](/assets/img/win32_div_optimization.assets/177238570.png)
  
- 化简：
  
![177688882](/assets/img/win32_div_optimization.assets/177688882.png)
  
- 推导：
  
![177878829](/assets/img/win32_div_optimization.assets/177878829.png)
  
- 结论：经过上面的推导我们可以发现`2^32 + M`相当于`MegicNumber`发生了进位，最高位超出了32位，因此需要通过加上`2^32`来将`MegicNumber`还原成正确的值。而最终通过汇编代码表现的是上面那堆公式的反推过程，将复杂的公式反推至不会发生溢出的汇编代码

#### 特征

```asm
mov    eax, imm
mul    reg
sub    reg, edx
shr    reg, 1
add    reg, edx
shr    reg, 2
```

#### 还原

和前面无符号变量除正非2的幂的还原方式一样，只是这里的`n`为直接使用`edx`产生右移32位再加上后面两次右移的结果，而`MagicNumber`要加上溢出的最高位`+100000000h`：`m = 2^n / c  =>  2^n / m = c`

### 7. 有符号数除正非2的幂

#### nVar / 7

```asm
...
mov    ecx, [esp+0Ch+nVar_4]  // nVar_4 / 7
mov    eax, 92492493h
imul   ecx
add    edx, ecx               // 比无符号除正非2的幂多了这一条
sar    edx, 2
mov    ecx, edx               // 比无符号除正非2的幂多了这一条
shr    ecx, 1Fh               // 比无符号除正非2的幂多了这一条
add    edx, ecx               // 比无符号除正非2的幂多了这一条
...
```

##### 指令分析

先来分析多的第一跳指令：`add edx, ecx`

- 发现`nvar / 7`的`MagicNumber`为`92492493h`（32位），很明显这个数的高位为`1`，且下面的指令是`imul`，说明`MagicNumber`被当作有符号数处理了；然而`MagicNuber`是个无符号数，这就产生了一个无符号数和有符号数相乘（将`92492493h`视为正数与有符号数`nVar`相乘）结果还要“正确”（不考虑溢出，因为这里的`imul`指令后面是单操作数，所以溢出的部分会存在`edx`中）的问题。

- 我们想要知道怎样才能保持结果是“正确”的，就必须先看“错误”的会是什么样：

    > 为简化公式，这里将32位的`92492493h`截断为16位的`9249h`，单操作数的`imul`指令的结果存放在16位寄存器`dx.ax`中

    - 上面发生“错误”的指令可以写成公式：`nVar * 9249h`，如果将`9249h`视为有符号数公式就可以进行转化和推导了：

        ```asm
        dx.ax = nVar * 9249h
              = nVar * -(10000h - 9249h)   // 一个数的补码再取负还是自身
                                           // 而这一步之后的9249h会被认为是无符号数
              = nVar * (9249h - 10000h)    // 消除外面的负号
              = 9249h*nVar - 10000h*nVar
        ```
    - 此时发现`9249h`被视为有符号数和`nVar`相乘的结果为`9249h*nVar - 10000h*nVar`（此时的`9249h`被视为了无符号），所以和我们想要的结果中多了一个`-10000h*nVar`，那么我们只要在结果中加上`10000h*nVar`得到`9249h - 10000h*nVar + 10000h*nVar`就是我们要的无符号数`9249h`和有符号数`nVar`相乘的“正确结果了”；而`10000h`超过了16位，`10000h*nVar`相当于将结果直接放到dx中

- 现在想要回到32位的场景中只需要把`10000h`扩展到`100000000h`，把`9249h`扩展到`92492493h`，把`dx.ax`扩展到`edx.eax`就可以了

- **再回头看上面的指令，执行完`imul`乘法后紧接着执行了一句`add edx, ecx`（`ecx`的值为`nVar`），那就相当于上面推导了将“错误”的结果再加上`10000h*nVar`就得到了“正确的结果”**

- 所以`add edx, ecx`这一句的目的就是为了修正`imul ecx`这一条指令将`92492493h`当作有符号数处理的问题

现在再来分析最后面多的三条指令：
```asm
...               // 前面一大堆乘的结果保存在了edx.eax中，而直接拿edx使用相当于将结果右移32位
sar    edx, 2     // 在这里再次算术右移2位，相当于结果右移34为
mov    ecx, edx
shr    ecx, 1Fh   // 将结果逻辑右移31位，相当于将最高的符号位移动到了最低为，其他位清零；即正数结果为0，负数结果为1
add    edx, ecx   // 正数：结果加0，负数：结果加1
```
##### 公式

最后三条指令的最终目的是正数结果不变，负数结果加1；这就是前面向零取整——正数向下取整、负数向上取整的公式：

![335c9d24-054a-4bce-9853-243c9a31d45f](/assets/img/win32_div_optimization.assets/335c9d24-054a-4bce-9853-243c9a31d45f.jpg)

##### 特征

```asm
...
mov    eax, imm
imul   reg
add    edx, reg
sar    edx, imm
mov    reg, edx
shr    reg, 1Fh
add    edx, reg
...
```

##### 还原

不用在意修正的那条指令和后面根据符号位调整的三条指令，和前面无符号变量除正非2的幂的还原方式一样：`m = 2^n / c  =>  2^n / m = c`


#### nVar / 5

```asm
...
mov    ecx, [esp+0Ch+var_4]  // nVar_4 / 5
mov    eax, 66666667h
imul   ecx
[         ]                  // 这里比上面少一条
sar    edx, 1
mov    ecx, edx
shr    ecx, 1Fh
add    edx, ecx
...
```

##### 指令分析

这里少的一句是修正的那句`add edx, ecx`，因为`66666667h`最高位为0，`imul`指令执行结果不会发生“错误”，所以就不需要这句了

##### 特征

根据上面的特征，判断`MagicNumber`最高位不为0时就不需要那句`add edx, reg`了

#### nVar / 3

```asm
...
mov    ecx, [esp+0Ch+var_4]  // nVar_4 / 3
mov    eax, 55555556h
imul   ecx
[         ]                  // 这里又比上面少一条
mov    ecx, edx
shr    ecx, 1Fh
add    edx, ecx
...
```

##### 指令分析

这里少的一句指令是`sar edx, 1`，这个实质上是直接使用了`edx`、原值右移32位刚刚够，所以不需要再次右移了

##### 特征

这里的特征就需要再判断有没有那句`shr edx, reg`了，没有就是右移32位

### 8. 有符号数除负非2的幂

> 这种情况其实和上面有符号数除正非2的幂是相反的，而其相反的**表现形式是`MagicNumber`要被视为负数**

#### nVar / -3

```asm
...
mov    ecx, [esp+0Ch+nVar_4]  // nVar / -3
mov    eax, 55555555h
imul   ecx
sub    edx, ecx
sar    edx, 1
mov    ecx, edx
shr    ecx, 1Fh
add    edx, ecx
...
```

##### 指令分析

前面提到这里的`MagicNuber`要被视为负数，而`55555555h`是个正数，所以就要将其修正为负数

#### nVar / -5

```asm
...
mov    ecx, [esp+14h+nVar_4]  // nVar / -5
mov    eax, 99999999h
imul   ecx
sar    edx, 1
mov    eax, edx
shr    eax, 1Fh
add    edx, eax
...
```

#### nVar / -7

```asm
...
mov    ecx, [esp+1Ch+nVar_4]  // nVar / -7
mov    eax, 6DB6DB6Dh
imul   ecx
sub    edx, ecx
sar    edx, 2
mov    ecx, edx
shr    ecx, 1Fh
add    edx, ecx
...
```