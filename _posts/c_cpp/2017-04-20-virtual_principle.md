---
title: 探索C++虚函数在内存中的表现形式及运行机制
comments: true
verifid: 2017042001
layout: default
category: C/C++原理探索
---

> 文章概要：从简单地例子来进行探索，但是篇幅较长，建议先仔细阅读下目录结构再阅读文章，便于跳读和回顾。<br>前面一段是简单地铺垫，然后观察单层继承下的虚表指针和虚表，分析了不同情况下它们的表现形式，并手动模拟了虚函数的跳转。在中间插了一段需要注意的间接调用问题和重载、覆盖、隐藏的区别，来指出使用继承和虚函数时需要注意的地方。最后分析了在构造析构、多重继承和菱形继承的下虚表指针和虚表的表现形式。<br><br>如果仅仅需要了解虚函数的实现机制，不想看这么多的内存数据就直接看这条篇文章吧 http://blog.csdn.net/haoel/article/details/1948051/ （但如果对虚函数实现机制还比较模糊，也建议先花几分钟看下这边文章再回头来看内存吧）

---

编译环境：```win7 32bit vs2013 C语言编译器``` 平台工具集：```Visual Studio 2013 - Windows Xp (v120_xp)```

---

* TOC
{:toc}

# 引言：虚函数与多态

> 本部分是基本概念的简单表述，可以跳过 ^_^

在面向对象编程最重要的思想就是多态，而多态是通过虚函数来实现的，虚函数在继承中使用。这里我们就探究探究虚函数的编译器对虚函数的实现及其在内存中的表现形式

## 继承与虚函数

### 什么情况下需要用到继承

在创建多个类时，会出现由于数据的重复而导致接口（方法/函数）的重复，于是就产生了冗余。我们可以通过组合的方式来解决冗余问题，但又产生了很多层不必要的调用。这时就需要继承——一个类可以继承/获取另一个类的部分数据成员和方法——来解决以上问题（这里为什么说是部分数据成员和方法，这和类权限问题相关，不做讨论）

### 一个简单的继承例子

游戏中有各种各样的角色，现在我们需要创建两个角色：枪手和骑兵。我们也只需要他们基本的操作：攻击和血量

于是就需要两个类：枪手类和骑兵类。这时我们发现，他们都需要存储血量的数据成员和攻击的成员方法，这时就可以创建一个具有这两个的士兵类让他们继承

#### 代码

> 为了演示方便就没有分开头文件、声明和实现

```cpp
#include <iostream>

// 士兵类，具有血量数据成员和攻击方法
class CSoldier
{
  public:
  CSoldier()
    : m_nBlood(0x20)
  {
    /* Nothing to do */
  }
  ~CSoldier()
  {
    std::cout << "~CSolder()" << std::endl;
  }
  
  void attack()
  {
    std::cout << "Soldier Attack!" << std::endl;
  }
  
  protected:
    int m_nBlood;
};

// 枪手类，继承士兵类，并覆盖士兵类攻击方法
class CGuner
  : public CSoldier
{
  public:
  CGuner()
    : CSoldier()
  {
    /* Nothing to do */
  }

  ~CGuner()
  {
    std::cout << "~CGuner()" << std::endl;
  }
  
  void attack()
  {
    std::cout << "Guner Attack!" << std::endl;
  }
};

// 骑士类，继承士兵类，并覆盖士兵类攻击方法
class CKnight
  : public CSoldier
{
  public:
  CKnight()
    : CSoldier()
  {
    /* Nothing to do */
  }

  ~CKnight()
  {
    std::cout << "~CKnight()" << std::endl;
  }

  void attack()
  {
    std::cout << "Knight Attack!" << std::endl;
  }
};

// 测试
int main()
{
  CGuner  gunerA;
  CKnight knightA;

  gunerA.attack();
  knightA.attack();

  return 0;
}
```

输出：

![](/assets/img/virtual/0x01_common_inherit_output.png)

正常调用攻击方法和析构

#### 问题

上面的代码执行是正常的，但是如果有很多的兵种并且出现了很多个对象实类再这样一个个调用就很麻烦了，尤其是场景不同需要操作的对象实类也不同。

这时就想到把所有子类对象赋给父类指针再进行操作，现实中也是这样做的

代码：

```cpp
int main()
{
  // 创建多个不同种类的士兵
  CGuner*  pGunerA  = new CGuner;
  CGuner*  pGunerB  = new CGuner;
  CKnight* pKnightA = new CKnight;
  CKnight* pKnightB = new CKnight;

  // 将需要操作的士兵放进一个其父类指针数组里
  const int nSoldierNum = 4;
  CSoldier* const pAllSoldier[ nSoldierNum ] =
  { pGunerA, pGunerB, pKnightA, pKnightB };

  // 调用攻击方法
  for(int i = 0; i < nSoldierNum; ++i)
  {
    pAllSoldier[ i ]->attack();
  }

  // 释放空间
  for(int i = 0; i < nSoldierNum; ++i)
  {
    delete pAllSoldier[ i ];
  }

  return 0;
}
```

输出：

![](/assets/img/virtual/0x02_common_base_call_output.png)

此时结果完全不对，并没有调用子类的攻击方法，而是直接调用了父类的攻击方法，而且在释放时也只是调用了父类的析构，并没有调用子类的析构！

#### 通过virtual关键字声明虚函数解决问题

对于上面的问题通过虚函数就可以解决

