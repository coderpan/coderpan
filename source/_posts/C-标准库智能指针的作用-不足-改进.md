title: 'C++标准库智能指针的作用&不足&改进'
date: 2015-12-14 10:55:50
tags: 智能指针
categories: C/C++
description: Introduce the smart_ptr & compare std auto_ptr
feature: smart_ptr.png
toc: true
---

本文主要讲述：
* C++标准库的智能指针的作用与不足
* 针对标准库智能指针的不足，改造智能指针

## C++标准库的智能指针的作用与不足

### 作用
在介绍之前，先抛出一个问题：为什么要用智能指针？举个例子：

<!-- more -->
``` c++
class A
{
public:
    A() : b(2) { pt = new char[100];}
    virtual ~A() { delete pt; std::cout << "destruct!" << endl; }
public:
    int b;
    char *pt;
};
int main()
{
		A* p = new A;
		p->b = 1;
		p->pt[0] = 'w';
		delete p;
		return 0;
}
```
输出结果：
``` shell
destruct!
```
这是一个再简单不过的例子了，大家都知道如果不delete p会有什么问题了，借助于java的内存回收机制（引用计数器），智能指针可以有效的解决动态申请的对内存自动释放的问题，看下面的例子：
```
int main()
{
		auto_ptr<A> p(new A);
		p->b = 1;
		p->pt[0] = 'w';
		return 0;
}
```
输出结果：
``` shell
destruct!
```
可以看到内存自动释放了，标准库智能指针的这个作用相信大部分人都是很清楚的，那么它有什么不足呢？
### 不足

auto_ptr的不足之处在于，不能存在两个智能指针指向同一个对象，比如：
``` c++
A* p = new A;
auto_ptr<A> pt1(p);
auto_ptr<A> pt2 = pt1;
```

pt2指向对象A，但pt1已经变成NULL，交出了对象的控制权，为什么会这样呢？因为auto_ptr的拷贝构造函数和=操作符重载函数使用的并非const T &，而是T&，限制始终只会有一个指针指向对象，这也是能保证内存自动释放不出错的关键之处，设想一下，如果有多个指针指向同一个对象，释放的时机不对会导致野指针的存在，后果无法预知。
正因为这样的特性，auto_ptr不支持STL容器，因为不支持无损拷贝的类型都无法作为容器的元素，这里就不举例子了，关于auto_ptr的使用也不再赘述。

## 针对标准库智能指针的不足，改造智能指针

跟我上面的关于auto_ptr的描述，我们改造智能指针的目标已经明确，那就是**支持任意多个智能指针指向同一对象，并且能在合适的时机自动释放**。要做到这一点，必须满足下面的条件：
* 智能指针类要重载常用的操作符，使用const T&，确保赋值操作不改变原智能指针对象的控制权
* 指向的对象必须包含引用计数属性，比如新增一个智能指针指向此对象，那么该对象的引用计数+1，此指针如果被其它对象的指针覆盖，此指针指向的对象的引用计数-1。

也就是说，要改造出上述要求的智能指针，必须指针类和对象类同时出动，指针的操作会修改对象的引用计数！如果做到了，那么不管对象被引用了多少次（不管多少个指针指向此对象），只要该对象的引用计数为0时，就可以释放这个对象的内存了，并且这样的智能指针还可以作为容器的元素。

### 所有对象的基类

要实现所有对象都有引用计数的属性，一个具有引用计数的句柄基类必不可少：
（_以下代码只是简单的说明思路，可能存在问题或者不完善的地方，后面会附上参考源码_）
_handle_base.h_
```c++
class CHandleBase
{
public:
		//增加计数
		void inc() { _counter++; }
		//减少计数
		void dec() { 
			_counter--; 
			if ( !_counter ){
				delete this;
			}
	    }
protected:
		CHandleBase() : _counter(0) {}
protected:
		int  _counter;
}
```
如果A类继承CHandleBase，那么A类的对象就具有引用计数的属性了:
``` c++
class A : public CHandleBase
{
public:
    A() : b(2) { pt = new char[100];}
    virtual ~A() { delete pt; std::cout << "destruct!" << endl; }
public:
    int b;
    char *pt;
};
```
上面的代码并行可能会有问题，自增和自减并非原子操作，这里只是为了说明计数的原理简化了，其实linux内核中也有atomic.h实现了自增自减的原子操作，原理其实很简单，就是在汇编命令ADD之前用LOCK命令加锁，自减同理。

_autoptr.h_
```c++
template<typename T>
class CAutoPtr
{
public:
	CAutoPtr(T* p = 0)
	{   
		_ptr = p;	
		if(_ptr)
		{
			//将对象的引用计数+1
			_ptr->inc();
		}   
	}
	//析构，减少一次引用
	~CAutoPtr()
    {
        if(_ptr)
        {
            _ptr->dec();
        }
    }
	//拷贝构造,没有改变原来的指针，实现多个智能指针指向同一对象
	CAutoPtr(const T*  p)
	{   
		_ptr = p;	
		if(_ptr)
		{
			//将对象的引用计数+1
			_ptr->inc();
		}   
	}
	//=号重载，赋值操作，很关键：被覆盖的指针指向的对象引用计数-1，传进来的指针指向的对象引用计数+1
	CAutoPtr& operator=(T* p)
   ｛
		if(_ptr != p)
		{
			if(p)
			{
			  p->inc();
			}
		  
			T* ptr = _ptr;
			_ptr = p;
		  
			if(ptr)
			{
				ptr->dec();
			}
		}
		return *this;
	}
	/*
	 *省略其它的操作符重载，凡是影响对象引用次数的操作都需要监控。
	*/
public:
    T*          _ptr;
}
```
这样使用CAutoPtr&lt;A&gt; p(new A);就可以创建A类型的智能指针了，看下面的使用实例：
```
int main()
{
		CAutoPtr<A> p1(new A);
		p1->b = 1;
		p1->pt[0] = 'w';
		
		CAutoPtr<A> p2(p1);  //此时，A类的对象的引用计数为2		
		CAutoPtr<B> p3(new B); //此时B类对象的引用计数为1
		p2 = p3;   //p2被覆盖，A类对象的引用计数减一，B类对象的引用计数加一
		p1 = p3;  //同上，此时A类对象的引用计数为0，会调用A类的析构函数释放对象。		
		return 0;  //return之后，B类对象也会自动释放，因为三个智能指针的生命期到了，会调用智能指针类的析构函数。
}
```
由于拷贝构造函数和赋值函数都被重写（有原来的引用变为常引用），拷贝之后不改变被拷贝的对象，满足了STL容器的条件，所以，改造后的智能指针可以用在容器了，就不再一一举例，感兴趣的同学可以自己测试
