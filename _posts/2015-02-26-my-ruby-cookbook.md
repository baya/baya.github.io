---
layout: post
title: 我的 Ruby Cookbook
---

## CSV To Excel

csv\_to\_excel.rb.private 内容:

~~~ruby

#!/usr/bin/env ruby

require 'csv'
require 'axlsx'

p = Axlsx::Package.new
book = p.workbook

book.add_worksheet(name: 'reports') do |sheet|
  CSV.foreach ARGV[0] do |row|
    sheet.add_row row
  end
end

fname = [File.basename(ARGV[0], File.extname(ARGV[0])), 'xlsx'].join('.')

p.serialize fname

~~~

~~~bash
$ chmod +x ./csv_to_excel.rb.private
$ ./csv_to_excel.rb.private ~/mydir/log/shipping/send/ship_sku.csv
~~~
通过 ship\_sku.csv 生成一份 ship\_sku.xlsx 的文件


## MD5

~~~ruby
require 'digest'
Digest::MD5.hexdigest 'test'
~~~

## Gem

### gem 升级到特定版本

~~~bash
gem update --system 1.3.7
~~~

### 写 gem

1. 生成gem文件
- bundle gem new_gem
2. 编码
new_gem.gemspec
3. build
gem build new_gem.gemspec
4. 发布
gem push new_gem-0.0.1.gem


## Case

### case on lambda

~~~ruby
require 'prime' # prime 是质数的意思

n = rand(1..10)
puts n

case n
when lambda(&:prime?)          # lambda(&:prime?).call(n) or lambda(&:prime?) === n
  puts "This number is prime"
when lambda(&:even?)           # lambda(&:even?).call(n) or lambda(&:even?) === n
  puts "This number is even"
else
  puts "This number is odd"
end

~~~

### case on ranges

~~~ruby

age = rand(1..100)
puts age

case age
when -Float::INFINITY..20       # (-Float::INFINITY..20).include? age or (-Float::INFINITY..20) === age
  puts "You'r too young"
when 21..64                     # (21..64).include? age or (21..64) === age
  puts "You are the right age"
when 65..Float::INFINITY        # (65..Float::INFINITY).include? age or (65..Float::INFINITY) === age
  puts "You're too old"
end

~~~

### case on class

~~~ruby
v = ['foo', 1, [1,2,3]][rand(3)]
puts v

case v
when String; puts "string"   # v.is_a? String or String === v
when Integer; puts "integer" # v.is_a? Integer or Integer === v
when Array; puts "array"     # v.is_a? Array or Array === v
end
~~~

### case on custom define object

~~~ruby
class StarLevel
  attr_reader :level
  
  def initialize level
    @level = level
  end

  def === star_string
    [level, '*'].join == star_string
  end
  
end

star1 = StarLevel.new 1
star2 = StarLevel.new 2
star3 = StarLevel.new 3
star4 = StarLevel.new 4
star5 = StarLevel.new 5

star_level = "2*"

case star_level
when star1; puts "1 star" # star1 === star_level
when star2; puts "2 star" # star2 === star_level
when star3; puts "3 star" # star3 === star_level
when star4; puts "4 star" # star4 === star_level
when star5; puts "5 star" # star5 === star_level
else
  puts "not defined level star"
end
~~~
## inject a Symbol

~~~ruby
 (1..10).inject(:*) # 等价于 (1..10).inject(1) {|m, i| m * i}
~~~

模拟实现如下,

~~~ruby
  class Range

    def symbol_inject init_value = 1, sym
	  m = init_value
	  p = sym.to_proc

      # m 是 p 的 receiver
	  each {|i| m = p.call m, i }
	  
	  m
	end
	
  end

  (1..10).symbol_inject(:*)
  (1..10).symbol_inject(2, :*)
  
~~~

## Sprintf

~~~ruby

'%d %d' % [20, 10]
# => '20 10'

sprintf('%d %d', 20, 10)
# => '20 10'

'%d' % 20
# => '20'

'%d %d' % [20, 10]
# => '20 10'

sprintf('%d %d', 20)
# => '20'

sprintf('%d %d', 20, 10)

~~~