具体方法，在父类的攻击方法和析构声明前面加上```virtual```关键字，子类可加可不加，但为了代码可读性一般都会加上

修改后的士兵类：

```cpp
class CSoldier
{
public:
  CSoldier()
    : m_nBlood(0x20)
  {
    /* Nothing to do */
  }

  virtual ~CSoldier()
  {
    std::cout << "~CSolder()" << std::endl;
  }

  virtual void attack()
  {
    std::cout << "Soldier Attack!" << std::endl;
  }

protected:
  int m_nBlood;
};
```

输出：

![](/assets/img/virtual/0x03_virtual_base_call_output.png)

这时调用攻击方法和析构就都正常了

## 纯虚函数与抽象类（接口类）

在上面的三个类中，我们发现士兵类根本不需要有实际的攻击方法，因为它只是一个类别，而具体的攻击方法应交给具体的士兵来实现的。这时就引入一个概念：抽象类。

抽象类：不能实例化出对象的类

实现抽象类的方法：

```cpp
class CSoldier
{
public:
  // 通过在成员方法的声明后面加上 = 0 的方式来声明这是一个抽象类
  // 这样就不能写任何的实现了，一写就报错，用该类实例对象时也报错
  virtual void attact() = 0;

  // VS2013下，抽象类可以写构造的声明和实现，但不能只写构造的声明
  CSoldier()
    : m_nBlood(0x20)
  {
    /* Nothing to do */
  }

  // VS2013下，抽象类必须要写虚析构的声明和实现
  // 只写声明会报链接错误，不写虚析构则不能正常调用析构
  virtual ~CSoldier()
  {
    std::cout << "~CSolder()" << std::endl;
  }

  // 在VC6下，如果在析构后面加上 = 0 则必须要写析构的实现
  // （VS2013也支持这个写法，但就算不写也必须写实现）
  // virtual ~CSoldier() = 0;

protected:
  int m_nBlood;
};
```

# 虚函数实现原理探索

## 继承后类的内存情况

### 普通继承

> 基于上面第一段没有加```virtual```关键字的代码

测试代码：

```cpp
// 给 CGuner 和 CKnight 加上下面两个数据成员
int m_nBlood; // 覆盖父类同名数据成员，构造时初始化为 0x30
int m_nTest;  // 构造时初始化为 0x10

int main()
{
  CSoldier soldierA;

  CGuner  gunerA;
  CKnight knightA;
  
  return 0;
}
```

内存观察：

![](/assets/img/virtual/0x04_common_inherit_memory.png)

![](/assets/img/virtual/0x05_common_soldier_memory.png)

![](/assets/img/virtual/0x06_common_guner_memory.png)

![](/assets/img/virtual/0x07_common_knight_memory.png)

我们发现，在父类内存中的数据成员虽然被子类覆盖了但还是出现在了子类内存中（如果父类有```private```数据成员也会出现在子类内存中，在此不做演示），如果有了解过组合内存情况的话就会发现继承的内存情况和组合的完全相同

总结：在普通继承后，子类中包含父类所有数据成员，并且父类数据成员出现在子类内存起始位置

### 有虚函数的继承

> 基于上面在士兵类攻击方法和析构声明前加```virtual```关键字的代码（非抽象类）

测试代码：

```cpp
// 给 CGuner 和 CKnight 加上下面两个数据成员
int m_nBlood; // 构造时初始化为 0x30
int m_nTest;  // 构造时初始化为 0x10

int main()
{
  CSoldier soldierA;

  CGuner  gunerA;
  CKnight knightA;
  
  return 0;
}
```

内存观察：

![](/assets/img/virtual/0x08_virtual_inherit_memory.png)

![](/assets/img/virtual/0x08_virtual_soldier_memory.png)

![](/assets/img/virtual/0x0C_virtual_guner_memory.png)

![](/assets/img/virtual/0x10_virtual_knight_memory.png)

上图中绿线框中的内容是与没加```virtual```关键字内存情况的区别之处，同时我们也发现了内存中新增的部分是一个指针，这就是传说中的虚表指针，而这个指针指向的内容就是传中的虚表

## 虚表指针和虚表

### 类内存中的虚表指针

![](/assets/img/virtual/0x08_virtual_inherit_memory.png)

通过监视窗口看到三个类中都出现了一个指针```__vfptr```放在对象内存的前四个字节（父类数据成员的前面），但又不像数据成员，因为它在不同的类中其值也不同

这就是虚表指针，至于它在不同类中值不同、到底是什么时候变化的？将会在后面虚表指针和虚表初始化的时机中阐述

### 虚表指针指向的虚表

先来看一波连续的内存图：

![](/assets/img/virtual/0x08_virtual_inherit_memory.png)

![](/assets/img/virtual/0x08_virtual_soldier_memory.png)
![](/assets/img/virtual/0x09_virtual_soldier_vfptr_memory.png)

![](/assets/img/virtual/0x0A_virtual_soldier_vfptr_destruct.png)

![](/assets/img/virtual/0x0B_virtual_soldier_vfptr_attact.png)

![](/assets/img/virtual/0x0C_virtual_guner_memory.png)
![](/assets/img/virtual/0x0D_virtual_guner_vfptr_memory.png)

![](/assets/img/virtual/0x10_virtual_knight_memory.png)
![](/assets/img/virtual/0x11_virtual_knight_vfptr_memory.png)

![](/assets/img/virtual/0x12_virtual_knight_vfptr_destruct.png)

