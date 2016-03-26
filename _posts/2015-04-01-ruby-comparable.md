---
layout: post
title: Ruby Comparable 实践
---

我们有一个 User 模型，此模型有一个 pack\_status 属性, pack\_status 主要用于控制用户的升级权值,
它有 4 个值: init, bronze, silver 和 gold, 其中 bronze 比 init 高级, silver 比 bronze 
高级, gold 比 silver 高级。 我们经常需要对用户之间的 pack_status 进行比较,

~~~ruby
  user1.pack_status > user2.pack_status
  user1.pack_status >= 'bronze'
~~~

如果按照字符串的形式去比较，我们会发现:

~~~
  'silver' > 'init' > 'gold' > 'bronze'
~~~

这种顺序和我们的预期不相符, 这时候我们可以把 `Comparable` 请出来了，故名思义, `Comparable` 是用来构造可以进行比较的类。

我们首先创建一个叫 PackStatus 的类, 然后在此类里面加入 Comparable 模块，

~~~ruby
  class PackStatus
    include Comparable
  end
~~~

此时 PackStatus 还没有什么用处，按照 Comparable 的定义，我们需要实现 `<=>` 方法,

~~~ruby

  class PackStatus

    include Comparable

    attr_reader :name

    STATUS = %w(init bronze silver gold)

    InvalidStatus = Class.new(StandardError)

    def initialize name
	  raise InvalidStatus if !STATUS.include?(name)
	  @name = name
	end

    def <=> other_status
	  if other_status.is_a? String
	    STATUS.index(name) <=> STATUS.index(other_status)
	  else
	    STATUS.index(name) <=> STATUS.index(other_status.name)
	  end
	end
	
  end
  
~~~

我们可以对其进行测试,

~~~ruby
  assert PackStatus.new('init') < PackStatus.new('bronze')
  assert PackStatus.new('init') < 'bronze'
  assert_equal PackStatus.new('init'), 'init'
  
  assert PackStatus.new('bronze') < PackStatus.new('silver')
  assert PackStatus.new('bronze') < 'silver'
  assert_equal PackStatus.new('bronze'), 'bronze'
  
  assert PackStatus.new('silver') < PackStatus.new('gold')
  assert PackStatus.new('silver') < 'gold'
  assert_equal PackStatus.new('silver'), 'silver'
~~~
