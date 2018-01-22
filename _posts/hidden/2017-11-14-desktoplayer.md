---
title: Desktoplayer样本分析及专杀工具
comments: true
verifid: 2017111401
layout: default
category: hidden
---

* TOC
{:toc}

## 概述

- 时间： 2017-11-26
- 样本来源：课堂作业
- 本文档讲述关于rmnet家族DesktopLayer样本的行为、清除方法、技术细节；


## 名词解释

- 挂钩(Hook):通过对源码的修改，达到更改原来代码执行流程，进而执行一些原来代码不具有的行为，这种修改手段可称为“挂钩(Hook)”。

- 代码注入:通过在原来的进程内存空间中添加一些额外代码并设定一定的触发条件，使得添加代码得以执行，最终实现让原来进程执行新添加的代码，以达到执行一些原来程序所不具有的功能的目的。原来进程中新添加的代码叫“注入代码”，而这种添加“注入代码”的行为可称为“代码注入”。

## 相关文件

- DesktopLayer.exe.v    : 样本；
- upDesktopLayer.exe.v  : 脱壳后的样本；
- DesktopLayer.exe.new.v：由样本释放到C盘新建文件夹的文件；
- dmlconf.dat：样本在iexplorer.exe 目录下创建的文件，用于写入系统时间信息				 和网络连通时间差；
- Hook ZwWriteVirtualMemroy Data.txt：API ZwWriteVirtualMemroy被Hook前后					的函数地址机器码，以及Hook过程中的 15 字节堆空间数据。

- 感染EXE和DLL文件过程中的文件: (Exe_Dll文件夹)
    - Exe_Sample.exe.v:一个满足感染条件且未被感染过的exe文件；
    - Exe_Sample_InFected.exe.v:被感染后的exe文件；
    - Exe_Sample_InFectedSrv.exe.v:被感染文件在运行时释放出来的文件；
    - Data2Exe_1.bin、Data2Exe_1.bin：感染时向EXE和DLL中写入的两段数据；
    - Data2ExeAsm.txt:被感染文件入口点代码行为分析文件；
	
- 感染HTML和HTM文件过程中的文件: (Html_Htm文件夹)
    - Html_Sample.html.v:一个满足感染条件且未被感染过的HTML文件；
    - Html_Sample_InFected.html.v:被感染后的HTML文件；
    - Data2Html.bin:感染时向HTML文件中追加的数据；

- 感染可移动磁盘文件过程中的文件: (RemovableDisk文件夹)
    - RECYCLER_1.rar:含有自动运行文件和写入的exe文件(RECYCLER 的二级文					  件名和EXE文件名均具有随机性)。
	


- 以上文件中相关文件的MD5和SHA1值如下：
    - 样本DesktopLayer.exe.v<br>MD5: FF5E1F27193CE51EEC318714EF038BEF<br>SHA1: B4FA74A6F4DAB3A7BA702B6C8C129F889DB32CA6	

    - 释放的文件DesktopLayer.exe.new.v、Exe_Sample_InFectedSrv.exe.v和感染
    - 可移动磁盘时写入的exe文件其实是同一个文件：<br>MD5: 240B869098AF035A4FD7968308C86EDD<br>SHA1: 0798717EEE86F30EBF8BD11F3C8F4C2B473BC724

	-向 HTML和HTM文件中写入的数据Data2Html.bin:<br>MD5: 36748DD7B6C9EFBF0A6371C307DC2D2C<br>SHA1: 1E3934254D07F67D54DFD5D69F86DDAC200BD39F

## 行为预览

样本名称：DesktopLayer.exe

样本类型：Win32.Ramnit(由360云查杀确定)

样本大小：55.0 KB(56320 字节)

传播方式：本地磁盘文件感染和可移动磁盘传播
		
