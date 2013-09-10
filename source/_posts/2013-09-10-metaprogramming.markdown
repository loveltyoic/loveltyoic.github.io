---
layout: post
title: "Ruby元编程笔记 —— （五） 编写代码的代码"
date: 2013-09-10 11:14
comments: true
categories: ruby
---
![]()

* 代码只不过是文本而已。
  `Kernel#eval()`方法会执行字符串中的代码，并返回执行结果。
  
<!-- more -->
* `Binding`就是一个用对象表示的完整作用域，通过创建`Binding`对象来捕获并带走当前的作用域。使用`Kernel#binding()`方法创建`Binding`对象：
```ruby
  class MyClass
    def my_method
      @x = 1
      binding
    end
  end
  b = MyClass.new.my_method
```
  对`*eval()`方法家族，可以传递一个`Binding`对象作为额外参数，代码就可以在这个`Binding`对象所携带的作用域中执行：
```ruby
eval "@x", b #=> 1
```
* `TOPLEVEL_BINDING` 表示顶级作用域的`Binding`对象。
* 可以把`Binding`看成是一个比块更纯净的闭包，它们只包含作用域而不包含代码。
* `eval`有三个可选参数
  1.  binding 指定执行上下文
  2.  file 当前处理代码所在文件
  3.  line 当前处理代码所在行号
* `eval()`方法总是需要一个代码字符串作为参数，而`instance_eval()`和`class_eval()`除了块，也可以接受代码字符串作为参数。
* `here`文档

  以`<<`开头，紧跟结束符，直到下一个结束符，在结束符之间定义字符串内容：
```ruby
  s = << END
    字符串内容
  END
```
* 只要能用块就尽量使用块。
* Ruby会自动把不安全的对象——尤其是从外部传入的对象——标记为污染的。通过调用`tainted?()`方法来判断类是不是被污染了。
* 给`$SAFE`全局变量赋值可以设置安全级别。从0（默认）到4共五个安全级别。在任何大于0的级别上，Ruby都会拒绝执行污染的字符串。
* `Kernel#load()`和`Kernel#require()`,接受文件名作为参数，并执行该文件中的代码。
* `Object#instance_variable_get()`和`Object#instance_variable_set()`分别读/写一个实例变量。
* 钩子方法`Hook Method`

  `inherited()`方法是`Class`的一个实例方法，当类被继承时，Ruby会调用这个方法。像`Class#inherited()`这样的方法称为钩子方法。
* 更多钩子方法
```
Module#included()
MOdule#extend_object()
Module#method_added() method_removed() method_undefined()
单件方法钩子：
Kernel#singleton_method_added singleton_method_removed() singleton_method_undefined
```
* 可以把一个普通的方法使用环绕别名变成一个钩子方法。
* 类扩展混入`Class Extension Mixin`
  1.  定义一个模块，如`MyMixin`
  2.  在`MyMixin`中定义一个内部模块，如`ClassMethods`，并给它定义一些方法。
  3.  覆写`MyMixin#included()`方法，用`ClassMethods`扩展包含者(使用`extend()`)
```ruby
    module MyMixin
      def self.included(base)
        base.extend(ClassMethods)
      end
      module ClassMethods
        def my_method
        end
      end
    end
```
  可以在`MyMixin`中直接定义一些额外的方法，将成为包含者的实例方法。
