---
title: KVC 背后的执行流程
date: 2018-03-19 19:12:21
categories: iOS
tags: [iOS]
---
## 引子
相当于是[官方文档](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueCoding/SearchImplementation.html#//apple_ref/doc/uid/20000955-CJBBBFFA)的一个翻译，做一个记录。  
<!-- more -->
### Basic Getter
`valueForKey:` 方法的默认实现，系统会对消息接收者对象，执行这 5 步：

1. 查找简单的取值方法，`Xcode 9` 环境下，按照优先级排列为：`get<Key>`, `is<Key>`,  `_<key>`，如果找到了直接跳转至步骤 5，否则执行第步骤 2。
2. 查找形如 `countOf<Key>`，`objectIn<Key>AtIndex:`，`<key>AtIndexes:` 方法。如果第一个方法及后边两个方法中的任一个实现了，系统就会以 `NSArray` 为父类，动态生成一个类型为 `NSKeyValueArray` 的集合类对象，并调用上述实现的方法，将结果直接返回，如果对象还实现了形如 `get<Key>:range:` 的方法，系统也会在必要的时候自动调用。从整体效果来看，系统会让消息接收者对象该属性对外表现的像是 `NSArray` 类型，虽然它有可能并不是。如果上述操作不成功就接着往下执行。
3. 如果上述两步失败，系统会查找形如 `countOf<Key>`，`enumeratorOf<Key>`，`memberOf<Key>:` 的方法实现，如果这三个方法都有实现，系统会自动生成一个另一种类型的集合类对象，这里我尝试了一下没有成功，所以不知道具体类型是什么，不过猜想应该是以 `NSSet` 为父类。同第2步，也会调用上述实现的方法将结果返回，从整体效果来看，系统会让消息接收者对象该属性对外表现的像是 `NSSet` 类型，虽然它有可能并不是。如果上述操作不成功就接着往下执行。
4. 如果上述操作都失败，而且消息接收者的类方法：`- (BOOL)accessInstanceVariablesDirectly` 返回 `YES`，系统会按照顺序查找以下实例变量：`_<key>`, `_is<Key>`, `<key>`, `is<Key>`，如果找到就直接获取实例变量的值并转至步骤 5 执行，否则转至步骤 6。
5. 如果获取到的变量的值所指向的是对象，直接将变量的值返回，外界直接可以获取到对象。如果变量的值是 `NSNumber` 支持的数值类型，包装成 `NSNumber` 类型对象并返回。如果不是 `NSNumber` 支持的类型，包装成 `NSValue` 对象并返回。
6. 如果上述步骤都失败了，调用 `valueForUndefinedKey:` 方法抛出异常，形如 `this class is not key value coding-compliant for the key ***`，不过 `NSObject` 的子类可以通过重载并根据 `key` 做一些特定处理。

### Basic Setter
`setValue:forKey:` 方法的默认实现，系统会对消息接收者对象，执行这 3 步：

1. 按照以下顺序，查找形如 `set<Key>:`，`_set<Key>` 的方法。如果找到了，调用并返回。
2. 如果消息接收者的类方法：`- (BOOL)accessInstanceVariablesDirectly` 返回 `YES`，按照以下顺序查找实例变量：`_<key>`, `_is<Key>`, `<key>`, `is<Key>`。如果找到了，赋值并返回。
3. 如果前两步失败，调用 `setValue:forUndefinedKey:` 方法抛出异常，形如 `this class is not key value coding-compliant for the key ***`，不过 `NSObject` 的子类可以通过重载并根据 `key` 做一些特定处理


### Mutable Arrays
即方法 `mutableArrayValueForKey:` 的默认实现。系统会对消息接收者对象，执行这 4 步：

1. 两对方法：`insertObject:in<Key>AtIndex:` 和 `removeObjectFrom<Key>AtIndex:`，以及 `insert<Key>:atIndexes:` 和 `remove<Key>AtIndexes:`，如果至少实现了一个插入新对象和一个移除对象的方法，系统会生成一个可以响应 `NSMutableArray` 方法的代理对象，并且调用上述实现的方法。如果消息接收者对象还实现了 `replaceObjectIn<Key>AtIndex:withObject:` 或者 `replace<Key>AtIndexes:with<Key>:`，那就更好不过了。
2. 如果第 1 步失败，系统会查找形如 `set<Key>:` 的方法，如果找到了，会生成并返回一个代理对象，该代理对象会通过调用实现的 `set<Key>:` 方法对 `mutableArrayValueForKey:` 做出响应。需要注意的是，这一步没有第 1 步效率高，因为系统会通过不断调用 `set<Key>:` 来创建一个新的集合对象而不是在原有对象的基础上进行修改。
3. 如果消息接收者的类方法：`- (BOOL)accessInstanceVariablesDirectly` 返回 `YES`，系统会按以下顺序查找实例变量：`_<key>`, `<key>`，如果找到了，返回一个代理对象，该对象会把所有 `NSMutableArray` 的方法调用转发给该实例变量（该实例变量通常都是 `NSMutableArray` 类型或者其子类）。
4. 如果上述操作都失败，返回一个可变集合类型的代理对象，一旦后续有任何 `NSMutableArray` 的方法调用，系统会自动调用 `setValue:forUndefinedKey:` 方法抛出异常。同样子类也可以重载该方法，实现自定义处理。

后续还有 `Mutable Ordered Sets`，即 `mutableOrderedSetValueForKey:` 方法的实现，以及 `Mutable Sets`，即 `mutableSetValueForKey:` 方法的实现两种情况，可以看[官方文档](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueCoding/SearchImplementation.html#//apple_ref/doc/uid/20000955-CJBBBFFA)，这里不再翻译解释。