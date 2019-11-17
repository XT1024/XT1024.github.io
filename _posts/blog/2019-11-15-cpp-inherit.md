---
layout: post
title: c++ public protected private继承
category: blog
description: 记录c++三种继承属性和区别
---

## c++ 继承小记

### public, protected, private基础属性
**public**属性的成员在是在任何地方都可以被访问的。
- > [A public member of a class is accessible everywhere.](https://en.cppreference.com/w/cpp/language/access)

**protected**属性的成员可以被类内其他成员以及类的友元、子类的成员访问、以及子类的友元（c++17)访问

**private**属性的成员仅可以被类内其他成员以及类的友元访问

由上述可知，对成员的访问限制是**private > protected > public**

### public, protected, private继承语义
子类继承基类的大前提：**不能扩大基类成员的被访问范围** 既：基类中的protected成员，无论采用何种继承，都不可能在子类中成为public属性，在类外被访问。
**public, protected, private继承都是针对基类的public, protected成员**
记住这个大前提可以帮助理解三种继承语义对基类成员的访问权限做了什么骚操作。

#### public 继承
- 基类中public成员：子类看来是public属性（可以在任何地方被访问）
- 基类中protected成员：访问限制protected > public，基于大前提，子类看来是protected属性（可以被子类的类内成员以及子类的友元...）
- 基类中private成员： private成员仅能被类内其他成员及其友元访问，因此子类是不可访问的。


#### protected 继承
- 基类中public成员：访问限制protected > public 子类看来是protected属性（可以被子类的类内成员以及子类的友元...）
- 基类中protected成员：访问限制protected = protected，子类看来是protected属性（可以被子类的类内成员以及子类的友元...）
- 基类中private成员： private成员仅能被类内其他成员及其友元访问，因此子类是不可访问的。


#### private 继承
- 基类中public成员：访问限制private > public, 子类看来是private属性（仅可以被类内其他成员以及类的友元访问）
- 基类中protected成员：访问限制private > public, 子类看来是private属性（仅可以被类内其他成员以及类的友元访问）
- 基类中private成员： private成员仅能被类内其他成员及其友元访问，因此子类是不可访问的。

### 应用
#### public继承
标准的IS-A用法，基类的public, protected在子类中访问权限不变。
> 如果你令class D（"Derived"） 以public形式继承class B（"Base"），你便是告诉c++编译器说，每一个类型为D的对象同时意识一个类型为B的对象，反之不成立。——《effective c++》条款32:Make sure public inheritance models "is-a".

既：任何能接受B类型参数的函数，都能传入D类型的参数。反之不成立。
```c++
#include <iostream>
class Animal {
public:
	virtual void eat() = 0;
	virtual void move() = 0;
	bool isAnimal() { return true; }
protected:
	void protectedThing() {
		std::cout << "this function only for animals class member accessible." << std::endl;
	}
};

class Pig: public Animal {
public:
	void eat() override {
		std::cout << "I am Pig end eat meat." << std::endl;
	}
	
	void move() override {
		protectedThing(); // ok
		std::cout << "I am Pig and move fast." << std::endl;
	}
};

void Run(const Animal& animal) {
	animal.move();
}

void watchPigEat(const Pig& pig) {
	pig.eat();
}

int main(int argc, char** argvs) {
	Pig pig;
	pig.eat();	// ok
	pig.move();	// ok
	pig.protectedThing(); // wrong
	
	Run(pig); // ok
	watchPigEat(pig); //ok
	watchPigEat(animal) // wrong, only for pig ,not other animal like dog.
}
```

#### protected继承
protected继承中，子类将以protected的视角访问的public,protected，并且子类的子类依旧可以访问到最初基类中的public, protected成员。
在public继承中提到了is-a的概念，接受Base对象参数的函数也可以接受Dervied对象。那么protected呢? 显然是不行的。因为Base中的public在子类中已经是protected。外界不可访问。
```c++
#include <iostream>
class Animal {
public:
	virtual void eat() = 0;
	virtual void move() = 0;
	bool isAnimal() { return true; }
protected:
	void protectedThing() {
		std::cout << "this function only for animals class member accessible." << std::endl;
	}
};

class StrangeObject: protected Animal {
public:
	virtual bool isMedicinal() {
		std::cout << "this is medicinal" << std::endl;
		return true;
	}
protected:
	void eat() override {
		std::cout << "It's a strange eat method." << std::endl;
	}
	
	void move() override {
		std::cout << "It's a strange move method" << std::endl;
	}
};
```
这是一个相对合适的例子，StrangeObject不是完整意义上的Animal，因为外界只关心她是否有药用价值。但是在StrangeObject内部，乃至与StrangeObject的子类，都是需要访问eat，move等函数方法，只不过外界并不关系。


#### private继承
相对于public继承的**is-a**模型，private更多的是工程实现上的意义，也就是`implemented-in-terms-of`，抽象的理解一下，private继承的基类public, protected在子类看来都是`基类的实现被继承，接口被省略`（基类中的public,protected在子类中是private访问权限，既基类中的这些函数、成员都是细枝末节，不需要被外界或者子类的派生类感知到。
具体可以看下面的例子。
```c++
class TcpWorker {
public:
	void SendTcpMessage() {
		std::cout << "I am sending tcp message" << std::endl;
	}
};

class Service: private TcpWorker {
public:
	void Send() {
		sendTcpMessage();
	}
}
```
**TcpWorker**中的`SendTcpMessage`在子类中实际上只是Send函数内部的具体发送逻辑。也就是说Service实际上是基于TcpWorker实现而来的(`implemented-in-terms-of`)
> 实际上`implemented-in-terms-of`和 `组合`很像，上面例子也可以用组合来实现
```c++
class Service {
public:
	void Send() {
		m_tcp_worker.SendTcpMessage();
	}
private:
	TcpWorker m_tcp_worker;
}
```
> 尽量使用组合！！！！当需要重定义基类的protected方法时考虑用private继承
