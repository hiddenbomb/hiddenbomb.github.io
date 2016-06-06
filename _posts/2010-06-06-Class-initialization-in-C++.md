#C++中类的初始化、构造与析构的顺序

今天调试代码发现一个现象，类的成员变量的定义顺序和初始化列表顺序原来是不能随便定义的。

请看如下代码：

	#include <iostream>
	#include <string>
	using namespace std;

	class A
	{
	public:
		A() : n(5), str("hello")
		{
			cout << n << " " << str << endl;
		}
	private:
		string str;
		int n;
	};

	int main(int argc, char* argv[])
	{
		A a;
		return 0;
	}

编译选项为`-Wall -Werror`，所用编译器及系统信息如下：
`g++ (GCC) 4.4.6 20120305 (Red Hat 4.4.6-4)`

提示如下错误：

	[guibw@tradetest1 init]$ g++ -Wall -Werror -o init init.cpp 
	cc1plus: warnings being treated as errors
	init.cpp: In constructor ‘A::A()’:
	init.cpp:14: 错误：‘A::n’将随后被初始化
	init.cpp:13: 错误：‘std::string A::str’
	init.cpp:8: 错误：  当在这里初始化时

在警告当作错误的严格级别下，**类成员变量的声明顺序必须与初始化列表中的顺序一致**，否则会报错。

由此发散联想，类的构造函数和析构函数执行顺序是怎样的？再加入多重继承，虚拟继承的情况呢？或者多个类组合在一起时呢？

通过实例来说明问题。

##1. 类组合时初始化列表的顺序

	#include <iostream>
	using namespace std;

	class A
	{
	public:
		A() { cout << "build A" << endl; }
		~A() { cout << "destroy A" << endl; }
	};

	class B
	{
	public:
		B() { cout << "build B" << endl; }
		~B() { cout << "destroy B" << endl; }
	};

	class C
	{
	public:
		C() : a(A()), b(B()) { cout << "build C" << endl; }
		~C() { cout << "destroy C" << endl; }
	private:
		A a;
		B b;
	};

	int main(int argc, char* argv[])
	{
		C c;
		return 0;
	}
Class C有两个成员分别是a和b，在初始化列表中初始化a和b，得到的输出结果如下：

	build A
	build B
	build C
	destroy C
	destroy B
	destroy A
如果将Class C中声明a和b的顺序交换一下，在-Wall -Werror的编译选项下编译不通过（见前文），但在普通编译选项下，编译通过得到的结果是：

	build B
	build A
	build C
	destroy C
	destroy A
	destroy B

由此可见，类成员变量在初始化列表中完成初始化时，初始化顺序与其声明顺序有关，而与其在初始化列表中的顺序无关（在严格级别下这两个顺序不一致会报错，见前文）。原因是成员变量初始化顺序与其在内存中顺序有关，而在内存中的排列顺序在编译期时根据声明的顺序确定。另外最先构造的成员最后被析构。

##2. 继承情况下的类构造析构顺序
将上文中的例子稍加改动，如果类C继承了类A和B，代码如下：

	#include <iostream>
	using namespace std;

	class A
	{
	public:
		A() { cout << "build A" << endl; }
		~A() { cout << "destroy A" << endl; }
	};

	class B
	{
	public:
		B() { cout << "build B" << endl; }
		~B() { cout << "destroy B" << endl; }
	};

	class C ： public A, public B
	{
	public:
		C() { cout << "build C" << endl; }
		~C() { cout << "destroy C" << endl; }
	};

	int main(int argc, char* argv[])
	{
		C c;
		return 0;
	}
输出结果为：

	build A
	build B
	build C
	destroy C
	destroy B
	destroy A
如果继承的顺序改为`class C : public B, public A`，输出结果为：

	build B
	build A
	build C
	destroy C
	destroy A
	destroy B
可以看出，父类的构造顺序与继承列表的顺序有关，构造时先构造父类，析构时先析构子类。

如果是**虚拟继承**的情况，



