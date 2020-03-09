
本文针对使用过 `Block` 结构，对 `Block` 的背后实现不太了解的读者。
这两天刚看完有关 `Block` 的一些内容，做一个小结，主要回答以下问题：

1. 都说 Block 可以看做是 Objective-C 中的对象，它的本质到底是什么？
2. Block 使用起来很方便，因为它能捕捉到其外部的变量，并进行访问和修改，这是怎么实现的？
3. Block 为什么要用 copy 修饰符？
4. Block 的循环引用到底是怎么回事？怎么解决？  

这些问题上网一搜一大堆，十分常见，希望这篇从稍微底层一点的解释，能给你一点小小的惊喜。

#### 问题 1. Block 的本质
最简单的一个 Block 定义：

``` mm
void (^aBlock)(void) = ^ {
	NSLog(@"这是一个最简单的 block。");
};
```
上述形式是我们平时简化之后的写法，其实最完整的写法是：

``` mm
void (^aBlock)(void) = ^void(void) {
	NSLog(@"这是一个最简单的 block。");
};
```
使用 `clang -rewrite-objc main.m` 命令将其转换为 .cpp 文件后，可以看到代码增加到一万多行，我们直接拉到最底部，可以找到 `main` 的定义：

``` mm
int main(int argc, const char * argv[]) {
    void(*aBlock)(void)= ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
    return 0;
}
```
这。。。太麻烦了，不过静下心来：先看左边，`void(*aBlock)(void)` 是典型的 C 函数指针，也就是说，`aBlock` 是一个变量，里边存储的是一个函数的地址，而这个函数是不带参数不带返回值的类型。再看右边，先不管前边的好多层括号，`&` 是取地址的操作，也就是取 `__main_block_impl_0` 的地址并赋值。前边的 `void(*)()` 是类型的强转，为了和左边要求的函数类型相匹配。后边的括号相当于是 `__main_block_impl_0` 的入参。所以稍微简化一下：

``` mm
__main_block_impl_0 temp = __main_block_impl_0( (void *)__main_block_func_0, &__main_block_desc_0_DATA);
aBlock = &temp;
```
到这，离第一个问题的答案已经不远了，`Block` 的本质这不就是一个指向 `__main_block_impl_0` 类型的指针嘛。至于 `__main_block_impl_0` 的定义，我们全局搜索一下就可以找到：

``` mm
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```
结构体。其实也不惊讶，`Objective-C` 中的类和对象本质不都是结构体嘛。这个结构体看起来也比较复杂，其实它只有两个成员变量：

``` mm
struct __block_impl impl;
struct __main_block_desc_0* Desc; 	// 指针变量，指向  __main_block_desc_0 类型
```
剩下的一坨是结构体的初始化方法（构造方法）。对照前边的 `main` 的代码，`__main_block_func_0` 对应结构体初始化方法中的 `fp`，`__main_block_desc_0_DATA` 对应初始化方法中的 `desc`，`flags` 默认是 0。通过这两个参数，我们看一下这个结构体是怎么初始化的。先看 `__block_impl` 的定义：

``` mm
struct __block_impl {
    void *isa;      // isa 指针
    int Flags;      // 标志位
    int Reserved;   // 保留位
    void *FuncPtr;  // 这不是函数指针！而是指针函数，也就是带返回值的函数，返回值是一个万能类型的指针 void *
};
```
经过初始化之后，变化如下：

``` mm
impl.isa = &_NSConcreteStackBlock;		// block 的类型
impl.Flags = flags;						// 默认为 0
impl.FuncPtr = fp;						// __main_block_func_0
Desc = desc;							// __main_block_desc_0_DATA
```
`__main_block_func_0` 和 `__main_block_desc_0_DATA` 又是啥？继续搜索：

``` mm
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
	NSLog((NSString *)&__NSConstantStringImpl__var_folders_90_cs8g8nkx25dckdwfnf1n66kr0000gn_T_main_00297a_mi_0);  // __main_block_func_0 中仅有一句 NSLog，这不正是 block 的定义吗？打印的结果是一个全局函数，搜索即可找到该函数的定义如下：
}

static __NSConstantStringImpl __NSConstantStringImpl__var_folders_90_cs8g8nkx25dckdwfnf1n66kr0000gn_T_main_00297a_mi_0 __attribute__ ((section ("__DATA, __cfstring"))) = {__CFConstantStringClassReference,0x000007c8,"\350\277\231\346\230\257\344\270\200\344\270\252\346\234\200\347\256\200\345\215\225\347\232\204 block\343\200\202",33}; // 翻译过来肯定是：这是一个最简单的 block。
```

到这里，我们可以看到，`aBlock` 的声明和实现就关联起来了，引用的关系大致为：

`aBlock` -> `__main_block_impl_0` -> `__block_impl impl.FuncPtr` -> `__main_block_func_0`。

