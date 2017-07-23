---
layout: default
comments: true
verifid: 2017072301
title: 为tshrak增加保存图片功能
category: wireshark
---

<iframe src="//ghbtns.com/github-btn.html?user=KevinsBobo&repo=wireshark-modify&type=watch&count=true" allowtransparency="true" frameborder="0" scrolling="0" width="110" height="20"></iframe>

github: https://github.com/KevinsBobo/wireshark-modify

> tshark 是 wireshark 的命令行版，所以修改起来更加容易 ^_^

* TOC
{:toc}

## tshark 捕获、解析逻辑

```
tshark.c
main()
    解析命令行参数
    初始化环境
    -->
    caption()
        初始化信息
        while(...)
            --> capchild\capture_sync.c -> syn_pipe_input_cp(...) // 回调的方式
                从管道(文件)中读取数据头部信息
                如果读到的是一个数据包的头部则开始处理
                --> capture_input_new_packets(capture_session ...)
                    根据传入的capture_session获得一个 capture_file 类型的数据指针
                    如需解码：
                        获得一个 epan_dissect_t 类型的空间
                        并根据 capture_file 结构体初始化数据
                        // edt 中将包含一个具有链表结构的数据包
                        --> process_packet(capture_file *cf, epan_dissect_t *edt ...)
                            解析数据
                            --> epan_dissect_run_with_taps(...)
                                获得 edt 中的 tvbuff_t 类型的数据指针 // 具有链表结构的数据包
                            --> print_packet(...)
                                打印数据包
                                --> print_columns()
                                    以列的形式打印数据
                                    通过 cf 获得数据包的列结构数据
                                    // 编号、时间 IP TCP/UDP 端口 类型 数据类型
                                    // 我是在这里通过字符串比较判断是否为 HTTP 图片 jpg/png/gif
                                    如果通过参数设置需要打印数据包16进制数据包
                                        则继续打印16进制数据
                                        这里打印的16进制数据是数据包的原格式
                                        后面我借鉴了这里面对 tvbuff_t 类型数据的分解的代码
                                        从而得到图片数据
``` 

## 调试、分析过程

### 抓包时调试、分析

> `tshark src tcp 80`

0. 单步找到 `caption() capture_input_new_packets() print_packet() print_columns()`等函数

1. 通过vs2013下`fprintf()`API断点并通过栈回溯分析函数调用情况(因为tshark是通过`fprintf`输出数据的)

2. 通过API断电发现了`print_columns()`的输出`HTTP`状态的位置

### 分析数据包时调试

> `tshark -r out.pcap -Y http -x`

3. 在`printf_columns()`中发现了输出16进制数据的`print.c -> print_hex_data()`函数

4. 通过条件断点发现这里输出的数据是抓取到的整个数据包

5. 于是找到了`print_hex_data()`中使用`tvbuff_t`的代码


## 保存图片新增代码

> 注：在tshark原代码中新增的代码都以`/* 保存图片新增 start */ ... /* 保存图片新增 stop */`的形式标注<br>调试新增代码都以`/* 调试新增 start */ ... /* 调试新增 stop */`的形式标注<br><br>为方便管理，新增代码的实现和声明放在了单独的`kevins-....c`和`kevins-....h`文件中，位于wireshark源码目录的`kevins`目录中<br>所有函数和全局变量均以`kevins-...`开头

### kevins-file.h

> 这里的函数是文件相关的，一个实创建目录，一个是初始化文件，没什么说的

```cpp
#define KEVINS_MAXPATHLEN 255

void kevins_init_floder(char* szFloder);

int kevins_init_file(char* szFile);
```

### kevins-save-pic.h

> 这是保存图片的主要代码

```cpp
/* 照抄 tsark.h 包含的头文件 start */
... // 省略
/* 照抄 tsark.h 包含的头文件 stop */

/* 新增的头文件 start */
#include <epan/tvbuff-int.h>
#include <kevins/kevins-file.h>
/* 新增的头文件 stop */

// 图片根目录
#define KEVINS_PIC_FLODER_NAME "d:\\kevins_save_pic\\"
// 图片名前缀
#define KEVINS_PIC_FILE_NAME "save_pic"

// 全局变量 标记数据包是否为图片
extern int kevins_g_is_pic;
// 全局变量 标记数据包来源IP
extern char kevins_g_src_ip[ KEVINS_MAXPATHLEN ];

// 全局变量 标记数据包来源IP
#define KEVINS_PIC_JPG 1
#define KEVINS_PIC_PNG 2
#define KEVINS_PIC_GIF 3

// 主要功能函数
void kevins_save_pic(epan_dissect_t * edt);
```

