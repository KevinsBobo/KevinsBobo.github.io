---
title: 植物大战僵尸关键数据及修改日志
comments: true
verifid: 2017072201
layout: default
category: hidden
---

* TOC
{:toc}

# 关键数据

## 简体中文年度破解版（v1.0.0.1051）

### 相关数据

1. 全局基地址：`0x006A9F38`
2. 战场环境指针：`*(*0x006A9F38+ 0x768)`
3. 阳光数量：`*(*(*0x006A9F38+ 0x768) + 0x5560)`
4. 僵尸数量：`*(*(*0x006A9F38+ 0x768) + 0xA0)`
5. 植物数量：`*(*(*0x006A9F38+ 0x768) + 0xBC)`
6. 过关倒计时：`*(*(*0x006A9F38 + 0x768) + 0x5600)`
7. 植物环境指针：`*(*(*0x006A9F38 + 0x768) + 0xAC)`
8. 植物信息：`0x0069F2B0`

    此处保存结构体数组，类型如下：

    ```
    // 剩余字段未知
    +0x0    uID     // 植物种类
    +0x8    uUI     // 植物外观
    +0x10   uCost   // 所需阳光
    +0x14   uRecharge   // 冷却时间
    +0x20   pName   // 植物名称
    ```

9. 卡片信息
 
    加载关卡时，在0x004897B2处中断，此时栈顶为结构体指针。
    ```
    +0x8    uPosX   // 屏幕X坐标
    +0xC    uPosY   // 屏幕Y坐标
    +0x34   uType   // 种类
    +0x48   IsNotSelect     // 是否处于未选中状态
    ```
    注意：卡片中显示僵尸时，需要在其原种类编号的基础上加0x3C。即显示0号僵尸时，此处设置为0x3C，以此类推。

### 相关函数

#### 1. 植物种植检查（0x0040E020）

```cpp
// 返回0表示当前位置可以种植
// 否则返回错误代码
int ConflitCheck @<eax>(
    void *pThis,                // 战场环境指针
    unsigned int uIdxX,         // X坐标
    unsigned int uIdxY @<eax>,  // Y坐标
    unsigned int uPlantType     // 植物种类
);
```

#### 2. 种植植物（0x0040D120）

```cpp
int Plant @<eax>(
    void *pThis,                // 战场环境指针
    unsigned int uIdxX,         // X坐标
    unsigned int uIdxY @<eax>,  // Y坐标
    unsigned int uPlantType,    // 植物种类
    int nReserve                // 未知，恒为-1
);
```

#### 3. 删除植物（0x004679B0）

> 注意：直接调用该函数可能会导致内存访问异常。

```cpp
void DeletePlant(
    void *pPlant    // 植物指针
)
```

#### 4. 放置僵尸（0x0042A0F0）

```cpp
int PlaceZombie @<eax>(
    void *pThis @<ecx>,         // 战场环境指针
    unsigned int uZombieType,   // 僵尸种类
    unsigned int uIdxX,         // X坐标
    unsigned int uIdxY @<eax>,  // Y坐标
);
```

#### 5. 减去种植所需阳光（0x0041BA60）

```cpp
// 返回false表示阳光不足
bool SubSun @<al>(
    size_t uNeedSun @<ebx>      // 所需阳光
);
```

#### 6. 开始关卡（0x0044F560）

```cpp
void StartLevel(
    unsigned int uLevel,    // 关卡编号
    bool IsNewGame   // 是否是新游戏，false表示重新开始当前关卡
);
```

#### 7. 结束关卡（0x00413400）

```cpp
void LevelFail(
    void *pThis,    // 战场环境指针
    int nReserve    // 未知，恒为0
)
```

### 其他功能性修改

#### 1. 删除所有植物

```
.text:0041BB28 cmp [eax+141h], bl
.text:0041BB2E jz short loc_41BAE0
```
将jz指令改为nop。

#### 2. 取消自动暂停

```
.text:0044F478 jmp sub_4502C0
```
将jmp指令改为ret。

#### 3. 禁用游戏内菜单

```
.text:00450102 jz short loc_450112
```

将jz指令改为jmp 0x0045016A

#### 4. 所有僵尸后退

```
.text:0052AB25 jz short loc_52AB30
```

将jz指令改为jmp。

#### 5. 全屏秒杀僵尸

```
.text:0052AB3E jnz loc_52ABE8
.text:0052AB44 cmp dword ptr [esi+6Ch], 0FFFFFFFCh
```

以上两行代码改为0xC7, 0x46, 0x28, 0x03, 0x00, 0x00, 0x00, 0x90, 0x90, 0x90。

#### 6. 取消自动产生僵尸

```
.text:0040DDDC jb short loc_40DDE8
```
将jb指令改为nop