再看 `__main_block_desc_0_DATA`：

``` mm
__main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
```
一个全局的函数，从字面看，存储着 `__main_block_impl_0` 的大小信息，第一个 `0` 意义暂未知。

来一个总结：

* `__main_block_impl_0`：结构体类型。定义了 `Block` 的具体类型，也算是 `Block` 的本质。
* `__main_block_func_0`： 全局的静态函数，是 `Block` 的具体实现。
* `__main_block_desc_0_DATA`：全局函数，相当于是 `Block` 的额外信息描述。
* `__block_impl`：`__main_block_impl_0` 内部的一个成员变量，把 `__main_block_impl_0` 和 `__main_block_func_0` 连接起来。

这里边的名称也是有规律的，基本是按照： `Block` 所属的方法名+在其所属方法中的定义顺序。例如，例子中我们的 `Block` 是定义在 `main` 方法中的第一个 `Block`。

`OC` 中所有的对象本质都是结构体，其都有一个 `isa` 指针，指向自己的类型（其实是类对象，也是结构体，类对象的 `isa` 指向其元类），通过以上我们也可以看到 `block` 内部也有 `isa` 指针，所以也可以把 `block` 当做对象来看待，例子中的 `block` 当然就是 `_NSConcreteStackBlock` 类型，位于栈区的 `block`。

除此以外，还有 `_NSConcreteGlobalBlock` 和 `_NSConcreteMallocBlock` 类型，顾名思义，前者是全局类型，后者则位于堆区。其中如果 `block` 定义时就是全局类型，或者其内部没有捕获任何外部的局部变量，此时的 `block` 就是 `_NSConcreteGlobalBlock` 类型。除此以外，其他定义的 `block` 默认就是 `_NSConcreteStackBlock` 类型，而一旦对后者执行 `copy` 操作，其类型就会变为 `_NSConcreteMallocBlock `。

#### 问题 2. Block 捕获并修改外部变量

##### 2.1 基本类型变量的捕获
可是平时我们使用 `block` 时，基本都会在 `block` 中用到外部变量，我们再看一个简单的例子：

``` mm
__block int val = 0;
int valNoBlock = 5;
void (^blk)(void) = ^{
	val = valNoBlock + 1;
};
blk();
NSLog(@"%d", val);
```

我们都知道，如果 `val` 声明时没有使用 `__block` 进行修饰，编译就会报错：

`Variable is not assignable (missing __block type specifier)`

而 `block` 内部只是读取并没有修改 `valNoBlock` 的值，所以就不用加 `__block` 修饰符。使用了 `__block` 之后，怎么就能修改外部的局部变量了？同样，我们依然对上述代码进行 `clang -rewrite-objc main.m` 操作。

拉到最底部，看 `main` 方法中两个变量的定义：

``` mm
__attribute__((__blocks__(byref))) __Block_byref_val_0 val = {(void*)0,(__Block_byref_val_0 *)&val, 0, sizeof(__Block_byref_val_0), 0};
int valNoBlock = 5;
```

好复杂！简化一下：

``` mm
__Block_byref_val_0 val = {(void*)0,(__Block_byref_val_0 *)&val, 0, sizeof(__Block_byref_val_0), 0};
```

加了 `__block` 修饰符的 `val` 变成了这么大一坨，类型也不再是简单的 `int` 了，而是 `__Block_byref_val_0`，这是个什么鬼？搜索得到定义：

``` mm
struct __Block_byref_val_0 {
	void *__isa;
	__Block_byref_val_0 *__forwarding;
	int __flags;
	int __size;
	int val;
};
```

没错，又是结构体。经过上述赋值，整个结构体的值变化如下：

``` mm
struct __Block_byref_val_0 {
	void *__isa;						// (void*)0
	__Block_byref_val_0 *__forwarding;	// &val
	int __flags;						// 0
	int __size;							// sizeof(__Block_byref_val_0)
	int val;							// 0
};
```
我们这里重点看 `__forwarding` 和最后一个 `int val`。前者是一个指针，指向自己。后者保存的是该变量的初始值。而作为对比，`valNoBlock` 则依旧是简单的 `int` 类型，没发生什么变化。

我们接着往下看 `block` 的定义部分，像第一个问题中一样，简化之后是：

``` mm
__main_block_impl_0 temp = __main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, valNoBlock, (__Block_byref_val_0 *)&val, 570425344);
blk = &temp;
```

比第一个例子中多了些东西，我们看一下 `__main_block_impl_0` 的定义：