![](/assets/img/virtual/0x13_virtual_knight_vfptr_attact.png)

仔细分析上面的内存和反汇编图，我们看到每个类的虚表指针分别指向一张虚表，而这几张虚表又分别存储了上面加了```virtual```关键字的虚函数的地址，虚表第一个内容指向的是对应的类的析构，第二个内容指向的是对应的类的```attact()```方法

所以我们发现虚表其实就是一个函数指针数组

### 一个简单地总结

对类继承后的虚表指针和虚表的内存进行分析发现每个类中都有一张存储自己虚函数地址的表，这样通过指针调用时，将先找到这张表，然后在通过这张表中的内容找到对应的方法，所以将避免没有虚函数时通过父类指针操作子类时调用的不是子类方法的问题了

而这一系列寻找对应的虚函数的操作是编译器在编译时生成查找代码，在运行时程序根据这些查找代码来找到对应的函数并调用的。可以称作动态绑定

### 虚表在继承后各种情况下的内存形态

#### 子类覆盖父类所有虚方法

上面对于继承后类的内存分析采用的就是子类覆盖父类所有方法的情况，子类虚表的内容和父类虚表的内容完全不同，也说明了，子类覆盖了父类所有的方法

#### 子类不覆盖父类虚方法

将前面的内存分析代码中```guner```类的析构和```attact()```方法去掉，采用完全继承父类的方法后```guner```虚表的内存形态：

![](/assets/img/virtual/0x14_virtual_guner_no_fun.png)

![](/assets/img/virtual/0x17_virtual_guner_attact.png)

![](/assets/img/virtual/0x15_virtual_soldier_destruct.png)

![](/assets/img/virtual/0x16_virtual_guner_destruct.png)

我们看到，```gunerA```和```soldierA```中的虚表指针依然不同，而虚表指针指向的虚表的析构也是不同的（因为此时编译器给```CGuner```类了一个默认析构，所以还是相当于覆盖了```CSoldier```的虚析构方法），但```gunerA```虚表中的第二项和```soldierA```虚表中的第二项完全相同、指向同一个方法```CSoldier::attact()```

这就说明了，在子类不覆盖父类虚方法时，子类的虚表中依然存放父类虚方法的地址。这也是合理的，因为子类继承了父类的虚方法，所以要能够通过子类的虚表找到继承过来的方法

#### 子类覆盖部分父类虚方法

> 上一步演示的子类不覆盖父类的方法实质上是子类覆盖父类的部分方法，因为编译器提供了一个默认析构。所以在此就不再进行过多演示。

结论：子类覆盖父类的虚方法地址会放在虚表对应的位置，而不覆盖的虚方法对应的位置存放的是父类虚方法的地址（关于虚表存放的内容的顺序将在后面探究）

#### 子类新增虚方法

在前面内存分析代码的```CKnight```骑士类中增加一个上马的方法：

```cpp
virtual void toHorsel()
  {
    std::cout << "Knight to horsel!" << std::endl;
  }
```

此时骑士类的内存情况：

![](/assets/img/virtual/0x18_virtual_knight_add_fun.png)

![](/assets/img/virtual/0x19_virtual_knight_vfptr.png)

![](/assets/img/virtual/0x1A_virtual_knight_destruct.png)

![](/assets/img/virtual/0x1B_virtual_knight_attact.png)

![](/assets/img/virtual/0x1C_virtual_knight_toHorsel.png)

我们看到，新增的```toHorsel()```方法添加到了虚表后面的位置

结论：子类新增的虚函数会出现在子类虚表的尾部，这样在用父类指针指向子类对象是依然能够通过虚表找到子类覆盖父类的虚方法地址

### 虚表内容的顺序

> 这个规律很简单，不做演示了^_^

虚表的顺序是按照虚方法在父类声明的顺序排列的，析构也不例外。如果子类新增了虚函数，在子类的虚表中出现在父类虚方法地址后面的内容是按照新增方法在子类中声明顺序排列的，即使子类新增方法的声明插在覆盖父类的虚方法声明的中间也不影响上面的排列规律

## 虚表指针和虚表初始化的时机

看了这么多在内存中的虚表指针和虚表，那么虚表指针和虚表是怎么来的呢？什么时刻初始化的？这个问题值得探讨

### 虚表指针

首先我们要知道在继承的情况下对象构造的顺序：先父类再子类

此时我们单步进入```gunerA```的创建中，于是先进入```CSoldier```的构造中：

![](/assets/img/virtual/0x1D_setp_soldier_structure.png)

- 进入之前：
    
    ![](/assets/img/virtual/0x1E_setp_soldier_structure_befor.png)

- 进入之后：

    ![](/assets/img/virtual/0x1F_setp_soldier_structure_after.png)

- 虚表：

    ![](/assets/img/virtual/0x23_soldier_vftab.png)

此时，虽然实在创建```CGuner```对象，但是在进入```CSoldier```构造时虚表指针发生了一次赋值，查看指向的虚表发现这是```CSoldier```的虚表

在单步进入```CGuner```的构造：

![](/assets/img/virtual/0x20_setp_guner_structure.png)

- 进入之前：

    ![](/assets/img/virtual/0x21_setp_guner_structure_befor.png)

- 进入之后：

    ![](/assets/img/virtual/0x22_setp_guner_structure_after.png)

- 虚表：

    ![](/assets/img/virtual/0x24_guner_vftab.png)

当进入```CGuner```的构造后虚表指针再次发生变化，这时指向的就是```CGuner```的虚表了

