---
layout: post
title: C++虚函数表结构及内存布局
category: cpp
---

最近被问及C++的多态实现原理，老实说，除了知道个一个指针和虚表的概念，其他并不了解。今天稍微闲点，回头来拿指针指來指去的喵喵虚表到底是个什么结构。一切建立在虚表存在和虚表指针存在且存在在对象首部的理论基础上。 

{% highlight cpp %}
    class Base
    {
    public:
        virtual void f() { cout << "Base::f" << endl; }
        void g() { cout << "Base::g" << endl; }
        virtual void h() { cout << "Base::h" << endl; }
    private:
        int n;
    };

    class Derive1 : public Base
    { // 重写一个虚函数
    public:
        virtual void f() { cout << "Derive1::f" << endl; }
        // virtual void h() { cout << "Derive1::h" << endl; }
    };

    class Derive2 : public Base
    { // 重写两个虚函数
        virtual void f() { cout << "Derive2::f" << endl; }
        virtual void h() { cout << "Derive2::h" << endl; }
    };

    class Derive3 : public Base
    { // 重写两个函数，并增加一个虚函数
        virtual void f() { cout << "Derive3::f" << endl; }
        virtual void j() { cout << "Derive3::j" << endl; }
        virtual void h() { cout << "Derive3::h" << endl; }
    };

    typedef void (*Func)(void);
    先上代码，1个基类，3个类继承它，并以不同的特色重写基类的函数，然后定义了相应的函数指针。

    先用段代码研究Base的结构：

    void BaseStruct()
    {
        Base base;
        Func pFun = NULL;
        cout << "Base大小" << sizeof(Base) << endl;
        // Base的虚表指针: 取Base的前4个字节，强转成指针，
        int* pBaseVtable = (int*)(*(int*)(&base));
        cout << "虚表地址:" << pBaseVtable << endl;
        cout << "虚表第一项的地址:" << (int*)(*pBaseVtable) << endl;
        cout << "虚表第一项调用输出:" << endl;
        pFun = (Func)(*pBaseVtable);
        pFun();

        cout << "下一个的地址:" << (int*)(*(pBaseVtable + 1)) << endl;
        cout << "下一个的调用输出:" << endl;
        pFun = (Func)(*(pBaseVtable + 1));
        pFun();
        cout << "再下一个是什么呢:" << (int*)(*(pBaseVtable + 2)) << endl;
        cout << "Base测试结束" << endl;
    }

{% endhighlight %}
测试结果是：

Base大小8
虚表地址:0x8049428
虚表第一项的地址:0x8048fda
虚表第一项调用输出:
Base::f
下一个的地址:0x8049006
下一个的调用输出:
Base::h
再下一个是什么呢:0x72654437
Base测试结束
验证结论：

1. 从Base对象的大小和最后的结果看出，对象首部确有一个指针，指向某个地方，这个地方顺序放着对象所属的类的虚函数地址。

2. 虚表中函数的顺序和类中的虚函数的声明顺序一致，交换f，h函数的顺序可以验证。

3. 而虚表后的那个位置的值，和平台实现有关，上述代码在ubuntu10.04 + gcc4.4.3下得到一个非法地址（没看出是干什么的），而在windows+mingw环境下，虚表后的那个位置正好是个0, 也就是个null指针。～

然后看看Derive1，重写其中一个虚函数后，Derive1的虚表内容

{% highlight cpp %}
    void Derive1Struct()
    {
        Derive1 derive1;
        Func pFun = NULL;
        cout << "Derive1大小" << sizeof(Derive1) << endl;
        int* pDeriveVtable = (int*)(*(int*)(&derive1));
        cout << "虚表地址:" << pDeriveVtable << endl;
        cout << "虚表第一项地址:" << (int*)(*pDeriveVtable) << endl;
        cout << "虚表第一项调用输出:" << endl;
        pFun = (Func)(*pDeriveVtable);
        pFun();

        cout << "下一个的地址:" << (int*)(*(pDeriveVtable + 1)) << endl;
        cout << "下一个的调用输出:" << endl;
        pFun = (Func)(*(pDeriveVtable + 1));
        pFun();
        cout << "再下一个是什么呢:" << (int*)(*(pDeriveVtable + 2)) << endl;
        cout << "Derive1测试结束" << endl;
    }
{% endhighlight %}
输出结果：

Derive1大小8
虚表地址:0x8049418
虚表第一项地址:0x8049032
虚表第一项调用输出:
Derive1::f
下一个的地址:0x8049006
下一个的调用输出:
Base::h
再下一个是什么呢:0
Derive1测试结束
验证结论：