### kevins-save-pic.c

> 实现代码

```cpp
#include <kevins/kevins-save-pic.h>
int kevins_g_is_pic = 0;
char kevins_g_src_ip[ KEVINS_MAXPATHLEN ];
void kevins_save_pic(epan_dissect_t * edt)
{
  if(edt == NULL)
  {
    return ;
  }
  char szPicPath[ MAXPATHLEN ] = { 0 };
  tvbuff_t * tvb = NULL;
  u_char* pData = NULL;
  unsigned long long *pVerify = NULL;
  FILE* fpPic = NULL;
  static int g_kevins_save_pic_time = 0;
  const guchar *cp = NULL;
  guint         length = 0;
  // gboolean      multiple_sources;
  GSList       *src_le = NULL;
  struct data_source *src = NULL;
  
  // 将 tvb 指针移动到合适的位置
  for( tvb = edt->tvb ; tvb != NULL; tvb = tvb->next)
  {
    // 可以确定 jgp/gif 图片肯定在最后一个数据包
    // PNG 图片在倒数第六个数据包
    // 所以直接将 jpg/gif 的指针移到最后一个数据包的位置
    if(((kevins_g_is_pic == KEVINS_PIC_JPG || kevins_g_is_pic == KEVINS_PIC_GIF)
         && tvb->next != NULL))
    {
      continue;
    }

    if(tvb->real_data == NULL)
    {
      return ;
    }
   
    // jpg 数据首地址在最后一个数据包地址的前两个字节的位置，png 和 gif 则正常
    pData = (unsigned char*)(tvb->real_data) - 2;
    pVerify = (unsigned long long*)tvb->real_data;
    // 再次判断，匹配则跳出循环，按照逻辑只有png才会多次判断
    if((*(unsigned short*)pData == 0xD8FF && *pVerify == 0x4649464A1000E0FF)                 // jpg
       || (*(unsigned long long*)(pData + 2) == 0x0A1A0A0D474E5089)                          // png
       || (((*(unsigned long long*)(pData + 2)) & 0x0000FFFFFFFFFFFF) == 0x0000613938464947) // gif
       )
    {
      break;
    }
  }
  /* 参考自 print.c -> print_hex_data 函数 */
  // 获取数据长度 和 http 数据包首部指针
  for(src_le = edt->pi.data_src; src_le != NULL;
      src_le = src_le->next)
  {
    if(src_le->next != NULL)
    {
      continue;
    }
    src = (struct data_source *)src_le->data;
    tvb = get_data_source_tvb(src);
    length = tvb_captured_length(tvb);
    if(length == 0)
      return ;
    // 获取http数据首部指针
    cp = tvb_get_ptr(tvb , 0 , length);
    
    if(cp == NULL)
    {
      return ;
    }
    break;
  }
  if(kevins_g_is_pic == KEVINS_PIC_JPG)
  {
    sprintf_s(szPicPath , MAXPATHLEN , "%s%s\\jpg\\" ,
              KEVINS_PIC_FLODER_NAME , kevins_g_src_ip);
    // 创建文件夹
    kevins_init_floder(szPicPath);
    sprintf_s(szPicPath , MAXPATHLEN , "%s%s\\jpg\\%s%d.jpg" ,
              KEVINS_PIC_FLODER_NAME , kevins_g_src_ip, KEVINS_PIC_FILE_NAME, g_kevins_save_pic_time++);
  }
  else if(kevins_g_is_pic == KEVINS_PIC_PNG)
  {
    // 偏移指针
    pData += 2;
    sprintf_s(szPicPath , MAXPATHLEN , "%s%s\\png\\" ,
              KEVINS_PIC_FLODER_NAME , kevins_g_src_ip);
    // 创建文件夹
    kevins_init_floder(szPicPath);
    sprintf_s(szPicPath , MAXPATHLEN , "%s%s\\png\\%s%d.png" ,
              KEVINS_PIC_FLODER_NAME , kevins_g_src_ip, KEVINS_PIC_FILE_NAME, g_kevins_save_pic_time++);
  }
  else if(kevins_g_is_pic == KEVINS_PIC_GIF)
  {
    // 偏移指针
    pData += 2;
    sprintf_s(szPicPath , MAXPATHLEN , "%s%s\\gif\\" ,
              KEVINS_PIC_FLODER_NAME , kevins_g_src_ip);
    // 创建文件夹
    kevins_init_floder(szPicPath);
    sprintf_s(szPicPath , MAXPATHLEN , "%s%s\\gif\\%s%d.gif" ,
              KEVINS_PIC_FLODER_NAME , kevins_g_src_ip, KEVINS_PIC_FILE_NAME, g_kevins_save_pic_time++);
  }
  // 创建文件
  if(kevins_init_file(szPicPath))
  {
    return ;
  }
  // 打开文件
  fopen_s(&fpPic , szPicPath , "wb");
  if(fpPic == NULL)
  {
    return ;
  }
  // 获取文件长度
  u_int pic_length = length - (pData - cp);
  
  // 写文件
  fwrite(pData , pic_length , 1 , fpPic);
  
  // 关闭文件
  fclose(fpPic);
}
```

