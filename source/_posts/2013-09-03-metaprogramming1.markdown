---
layout: post
title: "Ruby元编程笔记 -- （零） 引言"
date: 2013-09-03 10:20
comments: true
categories: ruby
---
![alt text](http://farm9.staticflickr.com/8446/7758810948_ce36812ac9_c.jpg)

*  元编程是编写在运行时操纵语言构件的代码。
*  内省`（introspection）`
```ruby
class Greating
  ...
end
my_obj = Greeting.new
my_obj.class #=> Greeting 查看对象的类
my_obj.class.instance_methods(false) #=> 查看类的实例方法，false表示只返回自身定义的，不包括继承方法
my_obj.instance_variables #=> 查看对象的实例变量
```