---
layout: post
title: Ruby Enumerable 实践
---

最近一段时间的大部分的编程工作好像是在处理各种各样的数组，比如说通过 SQL 从数据库里查询出了一组数据
对象，然后对这些对象进行统计，排序等。这些对象的结构可以用下面的 Ruby 代码描述:

~~~ruby
  class SomeItem
	attr_reader :gender      # female, male
	attr_reader :first_name
	attr_reader :last_name
	attr_reader :a_points
	attr_reader :b_points
	attr_reader :c_points
  end
~~~

我要处理的数组就是由 `SomeItem` 的实例构成，

~~~ruby
  list = [item1, item2, item3, ...] # item1, item2 等都是 SomeItem 的实例
~~~

现在我们要在 `list` 的基础上做一个报表, 报表的内容包括:

1. a\_points 的总量即: total\_a\_points
2. b\_points 的总量即: total\_b\_points
3. c\_points 的总量即: total\_c\_points
4. 女性的 a\_ponts 的总量即: female\_total\_a\_points
5. 女性的 b\_ponts 的总量即: female\_total\_b\_points
6. 女性的 c\_ponts 的总量即: female\_total\_c\_points
7. 男性的 a\_ponts 的总量即: male\_total\_a\_points
8. 男性的 b\_ponts 的总量即: male\_total\_b\_points
9. 男性的 c\_ponts 的总量即: male\_total\_c\_points

这个任务不是很复杂，我们通过 Ruby 中 `Array` 自带的方法就能实现上面所有的任务，

~~~ruby

# 1
total_a_points = list.reduce(0){|m, e| m + e.a_points }

# 2
total_b_points = list.reduce(0){|m, e| m + e.b_points }

# 3
total_c_points = list.reduce(0){|m, e| m + e.c_points }

female_items = list.select{|e| e.gender == 'female'}

# 4
female_total_a_points = female_items.reduce(0){|m, e| m + e.a_points }

# 5
feamle_total_b_points = female_items.reduce(0){|m, e| m + e.b_points }

# 6
feamle_total_c_points = female_items.reduce(0){|m, e| m + e.c_points }

male_items = list.select {|e| e.gender == 'male' }

# 7
male_total_a_points = male_items.reduce(0){|m, e| m + e.a_points }

# 8
male_total_b_points = male_items.reduce(0){|m, e| m + e.b_points }

# 9
male_total_c_points = male_items.reduce(0){|m, e| m + e.c_points }

~~~

任务完成了，但是完成的不漂亮, 如果我们在多个地方需要上面的这些统计数据, 那么我们需要把这些松散的代码
拷贝粘贴到其他地方。所以我希望能有一个统一的方式去生产，管理，访问这些统计变量。现在我们可以请
`Enumerable` 上场了。


首先我们需要一个集合类, 命名为: `SomeItemCollection`, Collection 表明这是一个集合类。

~~~ruby
  class SomeItemCollection
  end
~~~

现在这个类还没有多大的作用，然后我们将 `Enumerable` 混合进去, 并且为其实现 `each` 方法,

~~~ruby
  class SomeItemCollection
    include Enumerable

    def initialize(list)
	  @list = list
	end

    def each &p
	  @list.each &p
	end
	
  end
~~~

`include Enumerable` 将使 `SomeItemCollection` 拥有 `select`, `reduce` 等一系列方法
简单来说我们可以像操作数组对象一样操作 `SomeItemCollection` 的实例了。

我们注意到方法 `SomeItemCollection#each` 实际是作用在 `@list.each` 上, 可以说我们调用
`SomeItemCollection#select` 方法等同于 `@list.select`。

现在我们可以把统计变量加入到 `SomeItemCollection` 中,

~~~ruby
  class SomeItemCollection
  
    include Enumerable

    attr_reader :total_a_points
	attr_reader :total_b_points
	attr_reader :total_c_points
    attr_reader :female_total_a_points
	attr_reader :female_total_b_points
	attr_reader :female_total_c_points
	attr_reader :male_total_a_points
	attr_reader :male_total_b_points
	attr_reader :male_total_c_points

    def initialize(list)
	  @list = list
	  @total_a_points = reduce(0){|m, e| m + e.a_points }
	  @total_b_points = reduce(0){|m, e| m + e.b_points }
	  @total_c_points = reduce(0){|m, e| m + e.c_points }
	  female_items = select{|e| e.gender == 'female'}
	  @female_total_a_points = female_items.reduce(0){|m, e| m + e.a_points }
	  @female_total_b_points = female_items.reduce(0){|m, e| m + e.b_points }
	  @female_total_c_points = female_items.reduce(0){|m, e| m + e.c_points }
	  male_items = select{|e| e.gender == 'male'}
	  @male_total_a_points = male_items.reduce(0){|m, e| m + e.a_points }
	  @male_total_b_points = male_items.reduce(0){|m, e| m + e.b_points }
	  @male_total_c_points = male_items.reduce(0){|m, e| m + e.c_points }
	end

    def each &p
	  @list.each &p
	end
	
  end
~~~

我们注意到在 `SomeItemCollection` 里可以直接使用 `select`, `reduce` 等方法, 这些方法最终操作的对象
其实是 `@list` 里的对象。

现在我们先看看怎么生产这些统计变量,

~~~ruby
  list = [item1, item2, item3]
  collection = SomeItemCollection.new(list)
~~~

然后访问这些统计变量,

~~~ruby
  collection.total_a_points
  collection.total_b_points
  collection.total_c_points
  collection.female_total_a_points
  collection.female_total_b_points
  collection.female_total_c_points
  collection.male_total_a_points
  collection.male_total_b_points
  collection.male_total_c_points
~~~

小功告成。
