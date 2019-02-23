## 虚函数的工作原理

虚函数的实现要求对象携带额外的信息，这些信息用于在运行时确定该对象应该调用哪一个虚函数。典型情况下，这一信息具有一种被称为 vptr（virtual table pointer，虚函数表指针）的指针的形式。vptr 指向一个被称为 vtbl（virtual table，虚函数表）的函数指针数组，每一个包含虚函数的类都关联到 vtbl。当一个对象调用了虚函数，实际的被调用函数通过下面的步骤确定：找到对象的 vptr 指向的 vtbl，然后在 vtbl 中寻找合适的函数指针。

虚拟函数的地址翻译取决于对象的内存地址，而不取决于数据类型(编译器对函数调用的合法性检查取决于数据类型)。如果类定义了虚函数，该类及其派生类就要生成一张虚拟函数表，即vtable。而在类的对象地址空间中存储一个该虚表的入口，占4个字节，这个入口地址是在构造对象时由编译器写入的。所以，由于对象的内存空间包含了虚表入口，编译器能够由这个入口找到恰当的虚函数，这个函数的地址不再由数据类型决定了。故对于一个父类的对象指针，调用虚拟函数，如果给他赋父类对象的指针，那么他就调用父类中的函数，如果给他赋子类对象的指针，他就调用子类中的函数(取决于对象的内存地址)。

每当创建一个包含有虚函数的类或从包含有虚函数的类派生一个类时，编译器就会为这个类创建一个虚函数表（VTABLE）保存该类所有虚函数的地址，其实这个 VTABLE 的作用就是保存自己类中所有虚函数的地址，可以把 VTABLE 形象地看成一个函数指针数组，这个数组的每个元素存放的就是虚函数的地址。在每个带有虚函数的类中，编译器秘密地置入一指针，称为 vpointer（缩写为 VPTR），指向这个对象的 VTABLE。 当构造该派生类对象时，其成员 VPTR 被初始化指向该派生类的 VTABLE，所以可以认为 VTABLE 是该类的所有对象共有的，在定义该类时被初始化；而 VPTR 则是每个类对象都有独立一份的，且在该类对象被构造时被初始化。

通过基类指针做虚函数调用时（也就是做多态调用时），编译器静态地插入取得这个 VPTR，并在 VTABLE 表中查找函数地址的代码，这样就能调用正确的函数使晚捆绑发生。为每个类设置 VTABLE、初始化 VPTR、为虚函数调用插入代码，所有这些都是自动发生的，所以我们不必担心这些。

```C++
#include<iostream>

using namespace std;
 
class A
{
public:
    virtual void fun1()
	{
		cout << "A::fun1()" << endl;
	}
	virtual void fun2()
	{
		cout << "A::fun2()" << endl;
	}
};
 
class B : public A
{
public:
	void fun1()
	{
		cout << "B::fun1()" << endl;
	}
	void fun2()
	{
		cout << "B::fun2()" << endl;
	}
};
 
int main()
{
	A *pa = new B;
	pa->fun1();
	delete pa;
 
	system("pause"); 
	return 0;
}
```

毫无疑问，调用了 B::fun1()，但是 B::fun1() 不是像普通函数那样直接找到函数地址而执行的。真正的执行方式是：首先取出 pa 指针所指向的对象的 vptr 的值，这个值就是 vtbl 的地址，由于调用的函数 B::fun1() 是第一个虚函数，所以取出 vtbl 第一个表项里的值，这个值就是 B::fun1() 的地址了，最后调用这个函数。因此只要 vptr 不同，指向的 vtbl 就不同，而不同的 vtbl 里装着对应类的虚函数地址，所以这样虚函数就可以完成它的任务，多态就是这样实现的。

而对于 class A 和 class B 来说，他们的 vptr 指针存放在何处？其实这个指针就放在他们各自的实例对象里。由于 class A 和 class B 都没有数据成员，所以他们的实例对象里就只有一个 vptr 指针。

优点讲了一大堆，现在谈一下缺点，虚函数最主要的缺点是执行效率较低，看一看虚拟函数引发的多态性的实现过程，你就能体会到其中的原因，另外就是由于要携带额外的信息（VPTR），所以导致类多占的内存空间也会比较大，对象也是一样的。

含有虚函数的对象在内存中的结构如下:

```c++
class A
{
private:
	int a;
	int b;
public:
	virtual void fun0()
	{
		cout<<"A::fun0"<<endl;
	}
};
```

```text
class A 内存
VPTR    -->    &fun0()
a
b
```

#### 直接继承
那我们来看看编译器是怎么建立 VPTR 指向的这个虚函数表的，先看下面两个类：

