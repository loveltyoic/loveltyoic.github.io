---
layout: post
title: "ActiveRecord中的Callback浅析"
date: 2014-11-21 09:13
comments: true
categories: Rails
---

在使用Rails或者说ActiveRecord的过程中，我们都会用一些callback来使我们的代码更紧凑简洁。
我们只需要知道用法就可以了，但是你是否也会好奇，这些神奇的callback是怎么实现的呢?
下面通过源码加注释的方式粗略的说明一下我的探究过程。

首先定义User类，并给他添加几个callback，用了3种形式
```ruby
class ShowMyself
  def before_validation(model)
    puts 'def callback by object'
  end
end

class User < ActiveRecord::Base
  before_create :say_hello
  def say_hello
    puts 'def callback by method'
  end

  before_validation ShowMyself.new

  before_save do 
    puts 'def callback by block'
  end

end
``` 
那么问题来了，`before_validation ShowMyself.new`是怎么工作的呢？

- 从`before_validation`开始查找

```ruby
def before_validation(*args, &block)
  options = args.last
  if options.is_a?(Hash) && options[:on]
    options[:if] = Array(options[:if])
    options[:on] = Array(options[:on])
    options[:if].unshift lambda { |o|
      options[:on].include? o.validation_context
    }
  end
  set_callback(:validation, :before, *args, &block)
  # set_callback(:validation, :before, ShowMyself.new)
end
```
- 进入`set_callback(:validation, :before, ShowMyself.new)`

```ruby
def set_callback(name, *filter_list, &block)

  # name = :validation
  # filter_list = [:before, ShowMyself.new]

  type, filters, options = normalize_callback_params(filter_list, block)

  # 调用normalize_callback_params，得到如下返回

  # type = :before
  # filters = [ShowMyself.new]
  # options = {}

  self_chain = get_callbacks name

  # get_callbacks 得到 CallbackChain类型的实例, 
  # 在console里面通过User._validation_callbacks就可以得到它了
  # 其他的如_save_callbacks, _create_callbacks都一样可以看到
  
  mapped = filters.map do |filter|
    Callback.build(self_chain, filter, type, options)
    # Callback.build(self_chain, ShowMyself.new, :before, {})
    # 这里可以暂时知道它是一个Callback实例就可以了，后面会有说明
  end
  # mapped = [以ShowMyself.new为filter的Callback实例]

  # 记住name = :validation
  __update_callbacks(name) do |target, chain|
    # 在当前block中
    # target = User
    # chain = User.get_callbacks :validation

    options[:prepend] ? chain.prepend(*mapped) : chain.append(*mapped)
    # 把mapped加入到回调链中

    target.set_callbacks name, chain
    # 即User._validation_callbacks= chain
    # 执行到这里，ShowMyself.new就添加成功了
    # 可以执行User._validation_callbacks.first.filter看看
    # 应该看到类似如下结果
    # <ShowMyself:0x007ff556309938>
    # 正是ShowMyself的实例
  end
end
```
```ruby
# File activesupport/lib/active_support/callbacks.rb, line 740
def set_callbacks(name, callbacks)
  send "_#{name}_callbacks=", callbacks
end
```

- `normalize_callback_params`
```ruby  
def normalize_callback_params(filters, block) # :nodoc:
  type = CALLBACK_FILTER_TYPES.include?(filters.first) ? filters.shift : :before
  # type = filters中(即传过来的[:before, ShowMyself.new])第一个元素 = :before

  options = filters.extract_options!
  filters.unshift(block) if block
  [type, filters, options.dup]
end
```  
- `__update_callbacks`
```ruby
def __update_callbacks(name) #:nodoc:
  ([self] + ActiveSupport::DescendantsTracker.descendants(self)).reverse_each do |target|
    #当前类User以及他的子类都需要新增这个callback, 即回调也是会继承的
    chain = target.get_callbacks name
    yield target, chain.dup
  end
end
```

以上是定义callback的过程，那么callback是怎么执行的呢？

当执行user.valid?的时候, 其实调用了`run_validations!`

```ruby
  def run_validations! #:nodoc:
    _run_validation_callbacks { super }
  end
```
then

