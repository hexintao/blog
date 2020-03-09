
## 简介
KVO：即Key-Value-Observer，键值观测模式，它是一种允许当某些对象的特定属性值改变时，及时通知给对象的观察者（其他对象）的机制。  

## 观察者的注册和移除
KVO的大致流程包括：给要监听的属性所属的类添加观察者；接收到属性改变的通知后进行处理；处理完之后解除观察者三大步骤。流程很简单，就像要把大象装进冰箱总共需几步类似。其中，一对一代表对非集合类的属性监听，一对多代表对集合类的属性监听。
### 注册
注册方法：

``` mm
[person addObserver:observer
             forKeyPath:@"age"
                options:NSKeyValueObservingOptionNew
                context:NULL];
```
各个参数的意义很明了。分别依次是：**被观察类的实例对象，观察者类的实例对象，被观察的属性名称，观察选项，额外参数**。
### 注册选项
注册选项包括四个，它们的名字和效果依次是：

* NSKeyValueObservingOptionNew：通知观察者属性发生变化时的新值；
* NSKeyValueObservingOptionOld：通知观察者属性发生变化时之前的旧值；
* NSKeyValueObservingOptionInitial：在注册观察者方法（`addObserver:observer`）未返回时就会开始发送通知，因为被观察者的初始化（initial value）对于观察者来说也是新变化的值；
* NSKeyValueObservingOptionPrior：发送两条通知，也就是当属性值即将要发生变化时，即下边要说的`willChangeValueForKey`触发时间相对应，预先发送给观察者一条通知，待属性值改变之后跟上述三个选项一样还会发出通知。

### 通知的处理
观察者收到通知后，需要通过特定的方法进行处理，样例代码：

``` mm
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary<NSString *,id> *)change
                       context:(void *)context
{
    NSLog(@"%@", change);
    if ([keyPath isEqualToString:@"name"])
    {
		//通知事件的处理
        NSLog(@"%@的名字发生了变化！%@", object, change);
    }
}
```
这个方法一定要在观察者的类中进行重写。  
### 通知信息
通知处理方法中的`change:(NSDictionary<NSString *,id> *)change`是一个字典类型的对象，包含了此次变化的信息，例如`NSKeyValueObservingOptionNew `选项下，一对一的属性发生变化时接收到的变化信息如下：

``` mm
{
    kind = 1;
    new = hexintao;
}
```
第一条的`kind`是`NSKeyValueChange`类型的枚举值，它有如下四个定义：

* NSKeyValueChangeSetting：设置新值，被监听的是一对一的属性或者一对多的属性；
* NSKeyValueChangeInsertion：一对多的属性新插入了一个对象；
* NSKeyValueChangeRemoval：一对多的属性移除了一个对象；
* NSKeyValueChangeReplacement：一对多的属性替换了其中的某个对象。

第二条则会根据`NSKeyValueObservingOptions`观察选项、一对一或者一对多的属性不同而不同。大致也就是显示设置的新值、变化之前的旧值或者是一对多属性的添加、移除、替换的对象和序号（index）等。具体的信息可以`command`+`observeValueForKeyPath`，查看通知处理方法的官方注释，写的非常详细。


### 移除
待属性值发生变化的通知处理完毕之后，我们需要对注册的观察者进行手动解除，解除的方法是：

``` mm
- (void)removeObserver:(NSObject *)anObserver
            forKeyPath:(NSString *)keyPath

- (void)removeObserver:(NSObject *)observer
            forKeyPath:(NSString *)keyPath
               context:(void *)context
```
对没有没有进行监听的属性（keyPath）执行解除操作，会抛出异常。同样，如果对已经注册的监听属性没有执行解除操作，也会抛出异常。
## 自动通知和手动通知
如果按照以上操作步骤执行，则默认使用的是自动通知，即只要对属性值进行重新赋值（不管新值和旧值是否相同），观察者都会收到通知。而在实际应用中，有可能我们想根据自己的需要，待属性值满足我们的条件之后才给观察者发送通知，这时候我们就需要通过手动模式修改发送通知的条件和时间来达到目的了。
### 自动通知
如上述操作，发送通知的时机和条件无法进行修改。

``` mm
//第一次修改可以正常接收到通知
[person setValue:@"hexintao" forKey:@"name"];
//自动通知模式下，接下来这两次依然会接收到通知
[person setValue:@"hexintao" forKey:@"name"];
[person setValue:@"hexintao" forKey:@"name"];
```
当然，实际应用中，连续赋同样的值的情况不多见也不推荐，我们只是为了以此说明自动通知模式下的情况。
### 手动通知
首先需要关闭自动通知，在被观察者的类中重写类方法：

``` mm
+ (BOOL)automaticallyNotifiesObserversOf<Key>
{
    return NO;
}
```
或者是：

``` mm
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key
{
    BOOL automatic = YES;
    if ([key isEqualToString:@"name"])
    {
        automatic = NO;
    }
    else
    {
        automatic = [super automaticallyNotifiesObserversForKey:key];
    }
}
```
后一个方法较复杂，而且有把属性名称（`key`）拼错的风险，所以还是推荐使用第一种方法。  
接下来，要在需要发出通知的地方手动调用两个方法，这个例子中我们就取属性值的`setter`方法：

