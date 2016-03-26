---
layout: post
title: 一个简单的 Sql Builder
---

  如果说标题的名字要长，那么此标题可以叫作 **一个非常简单的很不完整的但是有用的 Ruby Sql Builder**, 这个 sql builder
目前只可以处理 where 语句, 那这个 sql builder 解决了我的哪些问题呢? 我参与的项目里有大量的 sql 语句， 这些 sql 语句是这样执行
的,

~~~ruby
  ActiveRecord::Base.connection.select_all sql
~~~

就是这么简单粗暴，一般的情况下，这么做也没有问题，无非是手写 sql, 但是当遇到一些查询的情况时，事情就变得有些棘手了。举个例子，有三
个查询条件: a, b, c, 并且假设它们对应的查询参数分别为 params[:a], params[:b], params[:c], 只有当查询参数不为空时, 才会
触发查询条件, 这样就会有七种情况，

~~~
  where a        # params[:a] 不为空， params[:b] 和 params[:c] 为空
  where b        # params[:b] 不为空, params[:a] 和 params[:c] 为空
  where c        # params[:c] 不为空, params[:b] 和 params[:a] 为空
  where a, b     # params[:a] 和 params[:b] 不为空, params[:c] 为空
  where a, c     # params[:a] 和 params[:c] 不为空, params[:b] 为空
  where b, c     # params[:b] 和 params[:c] 不为空, params[:a] 为空
  where a, b, c  # params[:a], params[:b], params[:c] 都不为空
~~~

首先我们按常规的方法来解决这个问题,

第一步，增加查询条件 a,

~~~ruby
  sql = "where"
  sql << " a = #{params[:a]}" if params[:a].present?
~~~

第二步，增加查询条件 b,

~~~ruby
  if sql == "where"
    sql << " b = #{params[:b]}" if params[:b].present?
  else
    sql << " and b = #{params[:b]}" if params[:b].present?
  end
~~~

第三步, 增加查询条件 c,

~~~ruby
  if sql == "where"
    sql << " b = #{params[:c]}" if params[:b].present?
  else
    sql << " and b = #{params[:c]}" if params[:c].present?
  end
~~~

最后一步,

~~~ruby
  sql = '' if sql == 'where'
~~~

我们看到虽然只有简单的三个查询条件，但我们还是写了 8 个判断语句, 写这么多的判断语句是很繁琐的一件事情, 能不能化繁为简呢? 这时候 sql builder 的作用就来了，代码
很简单，直接贴 SqlBuilder 的代码，

~~~ruby

class SqlBuilder

  class DuplicateStatementError < StandardError; end

  attr_reader :statements_chain, :key_words

  def initialize
    @statements_chain = Hash.new {|hash, key| hash[key] = []}
    @key_words = [:where]
  end

  def where str
    raise DuplicateStatementError.new('duplicate where statement') if statements_chain[:where].length > 0
    statements_chain[:where] << "WHERE #{str}"
    
    self
  end

  def and str
    if statements_chain[:where].empty?
      where str
    else
      statements_chain[:where] << "AND #{str}"
    end

    self
  end

  def or str
    if statements_chain[:where].empty?
      where str
    else
      statements_chain[:where] << "OR #{str}"
    end

    self
  end

  def to_s
    key_words.map {|key_word|
      statements_chain[key_word].join(" ")
    }.join(" ")
  end

end

~~~

现在我们利用 SqlBuilder 来增加查询条件,

~~~ruby
  sql = SqlBuilder.new
  sql.where("a = #{params[:a]}") if params[:a].present?
  sql.and("b = #{params[:b]}") if params[:b].present?
  sql.and("c = #{params[:c]}") if params[:c].present?
  sql.to_s
~~~

此时只需要三个判断语句，就生成了合适的 where 语句了。SqlBuilder 其他有利的两个条件是:

1. 很容易测试, SqlBuilder 本身的算法也容易测试;
2. 能够重复使用