所以可以总接出虚表指针的变化：在创建对象时，先进入的是父类构造，虚表指针同时指向的是父类的虚表；等进入到子类构造后，虚表指针再指向子类的虚表。假如有更多层的继承，顺序也是这样的，进入哪个类的构造后虚表指针就指向哪个类的虚表；出最后一个类的构造时就是出创建的对象的构造，所以虚表指针就不再变化，正确的指向了该类的虚表

可以这样理解虚表指针的变化顺序：在进入某一层类的构造中时，该构造函数有可能使用该类的虚方法，这时虚表指针正好指向该类的虚表，于是就正确调用了该类的虚方法

### 虚表

在项目属性中设置了固定基址后我们记录当前虚表的地址和内容，结束调试，第二次调试时将断点下在进```main()```函数前/时，再去查看上次记录的位置的内存，发现还是虚表原有的内存，等到对象进入对象构造时，虚表指针就指向了这块内存（虚表）。我们可以推测出虚表内容的产生实在编译时刻，所以在程序开始时直接将虚表内容读取到内存的全局数据区，也就是说虚表的内容是早都固定好的

# 实践：手动实现简单地虚表跳转功能

> 在了解虚表原理后我们通过不使用```virtaul```关键字的方法来实现和使用虚函数同样地效果。仅仅实现上面代码的效果：通过```CSoldier```指针调用不同兵种的攻击方法

1. 定义一个函数指针类型```vtFunPtr```

2. 在每个类（子类和父类）中都放一个静态函数指针数组```m_vtArr```

3. 在父类数据成员中增加一个指向函数指针数组的指针```m_vfPtr```，子类不能载声明与父类同名的```m_vfPtr```数据成员，（因为这样虽然覆盖书父类的同名数据成员，但是父类的数据成员还是会出现在子类内存中，在使用时就会出现问题）

4. 规定好成员函数在函数指针数组中的顺序

5. 初始化每个静态指针数组```m_vtArr```

6. 在每个类（子类和父类）的构造中将自己的静态指针数组```m_vtArr```地址赋给```m_vfPtr```

7. 按照约定调用

代码：

```cpp
#include <iostream>

class CSoldier;
// 定义成员函数指针
typedef void (CSoldier::*vtFunPtr)();

class CSoldier
{
public:
  CSoldier()
    : m_nBlood(0x20)
  {
    m_vfPtr = m_vtArr;
  }

  ~CSoldier()
  {
    std::cout << "~CSolder()" << std::endl;
  }
  
  void attack()
  {
    std::cout << "Soldier Attack!" << std::endl;
  }
    
  void run()
  {
    std::cout << "Soldier Run!" << std::endl;
  }
      
  // 虚表指针
  vtFunPtr* m_vfPtr;
  // 虚表
  static vtFunPtr m_vtArr[ 2 ];
          
protected:
  int m_nBlood;
};

class CGuner
  : public CSoldier
{
public:
  CGuner()
    : CSoldier() , m_nBlood(0x30) , m_nTest(0x10)
  {
    m_vfPtr = m_vtArr;
  }

  ~CGuner()
  {
    std::cout << "~CGuner()" << std::endl;
  }
  
  void attack()
  {
    std::cout << "Guner Attack!" << std::endl;
  }
    
  void run()
  {
    std::cout << "Guner Run!" << std::endl;
  }
      
  // 虚表
  static vtFunPtr m_vtArr[ 2 ];
        
protected:
  int m_nBlood;
  int m_nTest;
};

class CKnight
  : public CSoldier
{
  public:
  CKnight()
    : CSoldier() , m_nBlood(0x30) , m_nTest(0x10)
  {
    m_vfPtr = m_vtArr;
  }

  ~CKnight()
  {
    std::cout << "~CKnight()" << std::endl;
  }
  
  void attack()
  {
    std::cout << "Knight Attack!" << std::endl;
  }
    
  void run()
  {
    std::cout << "Knight Run!" << std::endl;
  }
      
  // 虚表
  static vtFunPtr m_vtArr[ 2 ];
        
protected:
  int m_nBlood;
  int m_nTest;
};

// 初始化虚表
vtFunPtr CSoldier::m_vtArr[] = { &CSoldier::attack , &CSoldier::run };
vtFunPtr CGuner::m_vtArr[] = { (vtFunPtr)&CGuner::attack , (vtFunPtr)&CGuner::run };
vtFunPtr CKnight::m_vtArr[] = { (vtFunPtr)&CKnight::attack , (vtFunPtr)&CKnight::run };

int main()
{
  // 创建多个不同种类的士兵
  CGuner*  pGunerA  = new CGuner;
  CGuner*  pGunerB  = new CGuner;
  CKnight* pKnightA = new CKnight;
  CKnight* pKnightB = new CKnight;

  // 将需要操作的士兵放进一个其父类指针数组里
  const int nSoldierNum = 4;
  CSoldier* const pAllSoldier[ nSoldierNum ] =
  { pGunerA, pGunerB, pKnightA, pKnightB };

  // 调用攻击方法
  for(int i = 0; i < nSoldierNum; ++i)
  {
    // 实现类似有虚函数时 pAllSoldier[i]->attack(); 同样地效果
    (pAllSoldier[i]->*(pAllSoldier[i]->m_vfPtr[0]))();
    (pAllSoldier[i]->*(pAllSoldier[i]->m_vfPtr[1]))();

    // 以上代码可以拆解为下面的方式，以便于理解：
    /*
     // 获取一个士兵对象
     CSoldier* pSoldier = pAllSoldier[ i ];
     // 获取这个士兵对象的攻击方法地址
     vtFunPtr pFun = pSol->m_vfPtr[ 0 ];
     // 通过下面变态的方式调用获取到的方法
     (pSoldier->*pFun)();
     */
  }
  
  return 0;
}
```