1. Derive1的虚表地址和Base的虚表地址不一样。写到这里，我又去弄了个Derive0，直接继承Base，不重写任何函数，发现虚表地址和Base的还是不一样，几乎可以证明无论怎么样，一旦继承自一个有虚函数的类，必然会有一份自己的虚表，不会因为和基类的虚表一样而重用。

2. Derive1的虚表中，函数的顺序和基类的函数好像是顺序相同，（为什么好像是，因为截止到这里证据其实并不充分），重写过的虚函数换上了新地址，没重写的过，仍然沿用基类的地址。

3. 这次，虚表的末尾变成了null，像是虚表结束符。

再看看Derive2，重写了两个虚函数的情况。

{% highlight cpp %}
    void Derive2Struct()
    {
        Derive2 derive2;
        Func pFun = NULL;
        cout << "Derive2大小" << sizeof(Derive2) << endl;
        int* pDeriveVtable = (int*)(*(int*)(&derive2));
        cout << "虚表地址:" << pDeriveVtable << endl;
        cout << "虚表第一项地址:" << (int*)(*pDeriveVtable) << endl;
        cout << "虚表第一项调用输出:" << endl;
        pFun = (Func)(*pDeriveVtable);
        pFun();

        cout << "下一个的地址:" << (int*)(*(pDeriveVtable + 1)) << endl;
        cout << "下一个的调用输出:" << endl;
        pFun = (Func)(*(pDeriveVtable + 1));
        pFun();
        cout << "再下一个是什么呢:" << (int*)(*(pDeriveVtable + 2)) << endl;
        cout << "Derive2测试结束" << endl;
    }

{% endhighlight %}
输出结果：

Derive2大小8
虚表地址:0x8049408
虚表第一项地址:0x804905e
虚表第一项调用输出:
Derive2::f
下一个的地址:0x804908a
下一个的调用输出:
Derive2::h
再下一个是什么呢:0
Derive2测试结束
验证结论：

1. 子类虚表地址和基类不一样。

2. 虚函数在虚表中的顺序维持和基类一直，交换Derive2中的虚函数声明顺序，可以验证。

3. 虚表末尾也是null。

这个测试结果基本上对Derive1的结论进行了验证。

再看看Derive3，重写2个基类虚函数，并自己增加了一个虚函数。

{% highlight cpp %}

    void Derive3Struct()
    {
        Derive3 derive3;
        Func pFun = NULL;
        cout << "Derive3大小" << sizeof(Derive3) << endl;
        int* pDeriveVtable = (int*)(*(int*)(&derive3));
        cout << "虚表地址:" << pDeriveVtable << endl;
        cout << "虚表第一项地址:" << (int*)(*pDeriveVtable) << endl;
        cout << "虚表第一项调用输出:" << endl;
        pFun = (Func)(*pDeriveVtable);
        pFun();

        cout << "下一个的地址:" << (int*)(*(pDeriveVtable + 1)) << endl;
        cout << "下一个的调用输出:" << endl;
        pFun = (Func)(*(pDeriveVtable + 1));
        pFun();

        cout << "下一个的地址:" << (int*)(*(pDeriveVtable + 2)) << endl;
        cout << "下一个的调用输出:" << endl;
        pFun = (Func)(*(pDeriveVtable + 2));
        pFun();
        cout << "再下一个是什么呢:" << (int*)(*(pDeriveVtable + 3)) << endl;
        cout << "Derive3测试结束" << endl;
    }
{% endhighlight %}
输出结果：

Derive3大小8
虚表地址:0x80493f0
虚表第一项地址:0x80490b6
虚表第一项调用输出:
Derive3::f
下一个的地址:0x804910e
下一个的调用输出:
Derive3::h
下一个的地址:0x80490e2
下一个的调用输出:
Derive3::j
再下一个是什么呢:0
Derive3测试结束
验证结论：

1. 其他的都一样，新增加的虚函数放在了虚表中基类虚函数的后面，Derive3的定义中，特意把新增的虚函数放在了基类的两个虚函数之间，来验证这一点。另外，删除Derive3中的f或者h函数，Derive3的虚表中函数的次序仍然是这样，基本也可以验证子类新增的虚函数在虚表中在父类的虚函数的后面。

以上都是单继承的情况下，虚表的内存布局。

多继承的时候，情况似乎更多了。初步验证了下，并没有再写测试代码验证了，而是直接那gdb查看内存布局了，不然太麻烦了，但是没有做记录：

1. 多继承情况下，针对每个有虚函数的基类，相应的有一个虚表指针。

2. 子类自己的虚函数，往第一个虚表指针指向的虚表的后面加。

上面说的多继承也只是简单的多继承，至于菱形继承，暂时没有任何考察。下次吧。。。天天看这么多类，我也是个累了啊。。。

