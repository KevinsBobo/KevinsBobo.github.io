---
title: vs2013环境下编译libuv-1.13.1
comments: true
verifid: 2017080201
layout: default
category: 网络相关
---

## 下载

- libuv: 需要下载release版 https://github.com/libuv/libuv/releases

- gyp: 原本为自动下载，但由于长城原因且懒得配置全局代理，需要手动下载后放到`libuv-1.13.1\build\gyp`目录下 https://chromium.googlesource.com/external/gyp.git/+archive/master.tar.gz

## 修改`vcbuild.bat`

删掉`125-128`行

```bat
if exist build\gyp goto have_gyp
echo git clone https://chromium.googlesource.com/external/gyp build/gyp
git clone https://chromium.googlesource.com/external/gyp build/gyp
if errorlevel 1 goto gyp_install_failed
```

只留下`goto have_gyp`这一句

## 编译

- 使用`VS2013 x86 本机工具命令提示`进入到`libuv-1.13.1`目录下

- 执行`vcbuild.bat release x86 static`编译静态库

## 编译成功后的项目设置

- `lib`文件在`libuv-1.13.1\Release\lib\`目录中

- `.h`文件在`libuv-1.13.1\include\`目录中

- 将上面两个目录拷到合适位置设置项目属性中的包含目录、库目录，并在`链接器->输入->忽略特定默认库`中添加`libcmt.lib`

- 引入相关`.lib`文件：

    ```
    #pragma comment(lib, "libuv.lib")
    #pragma comment(lib, "Iphlpapi.lib")
    #pragma comment(lib, "Psapi.lib")
    #pragma comment(lib, "Userenv.lib")
    ```