### Time strftime

~~~text
%Y%m%d           => 20071119                  Calendar date (basic)
%F               => 2007-11-19                Calendar date (extended)
%Y-%m            => 2007-11                   Calendar date, reduced accuracy, specific month
%Y               => 2007                      Calendar date, reduced accuracy, specific year
%C               => 20                        Calendar date, reduced accuracy, specific century
%Y%j             => 2007323                   Ordinal date (basic)
%Y-%j            => 2007-323                  Ordinal date (extended)
%GW%V%u          => 2007W471                  Week date (basic)
%G-W%V-%u        => 2007-W47-1                Week date (extended)
%GW%V            => 2007W47                   Week date, reduced accuracy, specific week (basic)
%G-W%V           => 2007-W47                  Week date, reduced accuracy, specific week (extended)
%H%M%S           => 083748                    Local time (basic)
%T               => 08:37:48                  Local time (extended)
%H%M             => 0837                      Local time, reduced accuracy, specific minute (basic)
%H:%M            => 08:37                     Local time, reduced accuracy, specific minute (extended)
%H               => 08                        Local time, reduced accuracy, specific hour
%H%M%S,%L        => 083748,000                Local time with decimal fraction, comma as decimal sign (basic)
%T,%L            => 08:37:48,000              Local time with decimal fraction, comma as decimal sign (extended)
%H%M%S.%L        => 083748.000                Local time with decimal fraction, full stop as decimal sign (basic)
%T.%L            => 08:37:48.000              Local time with decimal fraction, full stop as decimal sign (extended)
%H%M%S%z         => 083748-0600               Local time and the difference from UTC (basic)
%T%:z            => 08:37:48-06:00            Local time and the difference from UTC (extended)
%Y%m%dT%H%M%S%z  => 20071119T083748-0600      Date and time of day for calendar date (basic)
%FT%T%:z         => 2007-11-19T08:37:48-06:00 Date and time of day for calendar date (extended)
%Y%jT%H%M%S%z    => 2007323T083748-0600       Date and time of day for ordinal date (basic)
%Y-%jT%T%:z      => 2007-323T08:37:48-06:00   Date and time of day for ordinal date (extended)
%GW%V%uT%H%M%S%z => 2007W471T083748-0600      Date and time of day for week date (basic)
%G-W%V-%uT%T%:z  => 2007-W47-1T08:37:48-06:00 Date and time of day for week date (extended)
%Y%m%dT%H%M      => 20071119T0837             Calendar date and local time (basic)
%FT%R            => 2007-11-19T08:37          Calendar date and local time (extended)
%Y%jT%H%MZ       => 2007323T0837Z             Ordinal date and UTC of day (basic)
%Y-%jT%RZ        => 2007-323T08:37Z           Ordinal date and UTC of day (extended)
%GW%V%uT%H%M%z   => 2007W471T0837-0600        Week date and local time and difference from UTC (basic)
%G-W%V-%uT%R%:z  => 2007-W47-1T08:37-06:00    Week date and local time and difference from UTC (extended)

~~~


## File

### 获取文件扩展名

~~~ruby
File.extname(filename)
filename.chomp(File.extname(filename))
~~~

### 按行读取文件

- 参考: http://stackoverflow.com/questions/5545068/what-are-all-the-common-ways-to-read-a-file-in-ruby

~~~ruby
File.open file_path, 'r' do |file|
  file.each_line do |line|
    puts line
  end
end
~~~

### 创建文件目录

~~~ruby
FileUtils.mkdir_p(report_dir) if !File.exists?(report_dir)
~~~

## Lambda

### 常见形式

~~~ruby
minimal = -> {p :called}
minimal.call

loaded = ->(arg, default = :default, &block) {p [arg, default, block]}
loaded.call(:arg) {:block}
~~~

## Super

### Pass No Block Up

~~~ruby
class Parent
  def show_block(&block)
    p block
  end
end

class Children < Parent
  def show_block
    super(&nil)
  end
end

Children.new.show_block {:block}  #=> nil

class Children < Parent
  def show_block
    super
  end
end

Children.new.show_block {:block} #=> #<Proc:0x007f8f23a02b70@(irb):18>

