---
layout: post
title: gprotobuf--雏形 
category: python
---

闲来无聊，最近围观到赖总关于[那些基于protobuf的RPC](http://blog.csdn.net/lanphaday/archive/2011/04/10/6313243.aspx)的点评，加上项目上使用protobuf也有些时日了，并且最近还在看gevent，最终就有了[gprotobuf](https://github.com/zwxyswd/gprotobuf)这个想法：基于gevent和protobuf的RPC框架。

总得来说，没什么亮点，只是机械的组合。代码结构是照着工作项目里面搬的，RPC消息格式其实大同小异，对gevent也只是一知半解，只知道对python标准库中的socket，urllib2等进行了hack，其他的网络相关的API都不知道怎么用。文档太少，跟tornado一个级别的文档量。tornado可以说是纯python的，看源码也算是颇有心得。gevent就麻烦了，首先基于libevent，然而libevent并不熟悉，加上gevent非纯python，混搭了python，C，再加上一个c-ares，让人心生畏惧。我还是决定从libevent着手，不过这是下一步的事情了。

消息格式的设计，其实感觉都大同小异，个人觉得还是很认同，simpler is better。

{% highlight java %}

    enum ResponseType {
    RESPONSE_OK = 0;
    RESPONSE_ERROR = 1;
    };


    message Request {
    required bytes uuid = 1;
    required string service = 2;
    required string method = 3;
    optional bytes request = 4;
    };

    message Response {
    required bytes uuid = 1;
    required ResponseType type = 2;
    optional bytes response = 3;
    optional string error = 4;
    };


    message Error {
    required string info = 1;
    };

{% endhighlight %}
Parallel Pipelining是必须的，所以有uuid。

最后的代码中message Error 并没有用到，服务端出现错误，客户端本来就无能为力，只能log一下，所以只返回以一个字符串，用Error再包一层也无意义。
目前的不足：

1. 毫无测试，只有一个Echo的demo

2. 错误处理很弱，还有不少地方的exception都只是一个pass  ，这个真心的很尴尬。

3. 服务端没有安全相关的措施，例如，频繁收到某ip的非法格式的消息，采取严厉措施。

4. 没有benchmark，没有和tornado的实现对比，对性能一无所知。

5. 我忘记写setup.py了。 

6. 无任何有意义的注释。（目前注释都是emacs生成，占据50%的代码行数了吧  ）

**7. gevent文档里面并没有看到更新socket监听事件的函数，只有add和cancel，所以客户端连进来之后，直接add了READ和WRITE，而没有动态的修改需要监听的socket事件，这个地方让我很纠结，兴许add可以当作修改用？哎，也不知道该问谁。。。

最近心情略显骚动，打乱了原本的阅读和学习计划，让我今天很不爽。我还是应该先稳稳的走自己的路，让别人打的去。。。**

最近闲得蛋碎，开始再度深入学习C艹，大半年没碰C艹，现在再来围观，与大学时的意境已大相径庭，但是我还是更愿意从一个纯C的东西开始，比如libevent，或是redis，或是libev呢～

