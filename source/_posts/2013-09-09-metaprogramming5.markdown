---
layout: post
title: "Ruby元编程笔记 —— （四） 类定义"
date: 2013-09-09 09:14
comments: true
categories: ruby
---
![](http://farm3.staticflickr.com/2887/9694272518_88a646c15b_o.jpg)

* 类只是一个增强的模块。
* 在类（或模块）定义时，类本身充当了当前对象`self`的角色。

<!-- more -->
* 总是有一个当前类存在，当定义一个方法时，该方法将成为当前类的一个实例方法。
* 使用`class_eval()`可以修改当前类。会在一个已存在类的上下文中执行一个块。
* `class`关键字会打开一个新的作用域，而`class_eval()`使用扁平作用域。
* 如果想打开一个类定义并且用`def`关键字定义方法，则可以选择`class_eval()`方法。
* 在类定义中，当前对象`self`就是正在定义的类，当前类就是`self`。如果有一个类的引用，可以用`class_eval()`或`module_eval()`打开这个类
* 所有的实例变量都属于当前对象`self`
* 在类充当`self`时定义的变量是类实例变量，仅仅可以被类本身访问，而不能被类的实例或子类所访问。
* `@@`开头的是类变量，避免使用。
* `Class#new()`方法可以接受一个参数（所建新类的超类）以及一个块，创建一个匿名类。
* 类名不过是一个常量而已，可以给匿名类赋值，相当于给类命名。
* 只针对单个对象生效的方法，称为单件方法。
* 类方法的实质就是：他们是一个类的单件方法。
* 进入该对象的`eigenclass`的作用域。
```ruby
  class << an_object 
```
* `instance_eval()`方法也会修改当前类：它会将当前类修改为接收者的`eigenclass`，因此可以在其中定义单件方法。
* `instance_eval()`方法的标准含义是：我想修改`self`。
* 可以在子类中调用父类的类方法。
* 一个对象的`eigenclass`的超类是这个对象的类；一个类的`eigenclass`的超类是这个类的超类的`eigenclass`。
* 当类包含模块时，他获得的是该模块的实例方法——而不是类方法。
* 在类的`eigenclass`中包含模块，类的`eigenclass`的实例方法就成了类方法，这种技术称为类扩展`Class Extension`。更一般的，可以把模块混合到对象的`eigenclass`，称为对象扩展`Object Extension`。
* `Object#extend()`方法可以简化这种扩展
```ruby
  obj.extend MyModule #对象扩展
  class MyClass   #类扩展
    extend MyModule
  end
```
* 使用`alias`关键字，可以给方法取别名。
用法：
```ruby
  alias :新方法名 :原方法名
  alias_method :新方法名, :原方法名  #注意：方法版本，带逗号
```
* 重定义一个方法时，并不真正修改这个方法，仍然可以通过别名访问原始的方法。
* 环绕别名`Around Alias` 技巧：
  1.  给方法定义一个别名
  2.  重定义这个方法
  3.  在重定义方法中调用原始的方法
* 类宏`Class Macro`只是普通的类方法，可以用在类定义而已。
