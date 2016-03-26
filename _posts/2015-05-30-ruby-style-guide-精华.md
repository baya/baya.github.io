---
layout: post
title: Ruby Style Guide 精华
---

## 使用两个空格作为缩进

~~~ruby
def some_method
  do_something
end
~~~

## 定义方法时，如果有参数则使用括号，如果没有参数则不使用括号

~~~ruby

def m1(a, b)
  do_something
end

def m2
  do_something
end

~~~

## 使用类宏时，不使用括号

~~~ruby

class A
  attr_reader :a
end

~~~


## Classes 和 Modules 的布局

1. 首先 `extend` 和 `include`

2. 其次 inner classes
   比如 `CustomErrorKlass = Class.new(StandardError)`
   
3. 再其次一般常量

4. 再其次属性宏
   比如: `attr_reader :name`
   
5. 再其次其他宏
   比如: `validates :name`
   
6. 再其次公共类方法   

7. 再其次公共实例方法

8. 再其次保护方法

9. 最后私有方法


~~~ruby

class Person
    # extend and include go first
    extend SomeModule
    include AnotherModule

    # inner classes
    CustomErrorKlass = Class.new(StandardError)

    # constants are next
    SOME_CONSTANT = 20

    # afterwards we have attribute macros
    attr_reader :name

    # followed by other macros (if any)
    validates :name

    # public class methods are next in line
    def self.some_method
    end

    # followed by public instance methods
    def some_method
    end

    # protected and private methods are grouped near the end
    protected

    def some_protected_method
    end

    private

    def some_private_method
    end
  end
  
~~~

完整的 Ruby Style Guide 请看这里 [https://github.com/bbatsov/ruby-style-guide](https://github.com/bbatsov/ruby-style-guide) 或者这里 [Ruby Style Guide 注解](http://baya.github.io/2015/03/15/ruby-style-guide-%E6%B3%A8%E8%A7%A3/)