``` mm
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int valNoBlock;
  __Block_byref_val_0 *val; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _valNoBlock, __Block_byref_val_0 *_val, int flags=0) : valNoBlock(_valNoBlock), val(_val->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

相比第一个例子，多了两个成员变量：`int valNoBlock` 和 `__Block_byref_val_0 *val`，这正好和 `blk` 中使用到的两个变量相对应。这里的 `valNoBlock(_valNoBlock), val(_val->__forwarding)` 语法不懂什么意思，希望知道的可以告知。除了和第一个例子一样的初始化之外，加上前边的 `block` 的定义，我们可以看到：没有 `__block` 修饰的 `valNoBlock` 是直接把值传了进去，`block` 内部会有一个同样类型的变量接收其值，而 `__block` 类型的变量，`block` 内部对应有一个指针变量，存储着变量结构体的地址。到此，`__block` 的面纱我们基本也揭开了。

但是，`block` 还没有执行呢，我们接着往下看 `block` 定义完之后是怎么调用的。很简单的一句：

``` mm
((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
```

通过第一个例子我们知道，`block` 的实现是形如 `__main_block_func_0` 的函数，而它的地址正是存放在 `block` 中的成员变量 `impl.FuncPtr` 中的，所以这一句就是直接调用 `__main_block_func_0` 即 `block` 的实现函数。我们看一下 `__main_block_func_0` 的实现：

``` mm
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
	__Block_byref_val_0 *val = __cself->val; 	// bound by ref
	int valNoBlock = __cself->valNoBlock;		// bound by copy
	(val->__forwarding->val) = valNoBlock + 1;
}
```

`__cself` 类似于 `OC` 中的 `self`，在这里就是前边执行 `block` 时传入的 `blk`，或者更彻底的，就是那个 `__main_block_impl_0` 类型的结构体。从这里，我们更能直观的看到，当 `block` 执行时，有没有 `__block` 修饰的变量的区别。`val` 是个指针变量，取值自 `__main_block_impl_0` 结构体中的 `val` 变量，而后者的值是在 `block` 定义时指定的，存储的是 `__block` 类型变量，即 `__Block_byref_val_0` 结构体的地址。说白了，经过层层传递，终于把 `__block` 类型变量的地址传递到了 `block` 的实现函数中。

接着，**重新又定义**了一个 `int` 类型的 `valNoBlock`，并把外界变量的值赋给该变量，所以不加 `__block` 修饰的变量，如果在 `block` 内部对其修改，外界变量的值是不会跟着变的，它们是复制出来的两个完全独立的变量。这里有个小问题，例子中是 `int` 类型的变量，所以直接赋值没什么疑问，如果换作是对象，这一步对应的是什么操作呢？`copy`？`retain`？

最后一句：

`(val->__forwarding->val) = valNoBlock + 1;`

`(val->__forwarding->val)` 指向的是 `__Block_byref_val_0` 结构体中 `int` 类型的 `val` 成员变量，前边咱们提到正是它保存着 `__block` 类型变量的初始值，现在，修改的也正是它的值。所以，它变化，外界变量的值也会变化，因为它们指向的就是同一个结构体中的成员变量。

##### 2.2 对象类型的捕获
例子：

``` mm
__block NSString *str = [NSString stringWithFormat:@"%@", @"Hello"];
__block id aObj;
NSString *strNoBlock = @"world!";
void (^blk)(void) = ^{
	str = [NSString stringWithFormat:@"%@ %@", str, strNoBlock];
	aObj = str;
};
blk();
NSLog(@"%@", str);
```

这里使用 `stringWithFormat:` 初始化是因为 `__block` 类型的 `NSString` 如果通过字符串直接赋值，`clang` 过不去。`aObj` 存在的意义是 `clang` 之后的代码比 `NSString` 的简单，易读。

`__block` 类型的 `NSString` 就会变成如下的结构体：

``` mm
struct __Block_byref_str_0 {
 	 void *__isa;
	__Block_byref_str_0 *__forwarding;
	 int __flags;
	 int __size;
	 void (*__Block_byref_id_object_copy)(void*, void*);
	 void (*__Block_byref_id_object_dispose)(void*);
	 NSString *str;
};
```

对比基本类型的变量，多了两个函数指针：`__Block_byref_id_object_copy` 和 `__Block_byref_id_object_dispose`。 最后一个 `NSString *str`，同样也是为了存储值，只不过本例中是对象，要注意一点，这个对象的内存修饰符是默认的 `__strong`，而且因为是对象，所以它的内存也是由编译器管理的，至于怎么管理，下边会说。

`__block` 变量定义为，以 `aObj` 为例，简化之后为：

``` mm
__Block_byref_aObj_1 aObj = {(void*)0,(__Block_byref_aObj_1 *)&aObj, 33554432, sizeof(__Block_byref_aObj_1), __Block_byref_id_object_copy_131, __Block_byref_id_object_dispose_131};
```

同样，传的也是自身的地址，对应前边两个函数指针，这里传递的是两个全局静态函数：

``` mm
static void __Block_byref_id_object_copy_131(void *dst, void *src) {
 _Block_object_assign((char*)dst + 40, *(void * *) ((char*)src + 40), 131);
}
static void __Block_byref_id_object_dispose_131(void *src) {
 _Block_object_dispose(*(void * *) ((char*)src + 40), 131);
}
```

整个流程大致为：`__block` 类型的对象，其结构体中会绑定两个函数的地址，在 `Block` 定义时，对象的结构体就传入了 `Block` 中。当 `Block` 从栈区被拷贝到堆区时，`__block` 类型的对象的 `__Block_byref_id_object_copy_131` 方法，即 `_Block_object_assign` 方法就会自动调用，相当于 `Block` 持有了该 `__block` 对象，而当 `Block` 从堆区释放时，`__block` 对象的 `__Block_byref_id_object_dispose_131` 方法，即 `_Block_object_dispose` 就会自动调用，解除 `Block` 对 `__block` 对象的持有。也就是说，系统通过 `Block` 的拷贝和释放时机，完成对 `Block` 捕获的对象类型的内存管理。

使用时，为了可以在 `block` 内对外部的变量进行修改，简简单单的一个 `__block` 修饰符，其背后可以说发生了翻天覆地的变化。我们这里说的只是针对局部自动变量，对于全局变量、全局静态变量以及局部静态变量的修改，这里并未涉及，思路完全一样，可以自己动手分析。

#### 问题 3. Block 为什么要用 copy 修饰符？
经常使用 `Block` 的人都知道，如果要创建一个 `Block` 类型的属性变量，要使用 `copy` 修饰符。为什么呢？使用其他修饰符，例如 `strong` 会有什么后果？

我们先来想这个问题，为什么要使用属性变量呢？几乎绝大部分时候都是为了要在该对象的定义域之外使用该变量，对于 `Block` 同样如此。通过前边两个问题，我们还知道，对于 `_NSConcreteStackBlock` 类型的 `Block` 来说，不管是 `Block` 还是 `__block` 类型的变量，他们本质都是结构体变量，而且是位于栈上的结构体。所以，出了定义的作用域之后，它们的内存就会被释放掉。所以为了不让它们释放掉，就需要将其拷贝到堆区，由我们自己控制何时释放，当然，现在都是系统通过 `ARC` 机制自动控制。另外，在 `ARC` 环境中，[官方其实对这个问题也有说明](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/WorkingwithBlocks/WorkingwithBlocks.html)：

> Note: You should specify copy as the property attribute, because a block needs to be copied to keep track of its captured state outside of the original scope. This isn’t something you need to worry about when using Automatic Reference Counting, as it will happen automatically, but it’s best practice for the property attribute to show the resultant behavior. 

所以，现在写代码，基本可以忽略 `Block` 必须使用 `copy` 的顾忌，因为系统会自动判断是不是需要将 `Block` 拷贝到堆区，这跟我们是否使用 `copy` 无关。

#### 问题 4. Block 的循环引用到底是怎么回事？
`Block` 的循环引用是怎么回事儿？看一个简单的例子：

``` mm
/// AbstractComputer.m

typedef void(^blk_t)(void);

@interface AbstractComputer() {
    NSInteger _aIntegerInstance;
}
@property (strong, nonatomic) blk_t blk;
@property (copy, nonatomic) NSString *aStringProperty;
@end

@implementation AbstractComputer
- (instancetype)init {
    self = [super init];
    if (self) {
        self.blk = ^{
            NSLog(@"%zd", _aIntegerInstance);
            NSLog(@"%@", self.aStringProperty);
        };
    }
    return self;
}
@end
```

例子中，显而易见，`AbstractComputer` 类持有 `blk`，`clang` 之后，我们看 `blk` 的实现：

``` mm
struct __AbstractComputer__init_block_impl_0 {
	struct __block_impl impl;
	struct __AbstractComputer__init_block_desc_0* Desc;
	AbstractComputer *self;
	__AbstractComputer__init_block_impl_0(void *fp, struct __AbstractComputer__init_block_desc_0 *desc, AbstractComputer *_self, int flags=0) : self(_self) {
		impl.isa = &_NSConcreteStackBlock;
		impl.Flags = flags;
		impl.FuncPtr = fp;
		Desc = desc;
	}
};
```

`blk` 的结构体中出现了 `AbstractComputer *self` 这一变量，而且是默认的修饰类型，所以 `blk` 也强引用了 `AbstractComputer`。所以就造成了循环引用。解决的办法也很简单，最普遍的做法是在 `blk` 定义之前声明 `__weak typeof(self) weakSelf = self;` 之后在 `blk` 内部使用 `weakSelf` 代替 `self`。文件中出现了 `__weak` 修饰符之后，`clang` 过不去，一直报错，所以没法对比了，不过猜想这时 `blk` 的结构体内部的 `weakSelf` 变量，应该是弱引用类型。