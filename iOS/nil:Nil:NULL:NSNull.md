**原文地址：** [NSHipster](http://nshipster.com/nil/)

**原文作者：**[Mattt Thompson](http://nshipster.com/authors/mattt-thompson/)

## 引子
“虚无”是一个偏哲学的概念，但是理解“虚无”又是一项比较实际的事情。我们是物质世界的一份子，却一直努力用逻辑思维去理解宇宙中存在的种种未知。计算机，作为一个逻辑系统的实体代表，同样也面临着一个棘手的问题－－如何用“有”来定义“无”？
<!--more-->

在OC中，“空值”有好几种定义。为什么会有多种定义方式呢？想要回答这个问题，就不得不回到之前的[老套路](http://nshipster.com/ns_enum-ns_options/)－－OC是如何将C（程序式语言）和类Smalltalk（面向对象语言）连接起来？C语言中，“0”就代表着“空值”，`NULL`代表指针为空（含义即指针的内容为0）。

OC在C语言的基础上，增加了一个“空值”的定义——`nil`。`nil`的含义是一个对象的指针为空。虽然nil和NULL语义上有所区别，但本质上两者是等同的。

从框架的角度来说，Foundation框架定义了`NSNull`和一个类方法`+null`，而后者的返回值就是一个`NSNull`对象。`NSNull`和`nil`、`NULL`的区别在于：前者是一个实际存在的对象，而不是仅仅定义值为空。另外在Foundation/NSObjectRuntime.h中定义：`Nil`代表一个类指针为空。这个与`nil`只在大小写上有所区别却鲜为人知的`Nil`并不经常使用，不过它也是“空值”的一个定义。

## 关于nil的一些东西
OC对象通过`-alloc`方法开辟新内存，开始自己生命的旅程。一个对象的内容被初始为0，这意味着这个对象内部的所有指针都为`nil`。所以，在类的实例化方法中（`-init`），并不需要再重新执行`self.(association) = nil`类似的过程。

nil最被人熟知的一点是：尽管本身为空，但它却能从外界接收消息。

其他语言中，像C++，让一个空值调用方法会导致系统崩溃，但是在OC中，给`nil`发送消息的后果仅仅是返回一个0。这大大简化了程序的编写，因为不需要在执行方法前再检查对象是否为nil：

``` mm
// 举例来说，类似于这种判断语句...
if (name != nil && [name isEqualToString:@"Steve"]) { ... }

// ...可以被简化为:
if ([name isEqualToString:@"Steve"]) { ... }
```
在了解了nil的工作原理之后，OC语言允许我们这样简化代码，这也成了OC语言的一个特征，而且这种简化操作并不会给我们的应用造成潜在的bug。不过，我们得留意某些情况下值不允许为nil，这时候，我们就需要提前检查并做出错误提示或者通过增加`NSParameterAssert `抛出异常进行提示。
## NSNull：“有”定义“无”
NSNull在Foundation和其他一些框架中被大量使用，它被用来判断在一些集合类的对象，例如NSArray、NSDictionary中，不能含有nil。你可以把NSNull看作是一个装有NULL或nil值的箱子，就像下边这样：

``` mm
NSMutableDictionary *mutableDictionary = [NSMutableDictionary dictionary];
mutableDictionary[@"someKey"] = [NSNull null]; // 给 `someKey`的值设为null空对象
NSLog(@"Keys: %@", [mutableDictionary allKeys]); // @[@"someKey"]
```
总的来说，有以下4种“空值”的定义方式需要每一个OC编程人员知道：

| 符号    | 值            | 含义                     |
| ------- | ------------- | ------------------------ |
| NULL    | (void *)0     | C语言，指针为空          |
| nil     | (id)0         | OC语言，对象为空         |
| Nil     | (class)0      | OC语言，类为空           |
| NSNull  | [NSNull null] | OC语言，代表一个null对象 |