### tshrk.c新增代码

#### 头文件

> 将`kevins-....c`文件加入到tshark项目工程中并包含新增代码的头文件

```cpp
/* 保存图片新增 start */
#include <kevins/kevins-save-pic.h>
#include <kevins/kevins-file.h>
/* 保存图片新增 stop */
```

#### print_columns()函数中

> 这里主要判别是否为图片，并设置标志位

```cpp
static gboolean
print_columns(capture_file *cf)
{
  ...
  switch (col_item->col_fmt) {
    ...
    case COL_UNRES_NET_SRC:
      column_len = col_len = strlen(col_item->col_data);
      if (column_len < 12)
        column_len = 12;
      line_bufp = get_line_buf(buf_offset + column_len);
      put_spaces_string(line_bufp + buf_offset, col_item->col_data, col_len, column_len);
      /* 保存图片新增 start */
      strcpy_s(kevins_g_src_ip , KEVINS_MAXPATHLEN , col_item->col_data);
      /* 保存图片新增 stop */
      break;
    ...
    default:
      column_len = strlen(col_item->col_data);
      line_bufp = get_line_buf(buf_offset + column_len);
      put_string(line_bufp + buf_offset, col_item->col_data, column_len);
      /* 保存图片新增 start */
      if(!strcmp(col_item->col_data , "HTTP/1.1 200 OK  (JPEG JFIF image)"))
      {
        kevins_g_is_pic = KEVINS_PIC_JPG;
      }
      else if(!strcmp(col_item->col_data , "HTTP/1.1 200 OK  (PNG)"))
      {
        kevins_g_is_pic = KEVINS_PIC_PNG;
      }
      else if(!strcmp(col_item->col_data , "HTTP/1.1 200 OK  (GIF89a)")
              || !strcmp(col_item->col_data , "HTTP/1.1 200 OK  (GIF89a) (image/jpeg)"))
      {
        kevins_g_is_pic = KEVINS_PIC_GIF;
      }
      else
      {
        // 保险措施
        kevins_g_is_pic = 0;
      }
      /* 保存图片新增 stop */
```

#### process_packet()函数中

> 是在获取`edt`中`tvb`指针数据后调用保存图片的函数

```cpp
static gboolean
process_packet(capture_file *cf, epan_dissect_t *edt, gint64 offset, struct wtap_pkthdr *whdr,
               const guchar *pd, guint tap_flags)
{
  ...
  if (passed) {
    frame_data_set_after_dissect(&fdata, &cum_bytes);
 
 
    /* Process this packet. */
    if (print_packet_info) {
      /* We're printing packet information; print the information for
         this packet. */
      print_packet(cf, edt);
 
      /* 保存图片新增 start */
      if(kevins_g_is_pic)
      {
        kevins_save_pic(edt);
        kevins_g_is_pic = 0;
      }
      /* 保存图片新增 stop */
  ...

  /* 保存图片新增 start */
  // 保险措施，标志位置 0
  kevins_g_is_pic = 0;
  /* 保存图片新增 start */
 
  return passed;
}
```

## 效果展示

### 从已经捕获的数据包中抓取图片

> `tshark -r out.pcap`

![](/assets/img/wireshark/tshark_add_save_pic_1.png)
 
### 在捕获数据包同时保存图片

![](/assets/img/wireshark/tshark_add_save_pic_2.png)
 
### png 图片

![](/assets/img/wireshark/tshark_add_save_pic_3.png)
 
### gif 图片

![](/assets/img/wireshark/tshark_add_save_pic_4.png)