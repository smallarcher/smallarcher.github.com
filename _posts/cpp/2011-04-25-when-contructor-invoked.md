---
layout: post
title: 究竟什么时候构造函数被调用
category: cpp
---

'''
    #include <stdio.h>
    #include <iostream>

    class complex
    {
    public:
        complex() {};
        complex(double r) : re(r), im(0)
        {
            std::cout << "Construct complex(double)" << std::endl;
        }
        complex(double r, double i) : re(r), im(i) {}

        complex(const complex& c) : re(c.re), im(c.im)
        {
            std::cout << "Copy contrcut complex(complex&)" << std::endl;
        }

        complex& operator=(const complex& c)
        {
            std::cout << "operator = " << std::endl;
            if (this != &c)
            {
                this->re = c.re;
                this->im = c.im;
            }
            return *this;
        }
        virtual ~complex() {}
        friend std::ostream& operator<<(std::ostream&, const complex&);
    private:
        double re, im;
    };

    std::ostream& operator<<(std::ostream& out, const complex& c)
    {
        out << c.re << ", " << c.im;
        return out;
    }

    int main(int argc, char *argv[])
    {
        complex b = 3;
        complex c = b;
        std::cout << b << std::endl;;
        std::cout << c << std::endl;;
        return 0;
    }

'''

构造函数是有关如何建立起给定类型的一个值的地方。当程序里需要一个某类型的值，而某个构造函数又能通过把所提供的值作为初始式或者被赋值的值，去创建一个这样的值的时候，这个构造函数就会被调用。因此，具有一个参数的构造函数也可能不需要现实地调用。

complex b = 3; 的意思是 complex b = complex(3);

摘自C艹程序设计语言特别版P241

不过，老实说，真心的还没见过哪个书上，或是谁的代码里面有complex b = 3这种形式的代码，通常都是complex b(3)这样的显示的调用不同的构造函数。

而且按照书中的解释complex b = complex(3)，这种写法应该是存在一次构造函数，和一次复制构造函数的调用的，结果运行结果显示并没有调用复制构造函数。。我郁闷。。。

后来，忍痛把**复制构造函数加了private**限定后，过不了编译了。也证明理论上是需要复制构造的，只是编译器太“聪明”了。不过，在没有加任何优化选项的情况下，编译器的这个行为，真的让人很没有安全感。。（GCC4.4.3）也没找到禁用这种默认优化的选项。 
 长这么大还不知道编译器到底私底下做了多少事情。惭愧。

未完待续。。。