```C++
class base
{
private:
	int a;
public:
	void bfun()
	{
	}
	virtual void vfun1()
	{
	}
	virtual void vfun2()
	{
	}
};
 
class derived : public base
{
private:
	int b;
public:
	void dfun()
	{
	}
	virtual void vfun1()
	{
	}
	virtual void vfun3()
	{
	}
};
```

两个类的VPTR指向的虚函数表（VTABLE）分别如下：

```text
base类
            ——————
VPTR——>    |&base::vfun1 |
            ——————
           |&base::vfun2 |
            ——————
      
derived类
            ———————
VPTR——>    |&derived::vfun1 |
            ———————
           |&base::vfun2    |
            ———————
           |&derived::vfun3 |
            ———————
```
每当创建一个包含有虚函数的类或从包含有虚函数的类派生一个类时，编译器就为这个类创建一个 VTABLE，如上图所示。在这个表中，编译器放置了在这个类中或在它的基类中所有已声明为 virtual 的函数的地址。如果在这个派生类中没有对在基类中声明为 virtual 的函数进行重新定义，编译器就使用基类 的这个虚函数地址。（在 derived 的 VTABLE 中，vfun2 的入口就是这种情况。）然后编译器在这个类中放置 VPTR。当使用简单继承时，对于每个对象只有一个 VPTR。VPTR 必须被初始化为指向相应的 VTABLE，这在构造函数中发生。

一旦 VPTR 被初始化为指向相应的 VTABLE，对象就"知道"它自己是什么类型。但只有当虚函数被调用时这种自我认知才有用。 

没有虚函数类对象的大小正好是数据成员的大小，包含有一个或者多个虚函数的类对象编译器向里面插入了一个 VPTR 指针(void *)，指向一个存放函数地址的表就是我们上面说的 VTABLE，这些都是编译器为我们做的我们完全可以不关心这些。所以有虚函数的类对象的大小是数据成员的大小加上一个VPTR指针(void *)的大小。

##### 总结一下VPTR 和 VTABLE 和类对象的关系:
每一个具有虚函数的类都有一个虚函数表 VTABLE，里面按在类中声明的虚函数的顺序存放着虚函数的地址，这个虚函数表 VTABLE 是这个类的所有对象所共有的，也就是说无论用户声明了多少个类对象，但是这个 VTABLE 虚函数表只有一个。

在每个具有虚函数的类的对象里面都有一个 VPTR 虚函数指针，这个指针指向 VTABLE 的首地址，每个类的对象都有这么一种指针。

#### 虚继承
     这个是比较不好理解的，对于虚继承，若派生类有自己的虚函数，则它本身需要有一个虚指针，指向自己的虚表。另外，派生类虚继承父类时，首先要通过加入一个虚指针来指向父类，因此有可能会有两个虚指针。

## 继承类的内存占用大小

首先，平时所声明的类只是一种类型定义，它本身是没有大小可言的。 因此，如果用sizeof运算符对一个类型名操作，那得到的是具有该类型实体的大小。

计算一个类对象的大小时的规律：
1. 空类、单一继承的空类、多重继承的空类所占空间大小为：1（字节，下同）；
2. 一个类中，虚函数本身、成员函数（包括静态与非静态）和静态数据成员都是不占用类对象的存储空间的；
3. 因此一个对象的大小 ≥ 所有非静态成员大小的总和； 
4. 当类中声明了虚函数（不管是1个还是多个），那么在实例化对象时，编译器会自动在对象里安插一个指针 vPtr 指向虚函数表 VTable；
5. 虚承继的情况：由于涉及到虚函数表和虚基表，会同时增加一个（多重虚继承下对应多个）vfPtr 指针指向虚函数表 vfTabl e和一个 vbPtr 指针指向虚基表 vbTable，这两者所占的空间大小为：8（或8乘以多继承时父类的个数）；
6. 在考虑以上内容所占空间的大小时，还要注意编译器下的“补齐”padding的影响，即编译器会插入多余的字节补齐；
7. 类对象的大小 = 各非静态数据成员（包括父类的非静态数据成员但都不包括所有的成员函数）的总和 + vfptr 指针(多继承下可能不止一个) + vbptr指针(多继承下可能不止一个) + 编译器额外增加的字节。

##### 示例一：含有普通继承

```c++
class A   
{   
};  
 
class B   
{
	char ch;   
	virtual void func0()  {  } 
}; 
 
class C  
{
	char ch1;
	char ch2;
	virtual void func()  {  }  
	virtual void func1()  {  } 
};
 
class D: public A, public C
{   
	int d;   
	virtual void func()  {  } 
	virtual void func1()  {  }
};   
 
class E: public B, public C
{   
	int e;   
	virtual void func0()  {  } 
	virtual void func1()  {  }
};
 
int main(void)
{
	cout<<"A="<<sizeof(A)<<endl;    //result=1
	cout<<"B="<<sizeof(B)<<endl;    //result=8    
	cout<<"C="<<sizeof(C)<<endl;    //result=8
	cout<<"D="<<sizeof(D)<<endl;    //result=12
	cout<<"E="<<sizeof(E)<<endl;    //result=20
	return 0;
}
```

