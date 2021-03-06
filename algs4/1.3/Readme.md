1.3 背包、队列和栈

Api 一览：

1. 背包：不支持删除，迭代顺序不确定且与用例无关。
	* `void add(Item item)`
	* `boolean isEmpty()`
	* `int size()`
2. 队列：FIFO，先进先出队列。元素入列顺序和出列顺序相同。
	* `void enqueue(Item item)`
	* `Item dequeue()`
	* `boolean isEmpty()`
	* `int size()`
3. 栈：LIFO，下压栈，简称栈。元素入栈顺序和出栈顺序相反。
	* `void push(Item item)`'
	* `Item pop()`
	* `boolean isEmpty()`
	* `int size()`

要实现上述三种数据结构，目标有三个：

1. 可以处理任意类型的数据；
2. 所需的空间总是和集合的大小成正比；
3. 操作所需的时间总是和集合的大小无关。

#### 用数组实现：

目标1通过泛型来实现，这里跳过不讲。
用数组来实现上述三种数据模型的代码类似，队列和栈因为操作元素的顺序不一样，在插入元素和删除元素时的实现不一样。用数组表示的话，数据模型所容纳的元素数量便被限定了，为此又实现了动态扩展的方法保证元素数量可以自由扩充。动态扩展的思路是：添加元素时，当元素数量等于数组的大小时，就将数组的大小扩大一倍并将元素重新复制进去。删除元素时，当元素数量等于数组长度的四分之一时，则将数组的长度缩减一倍并复制元素，这样元素只占数组的二分之一，还有足够的空间可以满足插入新的元素。实现了目标2，不过这里所需的空间是和集合的大小乘一个常数成正比。

但是数组调整大小的操作耗时和元素的数量成正比，没有实现目标3。
引申出用新的数据结构的实现——链表。

#### 用链表实现
链表是递归的数据结构，它要么为空 null，要么指向一个节点，每个节点都含有一个泛型的元素和一个指向另一个节点的引用。
##### 用链表实现栈
1. `push`：表头插入节点，操作时间和集合大小无关
2. `pop`：删除表头节点，当只有一个节点时需要判断下，操作时间和集合大小无关
3. `isEmpty`：检查头部节点是否为 null 即可，操作时间和集合大小无关
4. `size`：用局部变量记录元素的个数即可

##### 用链表实现队列
内部不仅要保存头部节点的引用 first，也要保存尾部节点的引用，last。

1. `enqueue`：表尾插入节点，并更新 last。操作时间和集合大小无关
2. `dequeue`：表头删除节点，并更新 first，操作时间和集合大小无关

##### 用链表实现背包
栈的实现中，把 `pop` 部分去掉即是。

#### 使用新的数据抽象解决问题的思路
1. 定义 API；
2. 根据特定的应用场景开发用例代码；
3. 描述一种数据结构，并在 API 所对应的抽象数据类型的实现中根据它定义类的实例变量，即解决如何存储值的问题；
4. 描述算法，并实现类中的实例方法，即解决如何操作值的问题；
5. 分析算法的性能特点。

[代码：链表实现的背包、队列和栈](https://github.com/hexintao/blog/blob/master/algs4/1.3/bagQueueAndStack.md)

* [1.3.49：栈与队列](https://github.com/hexintao/blog/blob/master/algs4/1.3/1349.md)
* [1.3.45：栈的可生成性](https://github.com/hexintao/blog/blob/master/algs4/1.3/1345.md)
* [1.3.44：文本编辑器的缓冲区](https://github.com/hexintao/blog/blob/master/algs4/1.3/1344.md)
* [1.3.43：文件列表](https://github.com/hexintao/blog/blob/master/algs4/1.3/1343.md)
* [1.3.40：前移编码](https://github.com/hexintao/blog/blob/master/algs4/1.3/1340.md)
* [1.3.39：环形缓冲区](https://github.com/hexintao/blog/blob/master/algs4/1.3/1339.md)
* [1.3.37-1.3.38：Josephus 问题](https://github.com/hexintao/blog/blob/master/algs4/1.3/1337.md)
* [1.3.35：随机队列](https://github.com/hexintao/blog/blob/master/algs4/1.3/1335.md)
* [1.3.33：Deque](https://github.com/hexintao/blog/blob/master/algs4/1.3/1333.md)
* [1.3.32：Steque](https://github.com/hexintao/blog/blob/master/algs4/1.3/1332.md)


提高题⬆️

* [1.3.31：DoubleNode 构造双向链表](https://github.com/hexintao/blog/blob/master/algs4/1.3/1331.md)
* [1.3.30：链表的反转](https://github.com/hexintao/blog/blob/master/algs4/1.3/1330.md)
* [1.3.27：链表中最大的节点 - 1.3.29：环形链表实现 Queue](https://github.com/hexintao/blog/blob/master/algs4/1.3/1319.md)
* [1.3.19：删除链表尾节点](https://github.com/hexintao/blog/blob/master/algs4/1.3/1319.md)

链表练习⬆️
	
* [1315：输出倒数第k个字符](https://github.com/hexintao/blog/blob/master/algs4/1.3/1315.md)
* [1312：栈的 copy](https://github.com/hexintao/blog/blob/master/algs4/1.3/1312.md)
* [139-1310：补全括号-中序后序表达式转换](https://github.com/hexintao/blog/blob/master/algs4/1.3/139.md)
* [135-136：N的二进制](https://github.com/hexintao/blog/blob/master/algs4/1.3/135.md)
* [134：Parentheses](https://github.com/hexintao/blog/blob/master/algs4/1.3/134.md)
* [133：不可能序列](https://github.com/hexintao/blog/blob/master/algs4/1.3/133.md)