输出：

![](/assets/img/virtual/0x25_imitate_virtual_output.png)

增加了这么多代码只模拟了最简单的虚函数调用，能够想像到编译器在背后做了多么的处理！

# 虚函数与继承需要注意的问题

> 这里只是强调一些使用的概念，可以跳过 ^_^

## 需要明确的虚函数使用问题

### 虚函数的间接调用

编译器在编译时对于调用没有加```virtual```关键字的对象方法是直接通过函数地址来调用的，也就是说普通方法的调用是直接将地址写在调用位置的，称作**直接调用** ；那在有了```virtual```关键字之后再通过**指针或引用调用** 时，编译器在编译时肯定不会直接写，因为它需要查表才能知道要调用哪个方法，所以称作**间接调用** ，需要注意，只有通过指针或引用来调用才会发生间接调用！

那么证实上面说的直接调用和间接调用呢？看反汇编调用代码：

测试代码：

```cpp
CGuner  gunerA;
gunerA.attack();

CGuner* pGuner = &gunerA;
pGuner->attack();

CGuner& rGuner = gunerA;
rGuner.run();
```

反汇编：

![](/assets/img/virtual/0x26_intdirect_call.png)

虚表：

![](/assets/img/virtual/0x27_virtual_tab.png)

![](/assets/img/virtual/0x28_guner_destruct.png)

![](/assets/img/virtual/0x29_guner_attack.png)

![](/assets/img/virtual/0x2A_guner_run.png)

- 第一条调用语句对应的反汇编```call```指令后直接写了一个地址，这个地址是```CGuner```类```attack()```方法的地址，所以这里是直接调用

- 第二条调用语句对应的汇编代码比较多，我们只注意蓝线框部分，发现它寄存器中取出一个值并偏移4字节后再调用，所以这么多汇编代码就是找到虚表指针，然后从找到虚表之后再调用对应的函数。偏移4字节是因为```attack()```方法在虚表中处于第二个位置，一个位置4字节，所以从表头偏移4字节

- 第三条调用语句和第二条类似，不同的地方是偏移了8字节，因为```run()```方法在虚表中处于第三个位置

以上就是间接调用了，总结一下：**只有对于通过指针或引用的方式调用虚函数才是间接调用**，有个特例```pGuner->CGuner::attack(); 或 pGuner->CSoldier::attack()```这样的调用是直接调用

这时有个疑问：通过普通函数调用虚函数是直接调用还是间接调用？

我们在```CSoldier```类中增加一个普通成员方法```funTest()```来调用攻击方法：

```cpp
void funTest()
{
  attack();
}
```

测试代码：

```cpp
CGuner*   pGunerA = new CGuner;
CSoldier* pSoldierA = pGunerA;

pSoldierA->funTest();
```

输出结果：

![](/assets/img/virtual/0x2C_funTest_output.png)

结果竟然是正确的，所以来看一下反汇编：

![](/assets/img/virtual/0x2D_funTest_call.png)

我们看到是间接调用的，得到结论：在普通成员函数中调用虚函数也是间接调用

### 函数的重载、覆盖和隐藏在继承的虚函数中的表现

先用张图说下```C++```编译器函数匹配的过程：

![](/assets/img/virtual/0x2B_fun_mate.png)

再上张表来说明这三者的区别：

| - | 作用域 | 函数名 | 参数 | 返回值 |
| --- | --- | --- | --- | --- |
| 重载 | 相同 | 相同 | 不同 | 不影响 |
| 覆盖 | 子类和父类 | 相同 | 相同 | 相同 |
| 隐藏 | 重叠（包括子类和父类） | 相同 | 不同 | 不影响 |

现在根据一图一表应该就能够想到函数重载、覆盖和隐藏在继承的虚函数中的表现了

简单总结一下关键点：子类有个和父类同名同参的虚函数，会覆盖；子类有个和父类同名但不同参的虚函数，会隐藏（此时无论是对于子类来说无论是直接调用还是间接调用都调不到父类同名函数）

## 数据成员在继承中的表现

结合之前对于继承后的类内存的分析再强调一下：父类的所有数据成员都会出现在子类的内存中，无论有没有被覆盖

# 在复杂情况下编译器对虚函数（虚表指针、虚表）的实现

## 构造和析构时编译器对多态性的约束

通过前面一系列的实验，我们知道，在子类构造和析构时，虚表指针指向的都是子类的虚表，所以在子类的析构和构造中调用虚函数不会出现错误调用的情况

那如果实在子类构造之前的父类的构造时调用虚函数呢？

- 通过前面的实验，我们也知道在进入父类构造是虚表指针指向的是父类的构造，所以在这里调用虚函数也是不会出错的，所以直接看一下在```CSodier```构造中调用```attack()```的反汇编：

    ![](/assets/img/virtual/0x2E_struct_virtual.png)

    ![](/assets/img/virtual/0x2F_struct_virtual_call.png)
    
    没看错，就是是直接调用！想一想，在构造函数中绝对是调用当前类自己的方法，所以根本不需要间接调用

    那么要是在父类构造函数中通过普通函数调用虚函数会不会是直接调用？还是使用```funTest()```方法来测试：

    ![](/assets/img/virtual/0x30_struct_funTest.png)

    ![](/assets/img/virtual/0x31_struct_funTest_call.png)
    
    这时就是间接调用了

