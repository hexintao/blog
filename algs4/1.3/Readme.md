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
	* `void push(Item item)`
	* `Item pop()`
	* `boolean isEmpty()`
	* `int size()`
		
* [1219：字符串解析](https://github.com/hexintao/blog/blob/master/algs4/1.2/1219.md)