样本具体行为：该样本在运行过程中分3个阶段，下面详细说明每个阶段的关键行为。
- Phase1:勘测本机环境，确保样本的关键行为能顺利执行:
    1. 通过注册表获取 iexplorer.exe 程序的目录，并在该目录下查找验证 iexplorer.exe 程序是否存在；
    2. 尝试在C盘的相应位置(共7个备选路径，详见后文)创建文件夹"Microsoft",并复制新样本到新创建的文件夹中， 其中复制到新文件夹下的新样本内容即为手动脱壳后的upDesktopLayer.exe.v的内容；
    3. 启动新文件夹下的新样本程序。

- Phase2:通过 Hook ZwWriteVirtualMemory 完成对 iexplorer.exe的注入，关键代码均在 iexplorer.exe中。
    1. 验证特定文件夹(Phase1创建的文件夹)下是否存在样本文件；  
    2. 获取 ntdll.dll 的一些导出函数地址保存到全局变量中,便于后续代码中直接调用；
    3. Hook ZwWriteVirtualMemory
    4. 调用 CreateProcessA 来启动 iexplorer.exe 进程，在该 API内部会调用 ZwWriteVirtualMemory 进而完成对 iexplorer.exe 的注入；

- Phase3: iexplorer.exe的入口点被修改之后程序的原流程发生了变化，这使得 iexplorer.exe其实变成了一个执行注入代码的壳。注入代码的行为如下：
    1.	填写内存 PE 的 IAT 并处理节数据:
    2.	初始化 SOCKET
    3.	获取系统磁盘信息、版本信息和本地语言环境信息，用于向远程发送,并接收远程数据； 
    4.	创建6个线程，完成核心工作，关于6个线程的功能描述如下：

- Thread1: 20017ACA功能：
    - 每隔 1 秒就打开注册表项：HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
    - 并读取 Userinit 的键值，然后检查样本文件目录(c:\programfiles\microsoft\desktoplayer.exe) 是否在键值中，如果不在的话就将样本文件目录追加到该键值中,以达到开机启动样本的目的。
        
 
- Thread2: 20017626功能：
    - 间歇性的测试与 google.com 的 80 端口、bing.com 的 80 端口、yahoo.com 的 80 端口的连通性，只要有一个连通，就不再测试后面的网址并在全局变量 2001A23B 处保存两次能够连通的时间差(秒单位)。
        
- Thread3: 2001781F功能：
    - 每分钟向 "C:\Program Files\Internet Explorer\dmlconf.dat" 中写入 16 字节的数据，前8字节为系统时间，接着是 4 字节数据是两次连通特定网站的时间差， 最后 4 字节数据始终为 0。
            
- Thread4: 2001790C功能：
    - 每10分钟向 "fget-career.com的443端口" 发送当前系统时间信息以及含有本机信息的字符串，并接收 "fget-career.com" 发回的数据。
            
- Thread5: 20016EA8功能：
    1. 对 DRIVE_FIXED 类型的磁盘进行遍历感染，感染方式：
    2. 对 html和htm文件的感染方式:先检查文件内容的最后9字节数据是不是 “"</SCRIPT>” 以此来判断该文件是否被感染过，如果没有被感染，则在文件末尾添加数据，数据具体的内容可以本文档的相关文件中提取。 

    3. 对 EXE 或 DLL 文件的感染方式：
        1. 查看该文件的导入表中是否有按名称导入 “LoadLibraryA” 和 "GetProcessAddress" 这两个函数，有的话就获取该函数在 IAT 项中的 RVA, 没有的话不感染；
        2. 查看该文件节表后是否有一个节表大小的全0可用空间，如果有就利用该位置添加一个新节(添加的新节名称为 ".rmnet")，否则不感染;
        3. 对 “LoadLibraryA” 和 "GetProcessAddress" 的函数地址进行重定位，方便在注入代码中进行调用；
        4. 修改程序员入口点为新节地址，更改程序执行流程;
        5. 向源文件中写入两段数据，写入的数据见文件。
        6. 20016AA9  E8 DDFCFFFF  CALL 2001678B 该调用是处理 exe 和 dll 文件的关键调用
        7. 2001633E  E8 4A070000  CALL 20016A8D  该句是核心感染过程

                                      