- 那在父类的析构中调用虚方法呢？因为在进析构前虚表指针指向的是子类虚表，这时编译器会怎样操作才保证正确调用呢？

    就不上图了，直接说编译器的操作吧：**编译器在每进一个析构时都会将虚表指针指向当前类的虚表** 比如：析构顺序是先子类再父类，在进子类析构时会将虚表指针指向子类虚表（虽然本来指向的就是子类虚表，但还是会再为虚表指针进行一次赋值操作），出子类析构、进父类析构时就会将父类虚表指针指向父类虚表。并且在析构中调用虚方法也是直接调用

## 多重继承下的虚函数与虚表

### 实现代码

多重继承可理解为一个子类继承了多个父类

用沙发、床作为父类，用可在两者之间相互转换沙发床作为子类，以下是代码描述：

```cpp
#include <iostream>

class CSofa
{
public:
  CSofa()
    : m_nColor(0x01)
  {
    /* Nothing todo */
  }

  virtual ~CSofa()
  {
    std::cout << "~CSofa()" << std::endl;
  }
  
  virtual void site()
  {
    std::cout << "CSofa: Site" << std::endl;
  }
    
private:
  int m_nColor;
};

class CBed
{
public:
  CBed()
    : m_nColor(0x02)
  {
    /* Nothing todo */
  }

  virtual ~CBed()
  {
    std::cout << "~CBed()" << std::endl;
  }
  
  virtual void sleep()
  {
    std::cout << "CBed: Sleep" << std::endl;
  }
    
private:
  int m_nColor;
};

class CSofaBed
  : public CSofa,
    public CBed
{
public:
  CSofaBed()
    : CBed() , CSofa() , m_nColor(0x03)
  {
    /* Nothing todo */
  }

  virtual ~CSofaBed()
  {
    std::cout << "~CSofaBed()" << std::endl;
  }
    
private:
  int m_nColor;
};
```

### 子类内存表现

#### 无虚函数时

![](/assets/img/virtual/0x32_mulitiple_novirtual.png)

![](/assets/img/virtual/0x33_mulitiple_novirtual_memory.png)

我们看到两个父类的数据成员都出现在了子类的内存中，并且是按照子类继承列表的顺序排列的

#### 有虚函数时

![](/assets/img/virtual/0x34_mulitiple_virtual.png)

![](/assets/img/virtual/0x35_mulitiple_virtual_memory.png)

我们看到此时子类内存中多了两个地址数据，不需要进去查看里面是什么就能够猜到肯定是两个虚表指针。里面存的是什么呢？

![](/assets/img/virtual/0x36_mulitiple_virtual_vftab.png)

![](/assets/img/virtual/0x37_mulitiple_virtual_destruct.png)

![](/assets/img/virtual/0x38_mulitiple_virtual_site.png)

![](/assets/img/virtual/0x3B_mulitiple_virtual_vftab.png)

![](/assets/img/virtual/0x39_mulitiple_virtual_destruct.png)

![](/assets/img/virtual/0x3A_mulitiple_virtual_sleep.png)

通过内存分析发现，两张表中的第一项都存放了子类的析构（两张虚表中子类析构的地址是不同的，因为这不是真的析构，而是经过一层包装的析构，在这里将其理解为析构），第二项存放的是子类继承父类的虚方法的地址

所以可以推测出如果子类重写了对应父类的虚方法，那么对应的虚表中的内容也就变化了

### 对多重继承下虚表指针和虚表问题的探索

> 懒得上图了，直接文字描述吧 ^_^

1. 两张虚表中都存了子类的析构地址，那会不会析构两次？

    答案是肯定不会的，不管谁是编译器的作者，他都不会让这样的事情发生的

2. 在子类中新增一个两个父类都没有的虚函数，它会出现在哪张虚表的哪个位置？

    出现在内存中第一个对象的虚表指针指向的虚表的尾部，不会出现在第二个对象的虚表指针指向的虚表里

3. 子类继承的两个父类中有一个同名函数，通过不同的父类指针调用子类对象的该同名虚函数时会不会出问题？

    不会出问题，且正确调用，根据函数匹配规则，只在一个父类的函数中寻找匹配。那编译器是如何实现这一点的呢？
    
    通过变量窗口发现子类对象经过转换后的两个父类指针的竟然指向的不是同一个位置（第一个父类```CSofa```指针指向的是子类内存的首地址，第二个父类```CBed```指针指向的是内存中第二张虚表的首地址，还是看下图吧，理解更清晰一点），所以两个父类虚表指针都指向的是自己的虚表，说明在这里编译器为了获取两个父类各自的虚表指针而进行了偏移操作，而不是直接拿出内存中的第一个虚表指针来使用。

    ![](/assets/img/virtual/0x3C_mulitiple_virtual.png)
  
    但在子类没有覆盖两个父类的同名方法的前提下，通过子类直接调用或间接调用该方法编译器会报二义性错误（就算两个父类方法同名不同参也不行，根据前面的函数匹配过程和重载、覆盖、隐藏的概念来理解）

