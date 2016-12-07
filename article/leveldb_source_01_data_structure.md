# 0x01-数据结构-LevelDB源码阅读

> 欢迎转载，转载请注明出处！<br><br>
> Html版本: [leveldb\_source\_01\_data\_structure](http://kevins.pro/blog/leveldb_source_01_data_structure/)<br>
> MarkDown版本: [leveldb\_source\_01\_data\_structure.md](http://github.com/KevinsBobo/KevinsBobo.github.io/blob/master/article/leveldb_source_01_data_structure.md)<br><br>
> [Follow me on GitHub ^\_^](http://github.com/KevinsBobo/)

---

## 目录
+ [0x01\_stdint.h - 基本数据类型别名](#0x01_stdinth)
+ [0x02\_coding.h - 字节序及存储方法](#0x02_codingh)
+ [0x03\_slice.h - Slice结构](#0x03_sliceh)
+ [0x04\_status.h - 返回状态的数据结构](#0x04_statush)
+ [0x05\_arena.h - 简单的内存池](#0x05_arenah)

---
<span id="0x01_stdinth"></span>
## 0x01\_stdint.h
> `port/win/stdint.h`<br>
> 设置了基本数据类型别名

```cpp
// Define C99 equivalent types.
typedef signed char           int8_t;
typedef signed short          int16_t;
typedef signed int            int32_t;
typedef signed long long      int64_t;
typedef unsigned char         uint8_t;
typedef unsigned short        uint16_t;
typedef unsigned int          uint32_t;
typedef unsigned long long    uint64_t;
```

<span id="0x02_codingh"></span>
## 0x02\_coding.h
> `util/coding.h`<br>
> 定义了数字存储的方式-`little-endian`-小端序，和存储方法-`varint`-数值压缩存储

以下为操作32位整型函数的分析（具体实现细节见函数定义部分）：
```cpp
/* 存储函数开始 */
// 创建一个大小为value存储字节长度的字符数组，
// 然后调用EncodeFixed32，将返回的字符串数组添加至dst字符串对象中
extern void PutFixed32(std::string* dst, uint32_t value);

// 创建一个大小为5的字符数组，
// 然后调用EncodeFixed32，将返回的字符串数组添加至dst字符串对象中
extern void PutVarint32(std::string* dst, uint32_t value);

// 将自定义Slice类型数据的长度及内容存放至dst字符串对象中（Slice类型见slice.h）
// 先将数据长度value.size()通过PutVarint32函数添加至dst字符串对象中，
// 再将数据内容value.data()直接添加至dst字符串对象中
extern void PutLengthPrefixedSlice(std::string* dst, const Slice& value);
/* 存储函数结束 */

/* 获取函数开始 */
// 通过GetVarint32Ptr函数在Slice对象中获取数据原始数值，将其赋给value
// 获取到返回ture并创建新的Slice对象赋给input新对象的地址，否则返回false
// * 暂时不明白为什么要创建新对象并将地址赋给input，因为创建的新对象的
// * 第一个参数是原对象字符串数据最后一位字符的后一个指针，
// * 第二个参数最终应该为0，所以创建的对象也没有意义
// + 注释中解释是将Slice对象的指针向后移动
extern bool GetVarint32(Slice* input, uint32_t* value);

// 将input中的data()的内容和其原始数值作为data和size创建新的对象，
// 将新对象的地址赋给result，并对input进行remove_perfix()操作
extern bool GetLengthPrefixedSlice(Slice* input, Slice* result);

// 如果指针p指向的数据小于8位，直接将数据存放至指针v(value)指向的空间
// 返回该数据后面的指针
// 否则调用GetVarint32PtrFallback函数将所有数据存放至v(value)指向的空间
// 返回该数据后面的指针
inline const char* GetVarint32Ptr(const char* p,
                                  const char* limit,
                                  uint32_t* v);

// GetVarint32PtrFallback函数是完整版的GetVarint32Ptr函数，并且包含后者的功能
// 通过for循环和移位的方式依次将指针p指向的数据从低位开始存放至value中，
// 返回该数据后面的指针
extern const char* GetVarint32PtrFallback(const char* p,
                                          const char* limit,
                                          uint32_t* value);

// 获取varint数据，如果定义了kLittleEndian的值为ture，
// 则返回小端序数据，否则返回大端序数据
inline uint32_t DecodeFixed32(const char* ptr)
/* 获取函数结束 */


// 计算varint数据的字节长度
extern int VarintLength(uint64_t v);

/*varint数据存储实现开始*/
// 依次将value按字节从低位到高位存储至dst字符数组的低位到高位（下标实现）
// 如果定义了kLittleEndian的值为ture，
// 则直接按照相反的方式（大端序）进行字节拷贝操作
extern void EncodeFixed32(char* dst, uint32_t value);

// 与void同名函数操作相同，是按照指针的方式实现的，并返回操作完成后指针所在的位置
// 不过不同的是不关心kLittleEndian的值
extern char* EncodeVarint32(char* dst, uint32_t value);
/*varint数据存储实现结束*/
```

> 测试用例：[util/coding\_test.cc](https://github.com/KevinsBobo/leveldb/blob/master/util/coding_test.cc)<br>
> 更多内容细节和实现请参见[util/coding.h](https://github.com/KevinsBobo/leveldb/blob/master/util/coding.h)(中英文注释)和[util/coding.cc](https://github.com/KevinsBobo/leveldb/blob/master/util/coding.cc)

<span id="0x03_sliceh"></span>
## 0x03\_slice.h

> `include/leveldb/slice.h`<br>
> `leveldb`的一个基本数据结构

#### 1. 结构数据内容：
+ 一个指向外部存储的字节数组
+ 一个字节数组的长度值

#### 2. 优势：
+ 效率高
+ 节省空间
+ 可以包含`\0`字符

#### 3. 兼容性：
+ 能够很容易的和C++`string`类型相互转换
+ C风格字符串可以很容易的转换为`Slice`类型，但不能将`Slice`类型转换为C风格字符串。因为`Slice`类型的数据中允许包含`\0`

> 更多细节和例子参见[include/leveldb/slice.h](https://github.com/KevinsBobo/leveldb/blob/master/include/leveldb/slice.h)(中英文注释)和[中文文档](http://kevins.pro/blog/leveldb_chinese_doc/)

<span id="0x04_statush"></span>
## 0x04\_status.h

> `include/leveldb/status.h`<br>
> 返回`leveldb`操作执行后的状态

#### 1. 结构的数据内容：
+ 一个包含状态信息的字符型指针<br>成功状态(OK)时为NULL，否则指向一个包含如下信息的数组：
```cpp
state_[0..3] == 消息message长度
state_[4]    == 消息code
state_[5..]  == 消息message
```

+ 一组枚举类型，代表状态码：
```cpp
enum Code {
  kOk = 0,
  kNotFound = 1,
  kCorruption = 2,
  kNotSupported = 3,
  kInvalidArgument = 4,
  kIOError = 5
};
```

#### 2. 方法：
+ 支持赋值操作
+ 包含返回状态的方法
+ 包含判断错误状态的方法
+ 包含返回可打印的`string`类型的方法

> 更多细节和实现参见[include/leveldb/status.h](https://github.com/KevinsBobo/leveldb/blob/master/include/leveldb/status.h)(中英文注释)和[util/status.cc](https://github.com/KevinsBobo/leveldb/blob/master/util/status.cc)

<span id="0x05_arenah"></span>
## 0x05\_arena.h

> `util/arena.h`<br>
> 简单的内存池
