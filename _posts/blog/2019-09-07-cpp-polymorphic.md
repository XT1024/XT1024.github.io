---
layout: post
title: c++多态与虚函数表
category: blog
description: 记录c++多态的实现方式与虚函数表的关联
---
## c++内存结构
首先了解下c++程序的内存结构，分为四个部分`全局数据区`，`代码区`，`栈区`，`堆区`，`全局数据区`存放的是包括全局变量，常量，**虚函数表**表等信息；`代码区`则存放的是函数信息，如下图所示

![c++内存结构](/images/cpp-polymorphic/cpp_memory_struct.png)

上述描述的是c++程序的内存结构，而对于一个c++类的对象来说，它的内存结构包含了哪些信息呢。按我们对类的理解，`类`是一些方法和数据的封装。如下面的例子
```c++
#include <iostream>
class A {
public:
	void isReady() {};
public:
	static int static_member;
private:
	int m_count;
};

int main(int argc, char** argvs) {
	A a;
	std::cout << sizeof(a) << std::endl;
	return 0;
}
```
类`A`中声明定义了`普通成员函数` `静态成员变量` `普通成员变量` 而类`A`的对象`a`实际占用的内存大小只是4字节。说明对象`a`中实际只有int m_count的空间。`成员函数` `静态成员变量`都是存储在对象外，调用时传入this指针。此时内存空间大致如下

![普通对象内存结构](/images/cpp-polymorphic/non_virtual_object_struct.jpg)

## 虚表指针
回到本文的重点，c++多态是如何实现。还是上面的例子，增加虚函数定义。
```c++
#include <iostream>
class A {
public:
	void isReady() {};
	virtual void isVirtualFunc() { std::cout << "for virtual test";} // virtual
	virtual ~A() {} // virtual
public:
	static int static_member;
private:
	int m_count;
};

int main(int argc, char** argvs) {
	A a;
	std::cout << sizeof(a) << std::endl;
	return 0;
}
```
发现对象a的内存中已经不再是`int m_count`的节大小，而是变成了16个字节。增加虚函数定义后，实际上a中增加了一个指向虚函数表的指针（8个字节），此时对象a理应存储12个字节，但由于内存对齐，实际上占用了16字节。
虚函数表存储了一系列虚函数指针，指向实际的函数地址。
内存结构如下

![虚函数对象内存结构](/images/cpp-polymorphic/virtual_object_struct.jpg)

## 总结
c++多态的实现原理，是子类根据基类虚函数表以及自身虚函数的override情况，生成自身的虚函数表。类的虚函数表在编译时就已经确定，存储在二进制文件的只读数据段。
