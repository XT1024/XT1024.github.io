---
layout: post
title: 右值引用与移动构造
category: blog
description: 记录对右值引用的理解，以及在工程中的常用点
---

## 右值引用

### 左值与右值

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

### 右值引用

右值是将会消亡的不具名临时对象，那为了捕获右值，继续访问右值对象，c++11定义了右值引用类型。

```c++
class Example {
public:
	static Example GetExample() {
		Example a;
		return a; // 构造Example对象 
	}
};
Example example = Example::GetExample(); // 根据a（左值）复制产生一份临时对象Example b（右值）, a析构；随后临时对象b复制构造example（左值），之后临时对象b析构
Example&& example1 = Example::GetExample(); // 根据a（左值）赋值产生一份临时对象Example b（右值）， a析构；随后临时对象b 被右值引用example1（左值）绑定
```
上述例子在关闭编译器优化前提下，`Example::GetExample()`产生的临时对象通过默认复制构造函数创建了一个新的Example对象，临时对象析构消亡
而右值引用类型变量`example1` 绑定了原本会析构的临时对象，延长了临时对象的生命周期。

> tips: 右值引用绑定了右值，但是上述example1是`左值`，因为example1是可以多次访问的具名对象。

## 移动语义与移动构造
在具体应用中，右值引用的应用场景通常是体现在移动构造中，下面我们一步步的介绍下移动构造
首先给出一个简单的类定义

```c++
class Province {
public:
	Province(const char* city, int area): m_area(area) {
		m_city = new char[strlen(city) + 1];
	}

	Province(const Province& province) {
		// province持有堆上的资源，需要进行深度复制
		m_city = new char[strlen(province.m_city) + 1];
		strcpy(m_city, province.m_city);
		m_area = province.m_area;
	}

	Province& operator=(const Province& province) {
		if(this != &province) {
			m_city = new char[strlen(province.m_city) + 1];
			strcpy(m_city, province.m_city);
			m_area = province.m_area;
		}
		return *this;

	}

	～Province() {
		delete m_city;
	}

private:
	char* m_city;
	int m_area;

};

Province GetProvince() {
	Province a("guangdong", 23423);
	return a;
}

int main(int argc, char** argvs) {
	Province home = GetProvince(); // 1: 调用默认复制构造函数
	Province before("guangzhou", 231); // 2: 调用Province(const char*, int)构造函数
	Province after(before); // 3: 调用复制构造函数
	return 0;
}
```
Province类中存在指针，因此必须自定义复制构造和赋值函数，否则默认的复制构造函数将会两个Province对象持有同一份m_city, 最终在析构时出现`double free`。

而在之前的构造函数、复制构造函数、赋值成员函数基础上，c++11提出了`移动构造函数、移动赋值函数`，移动构造函数和移动赋值函数通过`右值引用`类型接受`右值`参数，例如

```c++
class Province {
...
	/**
	* 移动构造函数
	* Province&& 右值引用参数，接受右值传入
	*/
	Province(Province&& province) {
		m_city = province.m_city; // 直接剥夺右值province的char*, 减少资源开销。
		m_area = province.m_area;
		province.m_city = nullptr;
	}
...
}；
Province GetProvince() {
	Province a("guangdong", 23423);
	return a;
}
int main(int argc, char** argvs) {
	Province home = GetProvince(); // 1: a为左值，调用Province复制构造函数产生临时对象b,由于b为右值，则调用移动构造函数生成home对象
	Province before("guangzhou", 231); // 2: 调用Province(const char*, int)构造函数
	Province after(before); // 3: before为左值,则调用复制构造函数生成after
	return 0;
}
```

> 移动构造函数实际上是将资源从原本要消亡的右值中剥夺出来，移动到要构造的对象中，减少复制的成本。

因此，如果类`定义了移动构造函数`

- 上述1中`GetProvince`是`右值`将会调用`移动构造函数`，将右值持有的资源直接`移动`到新构建的对象。
- 上述3中`before`是`左值`将会调用`复制构造函数`, 完整的`复制`before生成新的对象

若`未定义移动构造函数`

- 上述1，3都将会调用`复制构造函数`，将根据传入的对象重新`复制`一份到新的对象里。

以上都在说明右值引用延续了右值生命周期，从在在移动构造函数中能将右值中持有的资源移动到新构造的对象中,
那右值引用是否可以绑定原本是左值的变量呢，c++11提供了`std::move`把左值强转成右值。

```c++
Province home = GetProvince(); // home为左值
Province a(home)				// 调用正常的复制构造函数
Province b(std::move(home))		// 因为std::move将home强转成左值，且Province自身定义了移动构造函数，则调用移动构造函数
// 特别注意：在home使用std::move转右值后，不应该再使用home，原因是home持有的资源已经部分被移动到b中，已经不是一个完整的Province对象
```

## 常见使用错误

*错误一*

```c++
class Error {
public:
	Error(const Error& error) {
		a = error.a;
		b = error.b;
		c = error.c;

	}
	Error(Error&& error) {
		a = error.a;
		b = error.b;
		c = error.c;
	}
private:
	int a;
	double b;
	long int c;
};
```
类本身不管理可转移的资源，上述定义移动构造函数没有意义，因为成员变量都是只复制，无法移动的类型。

*错误二*

```c++
class Error {
public:
	...
	
	Error(const Error& error) {
		m_message = new char[strlen(error.m_message) + 1];
		m_length = error.m_length;
	}
private:
	char* m_message;
	int m_length;
}

int main(int argc, char** argvs) {
	Error err1;
	Error err2(std::move(err1));
	return 0;
}
```
类自身没有定义移动构造，移动赋值函数，就算std::move强转err1为右值以后，因为不存在移动构造函数，也只会调用复制构造函数，并没有任何资源上的转移和重复利用。

*错误三*

```c++
class Error {
public:
	Error(const char* message, length) :m_length(length){
		m_message = new char[strlen(message) + 1];
	}

	Error(const Error& error) {
		m_message = new char[strlen(error.m_message) + 1];
		m_length = error.m_length;
	}

	Error(Error&& error) {
		m_message = error.m_message;
		m_length = error.m_length;
		error.m_message = nullptr;
	}
private:
	char* m_message;
	int m_length;
};

Error&& Analyze() {
	Error error("emmmm...", 32);
	...
	...
	return std::move(error);
}
int main(int argc, char** argvs) {
	Error err2(Analyze());
	return 0;
}
```
Analyze()返回的是右值引用类型，而定义在Analyze的error虽然转为右值，但由于返回时error作为函数内的变量已经析构了，所以函数返回内部栈上的右值引用是错误的。
可以等同与返回一个正常的左值引用类型，毕竟引用一个在内部域析构的对象是错误的。

## 总结

1. 右值引用类型是为了捕获c++中的右值，延续其生命周期。
2. 右值引用类型变量是左值。
3. 类中定义了移动构造函数，在遇到用右值构造新对象时，才会调用移动构造函数，否则会调用复制构造函数。
4. 右值在被用于移动构造函数后，不应该再继续使用。
4. std::move将返回转成右值，无论传入参数是左值还是右值。
5. 最好不要返回右值引用类型，除非你知道自己在做什么。







