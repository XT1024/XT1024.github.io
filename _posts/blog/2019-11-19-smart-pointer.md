---
layout: post
title: c++ 智能指针
category: blog
description: 循环引用与c++11智能指针的选用
---

## c++11 循环引用
### 循环引用导致内存泄露
智能指针是普通指针的封装，具有RAII的特性，当无人使用其指针时自动释放被指向的对象空间
看如下场景
```c++
#include <memory>
class A {
public:
	void Append(std::shared_ptr<B> b) {
		m_b = b;
	}
private:
	std::shared_ptr<B> m_b;
};

class B {
public:
	void SetMember(std::shared_ptr<A> a) {
		m_a = a;
	}
private:
	std::shared_ptr<A> m_a;
};

int main(int argc, char** argvs) {
std::shared_ptr<A> a(new A());
std::shared_ptr<B> b(new B());

a->Append(b);
b->SetMember(a);
return 0;
}
```
有趣的现象来了，我们知道shared_ptr引用计数为0时才会释放对象。上述代码在析构时具体的表现如下
> a退出作用域，a指向的对象A等待b指向的对象释放m_a，但是m_a释放的前提是b指向的对象析构。
> b退出作用域，b指向的对象B等待a指向的对象释放m_b,  但是m_b释放的前提是a指向的对象析构。
> 因此，a, b指向的对象都在相互等待对方的释放，而此时由于指针a,b都已退出作用域，上述直接导致了内存泄露

### 实际工程场景
emmm..上述原理我们可能都明白，但是在实际工程编码中，也比较容易忽视这一点。举个例子。
**由于工程要求，需要实现一颗二叉树，并且子节点访问父节点的时间复杂度为O(1)**

>so easy?
```c++
#include <memory>
class Node {
public:
	Node(int val): m_val(val) {}
	void SetLeft(std::shared_ptr<Node> left) {
		m_left = left;
	}
	void SetRight(std::shared_ptr<Node> right) {
		m_right = right;
	}
	void SetParent(std::shared_ptr<Node> parent) {
		m_parent = parent;
	}
private:
	int m_val;
	std::shared_ptr<Node> m_left;
	std::shared_ptr<Node> m_right;
	std::shared_ptr<Node> m_parent; // 说好的明白原理呢，这里有循环引用啊喂！
};

int main(int argc, char** argvs) {
	std::shared_ptr<Node> root(new Node(1));
	std::shared_ptr<Node> a(new Node(2));
	root->SetLeft(a);	// !!! 循环引用
	a->SetParent(root); // !!! 循环应用
}
```
好吧，那正确的操作应该是什么呢？ 请看下节精彩

## weak_ptr
c++11提供了std::weak_ptr来解决shared_ptr循环引用导致的内存泄露问题
![循环引用](/images/smart-pointer/reference_circle_one.png)
![外界引用销毁](/images/smart-pointer/reference_circle_one.png)

由于相互使用std::shared_ptr，产生了一个引用循环，一旦这个循环的外界引用正常销毁，这个环内的对象引用计数无法等于0，由此导致了内存泄露。

循环引用问题的根源是互相持有std::shared_ptr,而std::weak_ptr则并不增加std::shared_ptr的引用计数std::weak_ptr实际上是std::shared_ptr的一个观察者。

- std::weak_ptr可以获取被观察的std::shared_ptr引用计数，实际对象是否销毁。
- std::weak_ptr的lock方法可以基于被观察的std::shared_ptr创建一个新的std::shared_ptr(不抛异常，std::shared_ptr(const std::weak_ptr& r) r为空时抛异常)
- 创建的新的std::shared_ptr只临时使用，而不保存（否则同样会出现循环引用）

改正后的代码如下。
```c++
#include <memory>
#include <iostream>
class Node {
public:
	Node(int val): m_val(val) {}
	void SetLeft(std::shared_ptr<Node> left) {
		m_left = left;
	}
	void SetRight(std::shared_ptr<Node> right) {
		m_right = right;
	}
	void SetParent(std::shared_ptr<Node> parent) {
		m_parent = parent;
	}
	int GetVal() {
		return m_val;
	}
	int GetParentVal() {
		std::cout << "parent reference count" << m_parent.use_count() << std::endl;
		if(std::shared_ptr<Node> tmp = m_parent.lock()) {
			std::cout << tmp->GetVal() << std::endl;
			return tmp->GetVal();
			// 此时，如果tmp是最后一个std::shared_ptr引用,则tmp退出作用域后对象析构
		} else {
			std::cout << "weak_ptr观察的shared_ptr管理的对象已经释放" << std::endl;
			return -1;
		}
	}
private:
	int m_val;
	std::shared_ptr<Node> m_left;
	std::shared_ptr<Node> m_right;
	std::weak_ptr<Node> m_parent;
};

int main(int argc, char** argvs) {
	std::shared_ptr<Node> root(new Node(1));
	std::shared_ptr<Node> a(new Node(2));
	root->SetLeft(a);	
	a->SetParent(root); 
}
```