class Children < Parent
  def show_block
    super()
  end
end

Children.new.show_block {:block} #=> #<Proc:0x007f8f23a02b70@(irb):18>

~~~

### Pass No Args Up

~~~ruby
class Parent
  def show_args(*args)
    p args
  end
end

class Children
  def show_args(a, b, c)
    super()
  end
end

Children.new.show_args(:a, :b, :c)

~~~

## Unused Variable

~~~ruby
{:a => 'b', :c => 'd'}.each {|k, _| p k}
~~~

## 数字精确到小数

~~~ruby
1.234567.round(2)  #=> 1.23
'%5.2f' % 123.593  #=> '123.59'
~~~


## 动态定义方法

~~~ruby

# 1. 支持1.9
define_method :m do |a = false|
end

# 2. 1.8， 1.9
class_eval <<-EVAL
  def #{"m"}(a = false)
  end
EVAL

# 3. 1.8， 1.9
define_method :m do |*args|
  a = args.first
end

# 4, 来自于 http://api.rubyonrails.org/classes/Module.html

method_def = [
        "def #{method_prefix}#{method}(#{definition})",  # def customer_name(*args, &block)
        " _ = #{to}",                                    #   _ = client
        "  _.#{method}(#{definition})",                  #   _.name(*args, &block)
        "rescue NoMethodError => e",                     # rescue NoMethodError => e
        "  if _.nil? && e.name == :#{method}",           #   if _.nil? && e.name == :name
        "    #{exception}",                              #     # add helpful message to the exception
        "  else",                                        #   else
        "    raise",                                     #     raise
        "  end",                                         #   end
        "end"                                            # end
      ].join ';'
    end

    module_eval(method_def, file, line)


~~~

## Module

### Module Function

~~~ruby

module Mod
  def one
    "This is one"
  end

  module_function :one
end

module Mod
  def tow
    "This is tow"
  end

  extend self
end

module Mode

  def self.three
    "This is three"
  end
  
end

~~~

## 自定义异常带消息

~~~ruby

class MyError < StandardError
  def initialize(msg = "You've triggered a MyError")
    super(msg)
  end
end

~~~

## 迭代器

~~~ruby

def bundle(*enumerables)
  enumerators = enumerables.map { |e| e.to_enum }
  loop { yield enumerators.map { |e| e.next} }
end
a,b,c, d = [1,2,3], 4..6, 'a'..'e', [6, 9, 10, 12, 13]
bundle(a,b,c, d) { |x| print x}

~~~

## gbk 转换 utf8

~~~ruby
resp_msg = resp_msg.encode('utf-8', 'gbk', {:invalid => :replace, :undef => :replace, :replace => '?'})
~~~

## Use Variable inside Regex