4. 接上个问题，如果子类对象是```new```出来的，然后通过两个不同的父类指针分别```delete```该对象是否会正常调用析构？（分两次尝试，而不是同时```delete```两个父类指针）

    首先，通过第一个父类对象指针```delete```是不会有问题的，第一：还回去的空间首地址还是申请时的空间首地址，第二：一定会通过虚表正常调用析构

    那如果是通过第二个父类对象指针```delete```时，还回去的空间首地址就不是申请是的空间首地址了，但是实际中还是正常调用析构和释放空间，看来是编译器在```delete```时又帮我们把指针偏移回去了 ^_^

## 在菱形继承与虚继承下的表现

### 普通实现代码

套用上面的例子：沙发和床都是家具，所以应该再创建一个家具类，让这两个类继承家具类，然后沙发床类再继承这两个类，就构成了菱形继承

代码描述：

```cpp
#include <iostream>

class CFurnitrue
{
public:
  CFurnitrue()
    : m_nColor(0x00) , m_nTest(0xFF)
  {
    /* Nothing todo */
  }

  virtual ~CFurnitrue()
  {
    std::cout << "~CFurnitrue()" << std::endl;
  }
  
  virtual int getColor()
  {
    return m_nColor;
  }
    
protected:
  int m_nColor;
private:
  int m_nTest;
};

class CSofa
  : public CFurnitrue
{
public:
  CSofa()
    : CFurnitrue() , m_nTest(0x01)
  {
    /* Nothing todo */
  }

  virtual ~CSofa()
  {
    std::cout << "~CSofa()" << std::endl;
  }
  
  virtual void site()
  {
    std::cout << "CSofa: Site" << std::endl;
  }
    
private:
  int m_nTest;
};

class CBed
  : public CFurnitrue
{
public:
  CBed()
    : CFurnitrue() , m_nTest(0x02)
  {
    /* Nothing todo */
  }

  virtual ~CBed()
  {
    std::cout << "~CBed()" << std::endl;
  }
  
  virtual void sleep()
  {
    std::cout << "CBed: Sleep" << std::endl;
  }
    
private:
  int m_nTest;
};

class CSofaBed
  : public CSofa,
    public CBed
{
public:
  CSofaBed()
    : CBed() , CSofa() , m_nTest(0x03)
  {
    /* Nothing todo */
  }

  virtual ~CSofaBed()
  {
    std::cout << "~CSofaBed()" << std::endl;
  }
  
  virtual void lie()
  {
    std::cout << "CSofaBed: Lie" << std::endl;
  }
      
private:
  int m_nTest;
};
```

继承逻辑图：

![](/assets/img/virtual/0x40_diamond_pic.png)

#### 内存结构

```cpp
CSofaBed test;
```

首先来看没有虚函数时的```CSofaBed```内存情况：

![](/assets/img/virtual/0x3E_diamond_novirtual.png)

![](/assets/img/virtual/0x3F_diamond_novirtual_memory.png)

通过内存发现，```CFurnitrue```的数据成员在```CFofaBed```的内存中出现了两次，这就有一些疑问了，```CFofaBed```在使用它可以访问的```CFurnitrue```数据成员时到底去访问哪个？这个疑问留到下一节来探索

有虚函数时```CSofaBed```内存情况：

![](/assets/img/virtual/0x3D_diamond_virtual.png)

![](/assets/img/virtual/0x3E_diamond_virtual_memory.png)

内存中多了两张虚表，毫无疑问，这和多重继承的情况一样

- 第一张虚表：

    ![](/assets/img/virtual/0x41_diamond_destruct.png)

    ![](/assets/img/virtual/0x42_diamond_getColor.png)

    ![](/assets/img/virtual/0x43_diamond_site.png)

    ![](/assets/img/virtual/0x44_diamond_lie.png)

    仔细观察就会发现，这其实就是一个三层继承的虚表

- 第二张虚表：

    ![](/assets/img/virtual/0x45_diamond_destruct.png)

    ![](/assets/img/virtual/0x46_diamond_getColor.png)

    ![](/assets/img/virtual/0x47_diamond_sleep.png)

将这两张虚表和多重继承的一对比就会发现就是多重继承的虚表中多了一个```CFurnitrue```类的虚函数

#### 问题分析

接上边的疑问，首先子类内存中有两个```m_nColor```的数据，两个虚表中还各有一个```getColor()```虚函数地址，那分别访问这两个会出现什么问题？

- 测试：

    ```cpp
    CSofaBed test;
    test.getColor();
    ```
    
    报```getColor();```二义性错误

- 测试：

    在```CSofaBed```中覆盖```getColor()```，再次调用：

    报```return m_nColor;```二义性错误

- 测试：

    在```CSofaBed```中覆盖```m_nColor```和```getColor()```，再次调用：

    正常

三次测试，前两次错误，最后一次正确，但就算最后一次正确了，结果也不是我们想要的，因为在内存中```CSofaBed```的```m_nColor```和```CFurnitrue```的```m_nColor```不是同一个（回顾前面描述的数据成员在继承中的表现）

另外，就算通过一些方法避免了上面的错误，但是内存中出现了很多冗余，这是不希望发生地事情

### 虚继承

#### 声明方法

为了解决上面遇到的问题```C++```提供了一个虚继承的方法：在继承父类的声明前加```virtual```关键字：

```cpp
class CSofa
  : virtual pubilc CFurnitrue
{};

class CBed
  : virtual pubilc CFurnitrue
{};
```

为什么要加在```CSofa```和```CBed```继承```CFurnitrue```声明前呢？

