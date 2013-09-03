---
layout: post
title: "Ruby元编程笔记 —— （一） 对象模型"
date: 2013-09-03 10:27
comments: true
categories: ruby
---
![alt text](http://farm9.staticflickr.com/8465/8079198551_48f29d2544_c.jpg)

*  打开类`（Open Class）`

  在已有类中动态的添加方法。

* 通过`Object#methods`查看对象的方法，例如数组对象，可以`[].methods.grep /正则表达式/`查看指定匹配的所有方法名。

*  一个对象的实例变量存在于对象本身，而一个对象的方法存在于对象自身的类，称为类的实例方法。

<!-- more -->

*  一个类只不过是一个增强的`Module`， 增加了三个方法 —— `new(),allocate(),superclass()`，`superclass`返回超类。

*  类自身也是对象，所有的类最终都继承自`Object`。

*  可以将`Module`作为命名空间，例如
```ruby
Module Book
  class Fiction
    ...
```
则可以通过`Book::Fiction`来引用类，有效避免命名冲突。

*  接收者`(receiver)`就是调用方法时所用的对象，祖先链`(ancestors)`就是从一个类上溯到其顶级超类的整个类路径，可能包括模块。调用`ancestors()`方法来获得一个类的祖先链：`MyClass.ancestors`

*  类中`include`一个模块时，会将这个模块封装成一个匿名类插入祖先链，在类的正上方，但是通过`superclass`并不会显示这个匿名类。

*  查看类的私有实例方法
```ruby
Class.private_instance_methods
```

*  `self`的角色通常是由最后一个接收到方法调用的对象来充当，不过，在类和模块定义中（并且在任何方法定义之外），`self`的角色由这个类或模块担任：
```ruby
class MyClass
  self  #=> MyClass
end
```

* 当调用一个方法时，`Ruby`首先向右一步找到接收者的类，然后一直向上查找祖先链，直到找到该方法，或到达链顶端。

* 当一个类中包含多个`Module`时，最后一个`include`的`Module`最接近这个类，其中的方法被最先找到。