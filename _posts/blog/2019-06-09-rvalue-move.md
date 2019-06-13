---
layout: post
title: 右值引用与移动构造
category: blog
description: 记录对右值引用的理解，以及在工程中的常用点
---

## 右值引用

弄清楚右值引用，首先需要理解c++中的`左值`和`右值`，`左值`与`右值`是相对于赋值操作而言。

> 能出现在赋值操作符左边的称之为`左值`
> 只能出现在赋值操作符右边的称之为`右值`

例如
```c++
int a = 1; // a为左值，1为右值
A item = GetDefaultA() // item为左值，GetDefaultA()产生临时对象为右值

```

从例子可以看出常量值、临时对象等为右值，当然从上述简单的定义是不完备的，比如
```c++
class ExceptExample {
public:
	ExceptExample(int a): m(a) {} // a,m都是左值
	ExceptExample& operator = (const ExceptExample& example) = delete;
private:
	int m;
};
ExceptExample example1; // example1为左值
```
虽然ExceptExample delete了赋值操作符，导致example1不可能出现在赋值操作左边，但它依旧是左值。

从生命周期来定义`左值`和`右值`比较直观准确
> `左值`是表达式结束以后依然可以通过名称多次访问的持久对象

> `右值`是随着表达式结束而消亡的不具名临时对象

右值是将会消亡的不具名临时对象，那为了捕获右值，c++11定义了右值引用类型。

```c++
class Example {
public:
	static Example GetExample() {
		return new Example(); // 构造Example对象 
	}
};
Example example = Example::GetExample(); // 根据临时对象复制一份返回给example
Example&& example1 = Example::GetExample(); // example1为右值引用类型的变量，example1绑定到临时对象
```
上述例子在关闭编译器优化前提下，`Example::GetExample()`产生的临时对象通过默认赋值函数创建了一个新的Example对象，临时对象析构消亡
而右值引用类型变量`example1` 绑定了原本会析构的临时对象，延长了临时对象的生命周期。


## 移动语义与移动构造