- Thread6: 20016EC2功能：
    1. 每10秒钟遍历一次所有磁盘，当磁盘类型为可移动磁盘时，对该磁盘进行感染：
    1. 感染 DRIVE_REMOVABLE 磁盘的方式:首先判断该磁盘是否被感染过,如果已经被感染过则不再感染，否者进行关键感染行为。
    1. 对是否已经感染的判断:
        1. 磁盘根目录存在 "autorun.inf" 文件；
        2. "autorun.inf" 文件大小大于 3 字节；                                                        		3:"autorun.inf" 文件头3字节内容为 "RmN"
    1. 以上3条都满足时，表明该可移动磁盘已经被感染过。
    1. 对没有感染过的磁盘进行的感染行为:
        1. 磁盘根目录创建"RECYCLER"文件夹并设置属性为 HIDDEN；
        2. "RECYCLER"文件夹下创建子文件夹并设置属性为 HIDDEN；
        3. 子文件夹下创建文件"AyZIKwEU.exe"(文件名不唯一，具有随机性)并写入数据；写入的数据其实也是之前	释放出来的新样本的数据；
        4. 磁盘根目录创建 "autorun.inf" 并设置属性为 HIDDEN；
        5. 分4次写入数据到 "autorun.inf" 。达到可移动磁盘插入电脑后可以自动运行新样本的目的
 
## 清理方式
1. 使用字符串"KyUffThOkYwRRtgPP" 创建互斥体，如果互斥体已经存在，说明已经有样本在运行，此时需要遍历系统所有进程，查找名称为"DesktopLayer "和"iexplore "的进程:
    1. 对于"DesktopLayer "进程:直接结束；
    1. 对于"iexplore "进程:如果进程空间的 20010000 地址为有效地址，则直接结束进程，同时删除iexplore目录下的dmlconf.dat文件。
2. 依次在以下目录下查找"Microsoft"目录，如果找到该目录，则删除该目录及目录下的"DesktopLayer.exe"文件。
    1. "C:\Program Files\ ";	
    2. "C:\Program Files\Common Files\ ";
    3. "C:\Documents and Settings\Administrator\ ";
    4. "C:\Documents and Settings\Administrator\Application Data\ ";
    5. "C:\WINDOWS\system32\ ";
    6. "C:\WINDOWS\ ";
    7. "C:\DOCUME~1\ADMINI~1\LOCALS~1\Temp\";

3. 读取HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon的Userinit 的键值，并判断键值内容的最后一个启动项中是否包含"desktoplayer.exe"，如果包含，则删除最后一个启动项。

4. 遍历全盘文件，进行查杀：
    - 对于 DISK_FIXED类型的磁盘，在遍历时可以避开系统目录和Windows目录,对于EXE文件,如果文件MD5和特征文件的MD5匹配，则直接删除；
    - 对于EXE、DLL，如果节表中含有".rmnet"节，则可判定文件已经被感染，可由用户决定是删除文件还是修复文件(修复办法:删除".rmnet"节并修复入口点)；对HTML、HTM文件，可以通过文件最后9 字节内容是否是"</SCRIPT>"来判断文件是否被感染，文如果文件已被感染，则由用户决定是删除文件还是修复文件(修复办法:删除文件"<SCRIPT Language=VBScript>"之后的内容)。
    - 对于 DISK_REMOVABLE类型的磁盘，如果磁盘根目录有 "autorun.inf"文件且文件头3字节内容为"RmN"，则可判定该磁盘已经被感染，需要从该文件总提取住exe文件的路径，然后先删除"autorun.inf"文件，再删除exe 文件。
