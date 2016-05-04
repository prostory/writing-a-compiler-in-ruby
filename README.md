# 使用Ruby自底向上的编写一个编译器

早在2008年3月，我开始发布一系列关于如何使用Ruby编写一个编译器的文章。使用自底向上的方法，也就是，以代码生成器开始和我的工作方式来替代先编写一个解析器的更传统的方法。这里是到目前为止我在我的博客上发布的所有部分。

[原文](http://hokstad.com/compiler)

##目录
* [第1步](section1/README.md) 为main函数创建一个简单的prolog / epilog
* [第2步](section2/README.md) 函数调用 / Hello World
* [第3步](section3/README.md) 链接表达式；添加子表达式
* [第4步](section4/README.md) 定义函数；添加运行时支持
* [第5步](section5/README.md) 数字字面量；if .. then .. else
* [第6步](section5/README.md) 匿名函数: lamba / call
* [第7步](section7/README.md) 使用lambda/call和函数参数
* [第8步](section8/README.md) 赋值，算法和比较
* [第9步](section9/README.md) 一个内建的"while"语句
* [第10步](section10/README.md) 测试编译器：原始的”解析器“
* [第11步](section11/README.md) 一个单独的发射器类输出汇编语言