我们回顾一下前面菱形继承的内存情况，冲突是发生在这两个类继承了```CFurnitrue```时，并不是```CSofaBed```继承这两个类时

#### 内存表现

我们来看下这样声明后```CSofaBed```的内存：

- 无虚函数时：

    ![](/assets/img/virtual/0x48_virtual_diamond_novirtual.png)

    在此时的内存中发现```CFurnitrue```的数据成员出现在了整个内存的最后面，并且只有一份

    另外还发现内存中多了两个指针数据，分别查看它们指向的内存发现，指向的内存前4字节都是0（可以不用考虑这4字节的内容），后4字节是一个16进制数字，这是一个偏移值：将这两个值分别与指向偏移值指针地址相加```0x0012FF0C + 0x14 == 0x0012ff20; 0x0012FF14 + 0x0C == 0x0012FF20;```得到的都是```CFurnitrue```数据成员在```CSofaBed```内存中的首地址

    所以编译器一定是在背后通过地址的偏移来获得菱形虚继承最顶端类的数据成员地址，然后再进行相应的操作，从而避免了数据冗余。

    为什么有两个偏移值呢？因为时两个类继承了```CFurnitrue```然后```CSofaBed```再继承这两个类，回顾多重继承时，将子类指针转换为不同父类指针后发现有一个父类指针是经过偏移的地址，说明在菱形继承中将子类指针转换为中间两个父类指针时其中一个父类指针也会进行编译操作，因此这里写了两个偏移值，以保证通过中间两个父类指针操作子类对象不会出错

- 有虚函数时：

    ![](/assets/img/virtual/0x4A_virtual_diamond_virtual.png)
    ![](/assets/img/virtual/0x4B_virtual_diamond_virtual.png)
    ![](/assets/img/virtual/0x4C_virtual_diamond_virtual.png)

    这个内存结构就比较奇怪了，相比上面没有虚函数的内存结构，我们发现，除了每个指向偏移值地址的指针上面分别多了一个地址外，父类数据成员前面也多了一个指针

    首先我们看偏移地址：（偏移地址前面4字节的0变成了0xFFFFFFC，还是看不出这4字节代表什么，我们暂且认为它代表有没有虚函数）将两个偏移值分别和指向偏移值指针地址相加```0x0012FF04 + 0x18 == 0x0012FF1C; 0x0012FF10 + 0x0C == 0x0012FF1C;```得到的结果是父类数据成员前面哪个地址值的首地址（间接说明这个新出现的地址值属于```CFurnitrue```类的内容）

    这时来看指向偏移值地址的指针上面的那个指针，发现它们分别指向一块儿存函数地址的空间，所以这个指针就是虚表指针了，指向的地址的内容就是虚表了（再看一眼父类数据成员前面的指针，它也是虚表指针）。但是我们看到前两个虚表指针指向的虚表的形态除了没有虚析构的地址，其他内容都和多重继承中两个虚表的内容一样。那虚析构去哪儿了呢？
    
    再看第三张虚表，发现，虚析构在了这里！并且```CFurnitrue```的虚函数地址也在这里！说明这里代表的是```CFurnitrue```的虚表了，所以经过偏移值偏移到这里就一点都不奇怪了。另外，```CSofaBed```的析构存在这里也说明整个类在构建过程中（进入不同的构造、析构时）这里的虚表指针都会指向当时操作对应的类的虚表（这张虚表主要存储的是对应类的虚析构地址，和```CFurnitrue```的虚函数地址，不要和前两张虚表搞混了）（这里要跟的内存太多了，一个个截图有些吃不消就不放截图了，感兴趣的话自己跟一下内存 ^_^）

    如果```CSofaBed```覆盖了```CFurnitrue```的方法会出现在哪张虚表中？看内存图揭晓答案：

    ![](/assets/img/virtual/0x4D_virtual_diamond_sofabed.png)

- 只有虚析构时：

    ![](/assets/img/virtual/0x49_virtual_diamond_onlyvirtual.png)

    首先我们看到这时的内存那情况仅仅比没有一个虚函数的情况多了一个指针数据，而且偏移值上面4字节的内容也是0。这个指针数据肯定是用来存虚析构的虚表了，结合上面有虚函数的内存分析就清楚这里的确只需要一张虚表就够了

# 总结

由于篇幅已经够长了（读者不能耐心看下来就说明我的描述有问题 :<）所以就没有再探索上面提到的各种情况的各种变态组合了，但是不管怎样组合都离不开已经总结出来的规律，比如菱形继承中虚表的很多规律就和多重继承的一样。所以就写到这里了 ^_^

看了这么多内存，第一感觉就是编译器对虚函数的实现真复杂（编译器作者真伟大）！既然这么复杂，又要这么多的指针偏移、跳转、寻址的才实现想要的功能，那是不是要考虑速度问题了？

- 在LeetCode或牛客网刷过题的朋友肯定有感受，在需要大量输入输出时如果使用了```std::cout; std::cin;```就会导致程序运行超时，一部分原因是```iostream```就是一个菱形继承的例子，来看整个输出输入流的继承关系图：

    ![](/assets/img/virtual/0x4E_ios_inherit.png)

不过在开发中，程序写的不合理所带来的开销远远大于使用虚函数时的开销。但是如果可以不需要虚函数就可以实现的功能（比如说通过组合来实现上面沙发床的功能），就尽量不要使用继承了，多写些代码就能提高效率，又避免了继承的问题，也是不错的。