``` mm
- (void)setName:(NSString *)name
{
    if (name != _name)
    {
        //子类不能重写这两个方法，否则无法完成手动触发KVO
        //通过改变这两个方法的位置，可以自定义KVO触发的条件
        [self willChangeValueForKey:@"name"];
        _name = name;
        [self didChangeValueForKey:@"name"];
    }
}
```
上述两个方法一定要成对调用才会成功发出通知。  
刚才那个例子：

``` mm
//第一次修改可以正常接收到通知
[person setValue:@"hexintao" forKey:@"name"];
//手动通知模式下，接下来这两次不会接收到通知
[person setValue:@"hexintao" forKey:@"name"];
[person setValue:@"hexintao" forKey:@"name"];
```

## 依赖键的注册
有时候，我们监听的某个属性值可能会依赖于其他多个属性，只要其他属性发生了改变都会导致我们监听的属性发生变化，这种就叫做依赖键。例如，在`Person`类中有一个`personInfo`的属性，它返回的是对象的`name`和`age`的组合：

``` mm 
- (NSString *)personInfo
{
    return [NSString stringWithFormat:@"person's name:%@ age:%d", self.name, self.age];
}
```
如果我们对`personInfo`进行监听，则`name`和`age`的变化也会导致`personInfo`发生变化，这时候我们就需要设置依赖键。

``` mm
+ (NSSet *)keyPathsForValuesAffectingPersonInfo
{
    return [NSSet setWithObjects:@"name", @"age", nil];
}
```
然后再对`personInfo`进行注册监听，之后如果我们对`name`或者`age`的值进行修改的时候，观察者就会收到这样的通知：

``` mm
CollectionViewTest[742:17933] {
    kind = 1;
    new = "person's name:hexintao age:0";
}
```
也就是当关联属性中的任何一个发生了变化，我们监听的这个属性就会收到通知，说明其值发生了变化。
## 集合属性的监听
### 集合属性整体还是部分？
一对一属性的监听相对来说比较简单，只要值发生了变化我们收到通知进行处理即可。对于一对多集合类的属性来说，牵扯到是监听整个集合发生的变化还是其中元素的变化？这两种行为都可以通过KVO监听到，不过日常使用来说，我们更倾向于监听后者。  
对于集合属性，正常的添加或删除对象的操作并不能触发KVO，例如`[person.personFriends addObject:@"2in"]`，观察者并不会收到变化通知，不过对于集合属性整体的改变，例如`person.personFriends = [[NSMutableArray alloc]init]`，观察者可以正常收到通知。不过我们重点讨论通过特定的方法监听集合属性中对象的变化。大致分为两种方法：
#### 手动监听
集合属性的手动监听即：在被监听的类中重写一些修改集合元素的方法，之后调用这些方法对属性进行修改就可以触发KVO监听。    
我们可以根据需要实现：

``` mm
//插入对象
- (void)insertObject:(id)object in<Key>AtIndex:(NSUInteger)index
//移除对象
-(void)removeObjectFrom<Key>AtIndex:(NSUInteger)index
```
具体的操作元素的方法可见官方手册：[KVC官方指导](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/AccessorConventions.html#//apple_ref/doc/uid/20002174-178830-BAJEDEFB)  
之后我们通过调用重写后的方法修改集合属性时即可触发KVO。
#### 自动监听
自动监听大致是：通过`mutableArrayValueForKey `方法获得一个可变对象的代理，对其进行修改即可自动触发KVO。而`valueForKey`返回的则是不可变对象。  
使用样例：

``` mm
[[person mutableArrayValueForKey:@"personFriends"] addObject:@"3in"];

// 但是如果是将上述操作赋值给一个可变数组，再调用正常的类似于addObject方法将不会触发KVO监听。
NSMutableArray *friends = [person mutableArrayValueForKey:@"personFriends"];
//不能触发KVO模式
[person.personFriends addObject:@"4in"];  

//这两个方法能触发KVO，但是friends和person.personFriends指向的并不是同一个对象，不过其内容却完全一样，
//对friends操作会影响到person.personFriends的值，反过来也是如此！
[friends insertObject:@"5in" atIndex:0];  //可以触发KVO
[friends removeObjectAtIndex:0];  //可以触发KVO
```
## 参考资料
[Apple KVO官方手册](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html)  
[NSHipster KVO](https://www.google.com.hk/url?sa=t&rct=j&q=&esrc=s&source=web&cd=9&ved=0ahUKEwiZ9tW---jNAhVs5IMKHTI-AwoQFghXMAg&url=http%3a%2f%2fnshipster%2ecn%2fkey-value-observing%2f&usg=AFQjCNFprZJAqSav-dip36Gmq8D2oQVR-A&sig2=DGtyGssCcj_n3oqMM_uQuA)  
[可变对象在KVO中的监听](http://dijkst.github.io/blog/2013/06/21/ke-bian-shu-zu-zai-kvozhong-de-shi-xian/)  
[KVO详解](http://www.cnblogs.com/gatsbywang/p/5210031.html)