~~~ruby
ticket_log.ticketid.sub(/^#{config['agentid']}/, '')
~~~

## 生成订单号算法

~~~ruby
def gen_order_no
  rs = lambda {
    r = []
    1.upto(5) do
      r << (rand(122 - 97) + 97).chr
    end
    r.join
   }
  now = Time.now.strftime("%y%m%d%H%M%S")
  r  = [Time.now.to_f.to_s.split(".")[1]].pack('m')
    .gsub("\n",rs.call)
    .gsub("=",rs.call)
    .gsub("/",rs.call)
    .gsub("+",rs.call)
  "#{now}#{r}"[0,20]
end

~~~

## Eventmachine

### 定时任务

~~~ruby

require 'eventmachine'

EM.run {

  EM.add_periodic_timer(1) do
    sleep 2
    puts Time.now
  end

}

~~~


## ruby json symbolize_keys

~~~ruby

j = "{\"one\":1,\"two\":\"two\"}"
h2 = ActiveSupport::JSON.decode(j).symbolize_keys

h2[:one] #=> 1

# 或者

original_hash = {:info => [{:from => "Ryan Bates", :message => "sup bra", :time => "04:35 AM"}]}
serialized = JSON.generate(original_hash)
new_hash = JSON.parse(serialized, {:symbolize_names => true})

new_hash[:info]

~~~

## ActiveSupport to_datetime.in_time_zone

~~~ruby
"2013-01-14 14:38".to_datetime.in_time_zone
"2013-01-14 14:38".to_datetime.in_time_zone("Prague")
~~~

## Rake

### ruby 环境中加载(非 rails 环境) rake task

~~~ruby
Dir.glob('lib/tasks/**/*.rake').each {|r| import r }
~~~

### rake 获取环境变量 ENV

~~~text
$ rake mytask var=foo
and access those from you rake file as ENV variables like such:
p ENV['var'] # => "foo"
~~~

## bundle gemfile git

~~~ruby
gem 'rails', :git => 'https://github.com/rails/rails.git', :ref => '4aded'
gem 'rails', :git => 'https://github.com/rails/rails.git', :branch => '2-3-stable'
gem 'rails', :git => 'https://github.com/rails/rails.git', :tag => 'v2.3.5'
~~~

## 大块代码的赋值

~~~ruby
a ||= \
begin
 code here ...
end
~~~

## keyword argument and options 具名参数

~~~ruby

def foo(str: "foo", num: 424242, **options)
  [str, num, options]
end

foo #=> ['foo', 424242, {}]
foo(check: true) # => ['foo', 424242, {check: true}]

# 类似于
def foo(str: "foo", num: 424242, options: {})
  [str, num, options]
end

# 但是不能写成:
def foo(str: "foo", num: 424242, options={})
  [str, num, options]
end

def foo(str: "foo", num: 424242, *options)
  [str, num, options]
end

~~~

## 获取方法的参数

~~~ruby
def foo(x, y)
end

method(:foo).parameters # => [[:req, :x], [:req, :y]]

args = method(__method__).parameters.map { |arg| arg[1] }
~~~

## God 脚本监控

- http://godrb.com/

### 有环境变量的例子

~~~ruby
God.watch do |w|
  ...

  w.start = lambda { ENV['APACHE'] ? `apachectl -k graceful` : `lighttpd restart` }

  ...
end

God.watch do |w|
  ...

  w.env = { 'RAILS_ROOT' => "/var/www/myapp",
            'RAILS_ENV' => "production" }

  ...
end

~~~

### 例子

~~~ruby
RAILS_ROOT = "/Users/tom/dev/gravatar2"

%w{8200 8201 8202}.each do |port|
  God.watch do |w|
    w.name = "gravatar2-mongrel-#{port}"

    w.start = "mongrel_rails start -c #{RAILS_ROOT} -p #{port} \
      -P #{RAILS_ROOT}/log/mongrel.#{port}.pid  -d"
    w.stop = "mongrel_rails stop -P #{RAILS_ROOT}/log/mongrel.#{port}.pid"
    w.restart = "mongrel_rails restart -P #{RAILS_ROOT}/log/mongrel.#{port}.pid"

    w.pid_file = File.join(RAILS_ROOT, "log/mongrel.#{port}.pid")

    w.behavior(:clean_pid_file)

    w.start_if do |start|
      start.condition(:process_running) do |c|
        c.interval = 5.seconds
        c.running = false
      end
    end

    w.restart_if do |restart|
      restart.condition(:memory_usage) do |c|
        c.above = 150.megabytes
        c.times = [3, 5] # 3 out of 5 intervals
      end

      restart.condition(:cpu_usage) do |c|
        c.above = 50.percent
        c.times = 5
      end
    end

    # lifecycle
    w.lifecycle do |on|
      on.condition(:flapping) do |c|
        c.to_state = [:start, :restart]
        c.times = 5
        c.within = 5.minute
        c.transition = :unmonitored
        c.retry_in = 10.minutes
        c.retry_times = 5
        c.retry_within = 2.hours
      end
    end
  end
end

~~~

## 浅拷贝和深拷贝
- https://ruby-china.org/topics/22164
- clone , dup 都是浅拷贝
- 实现深拷贝: Marshal.load(Marshal.dump(obj))
- rails 中提供了 deep_dup

### 例子

~~~ruby

class Obj
  attr_accessor :first, :second
  def initialize
    @first = {:one => 'x',:two => 'y',:three => 'z'}
    @second = {[1,2]=>'x',[3,2]=>'o'}
  end
end

# 浅拷贝
obj1 = Obj.new
obj2 = obj1.clone

obj1.object_id != obj2.object_id
# 拷贝对象和源对象都指向同样的依赖
obj1.first.object_id == obj2.first.object_id

# 深拷贝
obj3 = Marshal.load( Marshal.dump(obj1) )
# 依赖也被拷贝了
obj3.first.object_id != obj1.first.object_id

~~~

## Test Unit

### test unit 运行一个单独的测试方法

~~~bash
$ ruby -Itest test/services/bet_query/hc_bet_query_test.rb -n test_expired_bet_query
~~~


## 将 hash 写入到 yml 文件

~~~ruby
TR_CITIES = {
  "Adana"=>["Seyhan", "Ceyhan", "Feke", "Karaisalı", "Karataş", "Kozan", "Pozantı", "Saimbeyli", "Tufanbeyli", "Yumurtalık", "Yüreğir", "Aladağ", "İmamoğlu", "Sarıçam", "Çukurova"]
}
  
File.open('tr_sub_cities.yml', 'w') do |file|
  file.write TR_CITIES.to_yaml(encoding: 'utf8')
end

res = File.load_file('tr_sub_cities.yml')
~~~

## delegate methods

### Forwardable

### Delegate (Rails only)

### SimpleDelegator

~~~ruby
class User
  def born_on
    Date.new(1989, 9, 10)
  end
end

class UserDecorator < SimpleDelegator
  def birth_year
    born_on.year
  end
end

decorated_user = UserDecorator.new(User.new)
decorated_user.birth_year  #=> 1989
decorated_user.__getobj__  #=> #<User: ...>
~~~

## UTF-8 Invalid Byte Sequences

问题代码,

~~~ruby
res.gsub("\n\n", "\n")
~~~
报 UTF-8 Invalid Byte Sequences 异常, 解决方法,

~~~ruby
ic = ::Iconv.new('UTF-8//IGNORE', res.encoding.to_s)
res = ic.iconv(res)
res.gsub("\n\n", "\n")
~~~

## Rake

###  run tasks from within rake tasks

参考: http://stackoverflow.com/questions/577944/how-to-run-rake-tasks-from-within-rake-tasks

实际应用

~~~ruby
  desc '统计所有追号状况'
  task :stat_all_zhuihao, [:num] => [:environment] do |t, args|
    Rake::Task['staging:stat_zhuihao'].reenable
    Rake::Task['staging:stat_zhuihao'].invoke('排列三[组三]', args['num'])
  end
~~~  

### use argument in rake task rake参数
- 参考: http://robots.thoughtbot.com/post/18129303042/how-to-use-arguments-in-a-rake-task
- 参考: https://github.com/robbyrussell/oh-my-zsh/issues/433

~~~ruby
namespace :tweets do
  desc 'Send some tweets to a user'
  task :send, [:username] => [:environment] do |t, args|
    Tweet.send(args[:username])
  end
end
rake tweets:send[cpytel]
~~~
使用zsh时会报zsh: no matches found: tweets:send[cpytel]的错误

- 解决办法

rake tweets:send\[cpytel\]
or
rake 'tweets:send[cpytel]'
or
Add alias rake='noglob rake' in your .zshrc

### 获取 task 的描述
- http://stackoverflow.com/questions/8781263/access-rake-task-description-from-within-task

~~~ruby
Rake::TaskManager.record_task_metadata = true

desc "Populate DB"
task :populate do |task|
  p task.comment # "Populate DB"
  p task.full_comment # "Populate DB"
  p task.name # "populate "
end
~~~

### Deep Copy

- http://www.blackbytes.info/2016/01/ruby-tricks/

~~~ruby
food = %w( bread milk orange )
food.map(&:object_id)       # [35401044, 35401020, 35400996]
food.clone.map(&:object_id) # [35401044, 35401020, 35400996]
~~~

~~~ruby
def deep_copy(obj)
  Marshal.load(Marshal.dump(obj))
end

deep_copy(food).map(&:object_id) # [42975648, 42975624, 42975612]
~~~
