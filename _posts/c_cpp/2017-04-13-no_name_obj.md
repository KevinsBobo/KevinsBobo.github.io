---
title: C++无名对象与编译器对其的优化
comments: true
verifid: 2017041301
layout: default
category: C/C++原理探索
---

---

编译环境：```win7 32bit vs2013``` 平台工具集：```Visual Studio 2013 - Windows Xp (v120_xp)```

---
## 对象在函数中的使用

> 首先我们来熟悉两个概念：

### 对象作为函数参数

> 传值、传址、传引用

传值：发生拷贝构造

传址、传引用：不发生拷贝构造

### 对象作为函数返回值

传值：发生拷贝构造，返回的对象生命周期：主调函数中调用函数的那一句话的分号之前

---

## 无名对象与编译器的优化

### 无名对象

```CTest(); // CTest是一个类```、```return obj; // obj为CTest的对象，并且函数的返回值是个对象而非引用```、```return 1; // 函数的返回值为CTest对象，这里会默认执行有参构造```、```fun(CTest()) // 在调用函数的参数中直接创建一个对象```，这里几句话执行的结果都会产生一个无名对象

### 优化情况

#### 对象作为函数返回值

##### 测试代码

```cpp
#include <cstdio>
#include <cstdlib>

class CTest
{
public:
    CTest(){
        m_a = 1;
        printf("CTest()\r\n");
    }
    CTest(int i){
        m_a = 2;
        printf("CTest(int)\r\n");
    }
    CTest(CTest &obj){
        m_a = 3;
        printf("CTest(CTest&)\r\n");
    }
    ~CTest(){
        printf("~CTest()\r\n");
    }
    void operator = (CTest &obj){
        printf("operator = (Ctest&)\r\n");
    }
private:
    int m_a;
};

CTest fun1(){
    return CTest();
}

CTest fun2(){
    return 1;
}

CTest fun3(){
    CTest obj;
    return obj;
}

int main(){

    printf("fun1()\r\n");
    fun1(), printf("123\r\n");
    system("pause");

    printf("fun2()\r\n");
    fun2(), printf("123\r\n");
    system("pause");

    printf("fun3()\r\n");
    fun3(), printf("123\r\n");
    system("pause");

    printf("CTest ret1 = fun2()\r\n");
    CTest ret1 = fun2();
    printf("123\r\n");
    system("pause");

    printf("CTest ret2\r\n");
    CTest ret2;
    system("pause");

    printf("ret = fun3()\r\n");
    ret2 = fun3() , printf("123\r\n");
    system("pause");

    system("pause");
    return 0;
}
```

##### 运行结果

![](/assets/img/nonameobj/runRet.png)

##### 优化情况分析

1. 编译器在函数返回无名对象时优化了一次拷贝构造，并将该无名对象的生命周期从函数内部扩展到了函数被调用的语句分号结尾处

2. 编译器在用函数返回的对象初始化一个对象时，优化了一次拷贝构造，将返回的对象的生命周期从函数被调用的语句分号结尾处扩展到了被初始化的对象的生命周期

3. 在没有重载=运算符的情况下编译器默认提供了一个```memcpy```功能的=运算符重载函数

#### 无名对象作为函数参数

代码：

```cpp
fun(CTest());
```

在这里本该是一次构造、一次拷贝构造，但编译器优化后是直接将构造的结果放进了```fun()```函数栈中的参数内存中，并没有在主调函数的局部变量空间存放```CTest()```的结果然后再通过拷贝构造传到```fun()```函数栈中的参数内存中，所以就是一次构造了
