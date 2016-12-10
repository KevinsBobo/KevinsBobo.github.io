# 学无止境

> 未经许可，不得转载。<br><br>
> Html版本：[learning\_log](http://kevins.pro/blog/learning_log/)<br>
> MarkDown版本：[learning\_log.md](http://github.com/KevinsBobo/KevinsBobo.github.io/blob/master/article/learning_log.md)<br><br>
> [Follow me on GitHub ^\_^](http://github.com/KevinsBobo/)

---

## 2016年12月09日
+ 阅读了《编码：隐匿在计算机软硬件背后的语言》15到19章：<br>
在实现自动操作中，条件跳转是重中之重，通过由逻辑门组成的各种计数器、选择器、累加器、加法器、锁存器等组合之后可以实现在存储器中寻址、获取数据、存储数据与条件跳转等功能，从而实现了自动操作。<br>
这些硬件的组合实现了计算机芯片的简单原型，而各种复杂的自动操作组合成的编码就是软件开发！这个程度的编码其实是在内存中存入相应的操作码和数据码，通过硬件的寻址与跳转功能对这些存储器中的数据进行操作。而对这些操作码起别名(助记符)、建立规则就形成了汇编语言！

---

## 2016年12月08日
+ 阅读了《编码：隐匿在计算机软硬件背后的语言》前14章：<br>
理解了如何使用继电器实现二进制的加减法，以及如何使用由继电器制作的边沿触发器作为计数器；似乎明白了为什么简单的晶体管能够实现功能如此强大的计算机芯片，其实这些复杂的机制背后都是由这些简单的编码组合实现的。

---

## 2016年12月07日
+ 完成了LevelDB的文档翻译工作 - [文档翻译](http://kevins.pro/blog/leveldb_chinese_doc/)
+ 注释了LevelDB的`include/leveldb/slice.h & status.h`

---

## 2016年12月06日
+ 进行了LevelDB的文档翻译工作，完成了3/4 - [文档翻译](http://kevins.pro/blog/leveldb_chinese_doc/)
+ 在翻译过程中简单了解了数据库快照的基本理论 - [Link](https://msdn.microsoft.com/zh-cn/library/ms175158.aspx)
+ 进行了LevelDB的注释工作，注释了`util/coding.h`

---

## 2016年12月05日
+ 开始阅读LevelDB源码 - [总结](http://kevins.pro/blog/leveldb_source_01_data_structure/)
+ 在阅读过程中使用OpenGrok辅助，但是发现很多时候需要使用VIM，但是在VIM插件不多的情况下有些力不从心，于是继续丰富VIM功能，把VIM打造成了一个强大的IDE - [参考](https://github.com/yangyangwithgnu/use_vim_as_ide/blob/master/README.md)

---

## 2016年12月04日
+ 学习了《算法笔记》第六章
+ 练习了《算法笔记》第六章 - vector - [Code](http://github.com/KevinsBobo/book_code/blob/master/algorithm_note/06_stl_01_vector.cpp)

---

## 2016年12月03日
+ 练习了《C++ Primer》练习册第七章 - [Code](http://github.com/KevinsBobo/book_code/blob/master/cpp_primer/07_class)

---

## 2016年12月02日
+ 继续学习C++

---

## 2016年11月26日
+ 学习了《C++ Primer》四到六章

---

## 2016年11月25日
+ 学习了《C++ Primer》第三章

---

## 2016年11月24日
+ 学习了《C++ Primer》前两章

---

## 2016年11月23日
+ 练习了《程序员代码面试指南》第一章第7节：打印窗口最大值 - [Code](http://github.com/KevinsBobo/book_code/blob/master/zuocodebook/01_StackQueue_07_getMaxWindow.c)
+ 练习了《程序员代码面试指南》第一章第6节：用栈实现汉诺塔 - [Code](http://github.com/KevinsBobo/book_code/blob/master/zuocodebook/01_StackQueue_06_hannoi_stack.c)
+ 练习了《程序员代码面试指南》第一章第6节：用递归实现汉诺塔 - [Code](http://github.com/KevinsBobo/book_code/blob/master/zuocodebook/01_StackQueue_06_hannoi_recursion.c)

---

## 2016年11月22日
+ 练习了《程序员代码面试指南》第一章第5节：用一个栈实现另一个栈排序 - [Code](http://github.com/KevinsBobo/book_code/blob/master/zuocodebook/01_StackQueue_05_sortStackByStack.c)

---

## 2016年11月21日
+ 练习了《程序员代码面试指南》第一章第1节：设计一个有GetMin功能的栈 - [Code](http://github.com/KevinsBobo/book_code/blob/master/zuocodebook/01_StackQueue_01_getMin.c)
+ 练习了《程序员代码面试指南》第一章第3节：仅用递归函数和栈操作逆序一个栈 - [Code](http://github.com/KevinsBobo/book_code/blob/master/zuocodebook/01_StackQueue_03_reverse.c)

---

## 2016年11月18日
+ 复习了《C和指针》第十、十一章

---

## 2016年11月17日
+ 练习了《C和指针》第九章——第17题 - [Code](http://github.com/KevinsBobo/book_code/blob/master/pointers_on_c/09_str_exercises_17.c)

---

## 2016年11月16日
+ 练习了《C和指针》第九章——第16题 - [Code](http://github.com/KevinsBobo/book_code/blob/master/pointers_on_c/09_str_exercises_16.c)

---

## 2016年11月15日
+ 练习了《C和指针》第九章——第15题 - [Code](http://github.com/KevinsBobo/book_code/blob/master/pointers_on_c/09_str_exercises_15.c)

---

## 2016年11月11日
+ 复习了《C和指针》第九章
+ 验证书中关于strtok函数返回指针内容的错误 - [Code](http://github.com/KevinsBobo/book_code/blob/master/pointers_on_c/09_str_function_strtok.c)

---

## 2016年11月10日
+ 练习了《C和指针》第八章——问题4 - [Code](http://github.com/KevinsBobo/book_code/blob/master/pointers_on_c/08_array_question_4.c)

---

## 2016年11月9日
+ 复习了《C和指针》第八章

---

## 2016年11月08日
+ 复习了《C和指针》第七章
+ 练习了《C和指针》第七章——第6题 - [Code](http://github.com/KevinsBobo/book_code/blob/master/pointers_on_c/07_functions_exercises_6.c)

---

## 2016年11月04日
+ 练习了《C和指针》第六章——第4题 - [Code](http://github.com/KevinsBobo/book_code/blob/master/pointers_on_c/06_pointers_exercises_4.c)
+ 复习了《C和指针》第五章——第4题

---

## 2016年11月03日
+ 复习了《C和指针》第六章——指针

---

## 2016年11月02日
+ 复习了《C和指针》第四、五章
+ 《C和指针》第五章练习题5 - [Code](http://github.com/KevinsBobo/book_code/blob/master/pointers_on_c/05_operators_exercises_5.c)

---

## 2016年11月01日
+ 练习了《C和指针》第三章练习题 - [Code](http://github.com/KevinsBobo/book_code/blob/master/pointers_on_c/03_data_question_24.c)

---

## 2016年10月31日
+ 复习了《C和指针》第三章

---

## 2016年10月26日
+ 复习了《C和指针》前两章 - [Code](http://github.com/KevinsBobo/book_code/blob/master/pointers_on_c/01_quickly_start.md)

---

## 2016年10月24日
+ 总结了《大话数据结构》中线性表的循环链表 - [Code](http://github.com/KevinsBobo/book_code/blob/master/data_structure/01_list_04_cricular.md)
+ 总结了《大话数据结构》中线性表的静态表建立方式 - [Code](http://github.com/KevinsBobo/book_code/blob/master/data_structure/01_list_03_static.md)

---

## 2016年10月23日
+ 练习了《大话数据结构》中线性表的单链表建立方式 - [Code](http://github.com/KevinsBobo/book_code/blob/master/data_structure/01_list_02_link.c)

---

## 2016年10月20日
+ 没有练习代码
+ 玩儿GitHub博客和MarkDown了
+ 有半天时间被紧急的事情占用

---

## 2016年10月19日
+ 练习了《大话数据结构》中线性表的数组建立方式 - [Code](http://github.com/KevinsBobo/book_code/blob/master/data_structure/01_list_01_array.c)
