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
* [第12步](section12/README.md) 添加数组的支持
* [插曲](interlude1/README.md) 一个简单的运算符优先级分析器
* [第13步](section13/README.md) 本地变量
* [第14步](section14/README.md) 变长参数
* [第15步](section15/README.md) 真正的语法分析器
* [第16步](section16/README.md) 扩展语法分析器
* [第17步](section17/README.md) 推断"let"表达式
* [第18步](section18/README.md) 插入运算符优先级分析器
* [插曲](interlude2/README.md) Ruby的对象模型
* [第19步](section19/README.md) 对象模型
* [插曲](interlude3/README.md) Ruby的编译问题
* [第20步](section20/README.md) 调度场解析组件的详细攻略
* [第21步](section21/README.md) 对符号类的基本支持；开始工作在attr_reader / attr_writer / attr_accessor上
* [第22步](section22/README.md) 转移到method_missing
* [第23步](section23/README.md) 真正的字符串：将字符串转换为对象
* [插曲](interlude4/README.md) 怎样实现闭包
* [第24步](section24/README.md) 外部作用域：清理并增加“main”对象
* [第25步](section25/README.md) 通过闭包实现define_method
* [第26步](section26/README.md) 尝试在STABS支持（debugging/gdb）
* [第27步](section27/README.md) 解析；开始删除'runtime.c'
* [第28步](section28/README.md) 将数字转换成Numbers
* [第29步](section29/README.md) 在我们新的Fixnum上操作
* [第30步](section30/README.md) 比较操作符