#### 7. 同时打开多个游戏进程
```
.text:00553EAE call ds:CreateMutexA
……
.text:00553F10 call ds:GetLastError
.text:00553F16 cmp eax, ERROR_ALREADY_EXISTS
.text:00553F1B jnz short loc_553F29
```

将jnz指令改为jmp。

#### 8. 强制载入上局存档

```
.text:004336E8 jnz short loc_433766
```
将jnz指令改为nop。

## Steam版（v1.2.0.1096）

 
### 相关数据

1. 全局基地址：`0x00731CDC`
2. 战场环境指针：`*(*0x00731CDC + 0x868)`
3. 阳光数量：`*(*(*0x00731CDC + 0x868) + 0x5578)`
4. 僵尸数量：`*(*(*0x00731CDC + 0x868) + 0xB8)`

### 相关函数

#### 1. 植物种植检查（0x00411520）

```cpp
// 返回0表示当前位置可以种植
// 否则返回错误代码
int ConflitCheck @<eax>(
    void *pThis,                // 战场环境指针
    unsigned int uIdxX,         // X坐标
    unsigned int uIdxY @<eax>,  // Y坐标
    unsigned int uPlantType     // 植物种类
);
```

#### 2. 种植植物（0x004105A0）

```cpp
int Plant @<eax>(
    void *pThis,                // 战场环境指针
    unsigned int uIdxX,         // X坐标
    unsigned int uIdxY @<eax>,  // Y坐标
    unsigned int uPlantType,    // 植物种类
    int nReserve                // 未知，恒为-1
);
```

#### 3. 放置僵尸（0x0042DE00）

```cpp
int PlaceZombie @<eax>(
    void *pThis @<ecx>,         // 战场环境指针
    unsigned int uZombieType,   // 僵尸种类
    unsigned int uIdxX,         // X坐标
    unsigned int uIdxY @<eax>,  // Y坐标
);
```

#### 4. 减去种植所需阳光（0x0041F620）

```cpp
// 返回false表示阳光不足
bool SubSun @<al>(
    size_t uNeedSun @<ebx>      // 所需阳光
);
```

### 其他功能性修改

取消自动暂停

```
.text:004540E5 push esi
.text:004540E6 call sub_455330
```

nop以上代码。



# 修改日志

> 注：标题后的代码为`git commit`记录ID

## 2017.7.22 - 修改Bug - `c4efbf`

- 一键通关时在D盘创建了一个`users.dat`文件

    - 原因：设置文件路径时，直接用文件名覆盖了整个路径

    - 修改：使用拼接的方法得到完整路径

## 2017.7.22 - 完成扫描房间功能 - `7537d7`

- 通过`WinPcap`库向局域网内所有IP的`27175`端口（植物方进入我是僵尸无尽版之后绑定的`SOCKET`端口）发送`TCP SYN`包，然后监听网卡数据，如果收到某IP的`SYN + ACK`包，说明该IP有一个游戏房间，则显示该IP

## 2017.7.20 - 增加一键设置通关存档 增加安装WinPcap选项 修改UI准备扫描房间选项 - `a7b192`

- 调整了UI结构

- 为启动器设置了UAC权限

- 完成一键设置通关文档

- 在目标主机没有安装WinPcap依赖包的情况下，可通过按钮安装（启动WinPcap安装程序，用户仍然需要手动安装）

## 2017.7.6 - 修改Bug - `20d298`

- IP写文件Bug：仅在起动器打开时向文件写入了第一张网卡的IP地址，随后选择网卡切换IP或手动切换IP无效。

    - 原因：写文件函数调用位置错误，误将其写在了`Dialog OnInitDialog()`函数中

    - 修改：将调用写文件函数放置在`启动`按钮的事件函数中

## 2017.7.5 - 增加选择IP - `25ba49`

- 所有新增代码在工程代码中都以`/* 选择IP新增 start */ ... /* 选择IP新增 end */`的注释方式标注

- 对`WpdPack`的修改：在文件`Include/pcap-stdinc.h`中注释了第69行`// #define inline __inline `

- 小问题：原工程使用`vs2015`编写的，项目属性需要进行修改

- 植物方为服务器端，`IP`默认为`0.0.0.0`，不需要根据网卡选择`IP`

- 僵尸方为客户端，需要根据网卡选择`IP`，且需要等待植物方进入`我是僵尸无尽版`之后才能进入该模式，否则连接不成功

- 发现的警告：一个结构体里有一个这样的数据成员`char szBuff[0];`，警告其不符合规范

- 发现的Bug：

    - 种植植物显示错位

    - 部分僵尸不能投放

    - 植物状态无存档，继续游戏时只剩下僵尸