前面三个 A、B、C 类的内存占用空间大小就不需要解释了，注意一下内存对齐就可以理解了。

求 sizeof(D) 的时候，需要明白，首先 VPTR 指向的虚函数表中保存的是类 D 中的两个虚函数的地址，然后存放基类 C 中的两个数据成员 ch1、ch2，注意内存对齐，然后存放数据成员 d，这样 4+4+4=12。

求 sizeof(E) 的时候，首先是类B的虚函数地址，然后类B中的数据成员，再然后是类 C 的虚函数地址，然后类 C 中的数据成员，最后是类 E 中的数据成员 e，同样注意内存对齐，这样 4+4+4+4+4=20。

##### 示例二：含有虚继承

```c++
class CommonBase
{
	int co;
};
 
class Base1: virtual public CommonBase
{
public:
	virtual void print1() {  }
	virtual void print2() {  }
private:
	int b1;
};
 
class Base2: virtual public CommonBase
{
public:
	virtual void dump1() {  }
	virtual void dump2() {  }
private:
	int b2;
};
 
class Derived: public Base1, public Base2
{
public:
	void print2() {  }
	void dump2() {  }
private:
	int d;
};
```

sizeof(Derived)=32，其在内存中分布的情况如下：

``` text
class Derived size(32):
     +---
     | +--- (base class Base1)
 | | {vfptr}
 | | {vbptr}
 | | b1
     | +---
     | +--- (base class Base2)
 | | {vfptr}
 | | {vbptr}
 | | b2
    | +---
 | d
    +---
    +--- (virtual base CommonBase)
 | co
    +---
```

##### 示例3

```c++
class A
{
public:
	virtual void aa() {  }
	virtual void aa2() {  }
private:
	char ch[3];
};
 
class B: virtual public A
{
public:
	virtual void bb() {  }
	virtual void bb2() {  }
};
 
int main(void)
{
	cout<<"A's size is "<<sizeof(A)<<endl;
	cout<<"B's size is "<<sizeof(B)<<endl;
	return 0;
}
```

执行结果：

```text
A's size is 8
B's size is 16
```

说明：对于虚继承，类 B 因为有自己的虚函数，所以它本身有一个虚指针，指向自己的虚表。另外，类 B 虚继承类 A 时，首先要通过加入一个虚指针来指向父类 A，然后还要包含父类 A 的所有内容。因此是 4+4+8=16。


## 两种多态实现机制及其优缺点

除了c++的这种多态的实现机制之外，还有另外一种实现机制，也是查表，不过是按名称查表，是 smalltalk 等语言的实现机制。这两种方法的优缺点如下：

1. 按照绝对位置查表，这种方法由于编译阶段已经做好了索引和表项（如上面的 `call *(pa->vptr[1]）` )，所以运行速度比较快；缺点是，当 A 的 virtual 成员比较多（比如1000个），而 B 重写的成员比较少（比如2个），这种时候，B 的 vtableＢ 的剩下的 998 个表项都是放 Ａ 中的 virtual 成员函数的指针，如果这个派生体系比较大的时候，就浪费了很多的空间。
2. 按照函数名称查表，这种方案可以避免如上的问题；但是由于要比较名称，有时候要遍历所有的继承结构，时间效率性能不是很高。
3. 如果继承体系的基类的 virtual 成员不多，而且在派生类要重写的部分占了其中的大多数时候，用 C++ 的虚函数机制是比较好的；
但是如果继承体系的基类的 virtual 成员很多，或者是继承体系比较庞大的时候，而且派生类中需要重写的部分比较少，那就用名称查找表，这样效率会高一些，很多的 GUI 库都是这样的，比如 MFC，QT。

##### C++如何不用虚函数实现多态 
可以考虑使用函数指针来实现多态

```c++
#include<iostream>
using namespace std;
 
typedef void (*fVoid)();
 
class A
{
public:
	static void test()
	{
		printf("hello A\n");
	}
 
	fVoid print;
 
	A()
	{
		print = A::test;
	}
};
 
class B : public A
{
public:
	static void test()
	{
		printf("hello B\n");
	}
 
	B()
	{
		print = B::test;
	}
};
 
 
int main(void)
{
	A aa;
	aa.print();
 
	B b;
	A* a = &b;
	a->print();
 
	return 0;
}
```

这样做的好处主要是绕过了 vtable。我们都知道虚函数表有时候会带来一些性能损失。

整理来自：[Reference](https://blog.csdn.net/hackbuteer1/article/details/7883531)