```ruby
def _run_#{name}_callbacks(&block)
  _run_callbacks(_#{name}_callbacks, &block)
  # _run_callbacks(_validation_callbacks, &block)
end
```
then
```ruby
def _run_callbacks(callbacks, &block)
  # 这里的callbacks就是User._validation_callbacks
  if callbacks.empty?
    block.call if block
  else
    runner = callbacks.compile
    # compile 是CallbackChain中定义的实例方法,返回一个lambda

    # runner就是如下形式的lambda
    # lambda { |env|
    #   user_callback.call env.target, env.value
    #   next_callback.call env
    # }

    # user_callback也是一个lambda
    # lambda { |target, _, &blk|
    #    filter.public_send method_to_call, target, &blk
    # }

    # 当前的self就是User实例，即user
    # Environment从后面可以知道就是一个Struct
    e = Filters::Environment.new(self, false, nil, block)
    runner.call(e).value
  end
end
```
- CallbackChain的定义(节选)

```ruby
class CallbackChain #:nodoc:#
  # 一个可遍历的链
  include Enumerable

  attr_reader :name, :config

  def compile
    @callbacks || @mutex.synchronize do
      @callbacks ||= @chain.reverse.inject(Filters::ENDING) do |chain, callback|
        callback.apply chain
        #进入Callback::apply
        #此时
        #callback.name = :validation
        #callback.kind = :before
      end
    end
  end

end
```

```ruby
class Callback #:nodoc:#
  def self.build(chain, filter, kind, options)
    new chain.name, filter, kind, options, chain.config
    # 此时
    # chain.name = :validation
    # filter = ShowMyself.new
  end

  attr_accessor :kind, :name
  attr_reader :chain_config

  def initialize(name, filter, kind, options, chain_config)
    @chain_config  = chain_config
    @name    = name
    @kind    = kind
    @filter  = filter
    @key     = compute_identifier filter
    @if      = Array(options[:if])
    @unless  = Array(options[:unless])
  end

  def apply(next_callback)

    user_conditions = conditions_lambdas
    user_callback = make_lambda @filter
    # 当前的@filter 就是ShowMyself.new
    # make_lambda很关键！定义在下面

    case kind
    when :before
      Filters::Before.build(next_callback, user_callback, user_conditions, chain_config, @filter)
    when :after
      Filters::After.build(next_callback, user_callback, user_conditions, chain_config)
    when :around
      Filters::Around.build(next_callback, user_callback, user_conditions, chain_config)
    end
  end

  def make_lambda(filter)
    case filter
    # 当filter是一个方法的时候, 如 :say_hello, 执行user.send :say_hello
    when Symbol
      lambda { |target, _, &blk| target.send filter, &blk }
    # 可以用string来定义写filter，用eval执行
    when String
      l = eval "lambda { |value| #{filter} }"
      lambda { |target, value| target.instance_exec(value, &l) }
    when Conditionals::Value then filter
    when ::Proc
      if filter.arity > 1
        return lambda { |target, _, &block|
          raise ArgumentError unless block
          target.instance_exec(target, block, &filter)
        }
      end

      if filter.arity <= 0
        lambda { |target, _| target.instance_exec(&filter) }
      else
        lambda { |target, _| target.instance_exec(target, &filter) }
      end
    else
      # ShowMyself.new的情况
      # scope=>[:kind, :name]
      scopes = Array(chain_config[:scope])
      method_to_call = scopes.map{ |s| public_send(s) }.join("_")
      # method_to_call = "before_validation"

      lambda { |target, _, &blk|
        filter.public_send method_to_call, target, &blk
        # ShowMyself.new.public_send "before_validation", user
        # 看这个lambda, 再回头看看ShowMyself里
        # def before_validation(model)
        #   p model.inspect
        #   puts 'def callback by class'
        # end
        # 转了一圈，终于真相大白了
      }
    end
  end

end
```

## 附录

callback的主要实现都在ActiveSupport::Callbacks中

```ruby
# activrecord/lib/active_record/callbacks.rb

module ActiveRecord
  module Base
    include Callbacks
  end

  module Callbacks
    extend ActiveSupport::Concern
    module ClassMethods
      include ActiveModel::Callbacks
    end
  end
  # 上面翻译过来就是 ActiveRecord::Base.extend ActiveModel::Callbacks
end
```

```ruby
# activemodel/lib/active_model/callbacks.rb

moudle ActiveModel
  module Callbacks
    def self.extended(base) #:nodoc:
      base.class_eval do
        include ActiveSupport::Callbacks
        # 结果就是 ActiveRecord::Base.include ActiveSupport::Callbacks
      end
    end
  end
end
``` 