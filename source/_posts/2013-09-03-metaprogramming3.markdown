---
layout: post
title: "Ruby元编程笔记 —— （二） 方法"
date: 2013-09-03 15:07
comments: true
categories: ruby
---

![alt text](http://farm3.staticflickr.com/2888/9656919280_8ab725e72a_c.jpg)

* 通过`Object#send()`取代点标记符来调用`MyClass#my_method()`方法：
```ruby
obj = MyClass.new
obj.send(:my_method,arguments)
```
  通过`send()`方法，调用的方法名可以是一个参数，这样可以在代码运行期动态决定调用那种方法，这种技术称为动态派发（`Dynamic Dispatch`）。

<!-- more -->

* 字符串转符号
`String#to_sym()或String#intern()`

  符号转字符串
`Symbol#to_s()或Symbol#id2name()`
* 动态定义方法（`Dynamic Method`）
在类或模块中，使用`define_method()`定义一个方法:
```ruby
class MyClass
  define_method :my_method do |my_arg|
    my_arg * 3
  end
end
```
* 可以用send()调用任何方法，甚至私有方法。
* 匹配括号中正则表达式的字符串会存放在$变量中。`=~`是正则匹配符。
* `method_missing`是`Kernel`的一个实例方法，所有对象均存在。当调用一个不存在的方法时，`Kernel#method_missing()`方法回抛出一个`NoMethodError`进行响应。每一个它处理的消息都带着被调用方法的名字，以及所有调用是传递的参数和块。
* 覆写`method_missing`方法可以调用接收者实际上不存在的方法，这被称为一个幽灵方法
* 用`super`调用被覆写的超类中的同名方法。
* 一个捕获幽灵方法调用病吧他们转发给另一个对象的对象（有时也会在转发前后包装一些自己的逻辑），称为动态代理。
* 幽灵方法不是真正的方法，不会出现在`Object#methods()`获得的方法列表中。
* 仅在必要时才使用幽灵方法，首先是用一个普通方法来实现功能，当确信代码没有问题时，把这些方法重构到`method_missing()`中。
* 当幽灵方法和一个真实方法发生名字冲突时，后者会胜出。
* 应该在代理类中删除绝大多数继承来的方法，这就是白板（`Blank Slate`）类。
* 用`Module#undef_method()`删除所有（包括继承）方法；
  
  用`Module#remove_method()`删除自身定义的方法。