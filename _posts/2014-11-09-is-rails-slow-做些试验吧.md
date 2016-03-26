---
layout: post
title: Is Rails Slow? 做些试验吧
---

看了这个帖子 [https://ruby-china.org/topics/22558](https://ruby-china.org/topics/22558)

里面有个 ppt: [https://speakerdeck.com/a_matsuda/is-rails-slow](https://speakerdeck.com/a_matsuda/is-rails-slow)

这个 ppt 的内容很好, 下面是来自 @lgn23st 对此 ppt 的总结,

- Ruby 2.0 到 2.1 的 GC 改进和性能优化
- Rails 4.1 到 4.2 的性能优化
- 各种 Template engine 以及坑爹的 url helpers 对性能的影响
- 如何用最潮的 benchmark 工具和方法
- 顺便给 hamlx 打了个广告

> 一个 PPT 中把这些全部展示出来了，值得每个 Rubyist 亲自动手跟着 PPT 实践一遍！

我看了好几遍 ppt, 想动手对这个 ppt 实践一遍, 动手前先了解下我们要达到的目的吧,

1. 体验 Ruby 2.0 到 2.1 的 GC 改进和性能优化
2. 体验 Rails 4.1 到 4.2 的性能优化
3. 感受下 各种 Template engine 以及坑爹的 url helpers 对性能的影响
4. 体验最潮的 benchmark 工具和方法
5. 体验下 hamlx
6. 体验 haml 分别在 rails 和 sinatra 下显著地性能差异，如果改成 erb 会怎样呢?

现在我们开始动手实践,

## 第一回合

- 建立一个 rails 项目

~~~
 % rails new rails_is_slow
~~~

- 配置环境变量 SECRET\_KEY\_BASE

~~~
% export SECRET_KEY_BASE=abc123
~~~

若以 production 环境启动 rails 应用，必须配置 SECRET\_KEY\_BASE, 可以在 config/secrets.yml 文件里找到 secret\_key\_base。

- 生成控制器和配置路由

~~~
% rails g controller hello index
~~~

config/routes.rb,

~~~ruby

Rails.application.routes.draw do
  get 'hello' => 'hello#index'
end

~~~

- 启动 rails server

~~~
  rails s -e production
~~~

- 使用浏览器访问 http://localhost:3000/hello

访问 5 次, 9ms, 12ms, 12ms, 11ms, 11ms 平均每次的响应时间是  11 ms

- 建立一个 sinatra 项目

~~~
  % mkdir rails_is_slow_sinatra
~~~

Gemfile

~~~ruby
  source 'https://rubygems.org'
  gem 'sinatra'
~~~

app.rb

~~~ruby
  require 'bundler/setup'
  Bundler.require

  get '/hello' do
    'Hello, World!'
  end
~~~

- 启动 sinatra

~~~
  RACK_ENV=production bundle ex ruby app.rb
~~~

- 使用浏览器访问 http://localhost:4567/hello

访问 5 次, 9ms, 11ms, 9ms, 10ms, 9ms 平均每次的响应时间是 9.6ms


> 在不使用数据库，不使用视图模板时，即两个应用都为裸应用时, rails 和 sinatra 两者的性能几乎相同

查看 rails 的 production.log,

~~~
I, [2014-11-09T21:23:05.124508 #14822]  INFO -- : Started GET "/hello" for 127.0.0.1 at 2014-11-09 21:23:05 +0800
I, [2014-11-09T21:23:05.125675 #14822]  INFO -- : Processing by HelloController#index as HTML
I, [2014-11-09T21:23:05.127045 #14822]  INFO -- :   Rendered text template (0.0ms)
I, [2014-11-09T21:23:05.127308 #14822]  INFO -- : Completed 200 OK in 1ms (Views: 0.5ms | ActiveRecord: 0.0ms)
I, [2014-11-09T21:23:12.206147 #14822]  INFO -- : Started GET "/hello" for 127.0.0.1 at 2014-11-09 21:23:12 +0800
I, [2014-11-09T21:23:12.207563 #14822]  INFO -- : Processing by HelloController#index as HTML
I, [2014-11-09T21:23:12.208128 #14822]  INFO -- :   Rendered text template (0.0ms)
I, [2014-11-09T21:23:12.208380 #14822]  INFO -- : Completed 200 OK in 1ms (Views: 0.3ms | ActiveRecord: 0.0ms)
~~~

我们注意到在 rails 中, hello 这个请求只花费了 1ms

## 第二回合

- 建立一个 scaffold, 并且使用 haml 模板

Gemfile

~~~ruby
  +gem 'haml-rails'
~~~

~~~
  % rails g scaffold user name account email phone zip address birthday:datetime age:integer company bio:text admin:boolean
  % RAILS_ENV=production rake db:migrate
~~~

- 通过 seeds 准备 10,000 条记录

db/seeds.rb

~~~ruby
date_range = Date.parse('1960/1/1')..(10000.days.since(Date.parse('1960/1/1')))
[*1..10000].each do |i|
  User.create! name: "User %05d" % i,
  account: "user_%05d" % i,
  email: "user%05d@example.com" % i,
  phone: "%011d" % i,
  zip: "%03d" % (i / 10),
  address: "#{i}-#{i} Tokyo, Japan" % i,
  birthday: rand(date_range),
  age: i / 365,
  company: "#{i}Signals",
  bio: 'Lorem ipsum dolor sit amet, consectetur adipisicing
elit, sed do eiusmod tempor incididunt ut labore et dolore
magna aliqua. Ut enim ad minim veniam, quis nostrud exerci
tation ullamco laboris nisi ut aliquip ex ea commodo
consequat. Duis aute irure dolor in reprehenderit in
voluptate velit esse cillum dolore eu fugiat nulla pariatur.
Excepteur sint occaecat cupidatat non proiden
t, sunt in culpa qui officia deserunt mollit anim id est
laborum.',
  admin: true
end

~~~

~~~
% RAILS_ENV=production bin/rake db:seed
~~~

- 启动 rails 服务, 并且查看日志

~~~
  % tail -f log/production.log | grep Completed
~~~

使用浏览器访问 http://localhost:3000/users, 我们可以看到下面的日志输出,

~~~
I, [2014-11-09T22:21:11.856447 #51508]  INFO -- : Completed 200 OK in 7499ms (Views: 7378.1ms | ActiveRecord: 112.5ms)
I, [2014-11-09T22:21:38.756297 #51508]  INFO -- : Completed 200 OK in 7598ms (Views: 7538.8ms | ActiveRecord: 58.8ms)
I, [2014-11-09T22:21:58.880376 #51508]  INFO -- : Completed 200 OK in 8133ms (Views: 8069.8ms | ActiveRecord: 62.5ms)
I, [2014-11-09T22:22:05.772164 #51508]  INFO -- : Completed 200 OK in 6774ms (Views: 6718.8ms | ActiveRecord: 54.0ms)
I, [2014-11-09T22:22:17.464458 #51508]  INFO -- : Completed 200 OK in 7802ms (Views: 7732.4ms | ActiveRecord: 68.0ms)
I, [2014-11-09T22:22:27.680324 #51508]  INFO -- : Completed 200 OK in 8425ms (Views: 8339.1ms | ActiveRecord: 84.7ms)
~~~

同时日志中有一些和资源加载相关的错误,

~~~
ActionController::RoutingError (No route matches [GET] "/stylesheets/application.css"):
  actionpack (4.1.7) lib/action_dispatch/middleware/debug_exceptions.rb:21:in `call'
  actionpack (4.1.7) lib/action_dispatch/middleware/show_exceptions.rb:30:in `call'
~~~

所以我们移除 layout 和 assets,

~~~ruby
class UsersController < ApplicationController
  before_action :set_user, only: [:show, :edit, :update, :destroy]
  +layout :false
  ...
~~~

在没有 layout 的情况下，响应时间如下,

~~~
I, [2014-11-09T22:29:30.293831 #52308]  INFO -- : Completed 200 OK in 7114ms (Views: 7020.4ms | ActiveRecord: 85.8ms)
I, [2014-11-09T22:29:43.623968 #52308]  INFO -- : Completed 200 OK in 8484ms (Views: 8293.2ms | ActiveRecord: 190.3ms)
I, [2014-11-09T22:30:34.257335 #52308]  INFO -- : Completed 200 OK in 7203ms (Views: 7102.5ms | ActiveRecord: 99.5ms)
I, [2014-11-09T22:30:45.708797 #52308]  INFO -- : Completed 200 OK in 9109ms (Views: 9019.5ms | ActiveRecord: 88.2ms)
I, [2014-11-09T22:30:57.693639 #52308]  INFO -- : Completed 200 OK in 9040ms (Views: 8955.4ms | ActiveRecord: 83.7ms)
~~~

此时响应时间还是 8ms 左右, 我们可以看到 ActiveRecord 花费的时间并不多，只有 80ms 左右，大部分(98%)的时间都用在了渲染 Haml视图上去了。我们先不着急
和 sinatra 作比较, 我们将 haml 和 erb 作个比较。在 routes.rb 里增加新的路由,

~~~ruby
Rails.application.routes.draw do
  resources :users

  +get 'users_index_erb' => 'users#index_erb'
  get 'hello' => 'hello#index'
end
~~~

在 UsersController 里增加一个方法: index_erb,

~~~ruby
class UsersController < ApplicationController
  ...
  def index_erb
    @users = User.all
  end
  ...
end
~~~

增加相应的视图文件 index_erb.html.erb, 可以利用  [https://haml2erb.org/](https://haml2erb.org/) 把 index.html.haml 的内容转换为 erb,

~~~html
<h1>Listing users</h1>
<table>
  <tr>
    <th>Name</th>
    <th>Account</th>
    <th>Email</th>
    <th>Phone</th>
    <th>Zip</th>
    <th>Address</th>
    <th>Birthday</th>
    <th>Age</th>
    <th>Company</th>
    <th>Bio</th>
    <th>Admin</th>
    <th></th>
    <th></th>
    <th></th>
  </tr>
  <% @users.each do |user| %>
    <tr>
      <td>
        <%= user.name %>
      </td>
      <td>
        <%= user.account %>
      </td>
      <td>
        <%= user.email %>
      </td>
      <td>
        <%= user.phone %>
      </td>
      <td>
        <%= user.zip %>
      </td>
      <td>
        <%= user.address %>
      </td>
      <td>
        <%= user.birthday %>
      </td>
      <td>
        <%= user.age %>
      </td>
      <td>
        <%= user.company %>
      </td>
      <td>
        <%= user.bio %>
      </td>
      <td>
        <%= user.admin %>
      </td>
      <td>
        <%= link_to 'Show', user %>
      </td>
      <td>
        <%= link_to 'Edit', edit_user_path(user) %>
      </td>
      <td>
        <%= link_to 'Destroy', user, :method => :delete, :data => { :confirm => 'Are you sure?' } %>
      </td>
    </tr>
  <% end %>
</table>
<br/>
<%= link_to 'New User', new_user_path %>
~~~

- 使用浏览器访问 http://localhost:3000/users\_index\_erb

访问 5 次，每次的响应时间分别是: 10ms, 9ms, 9ms, 11ms, 10ms

日志输出为:

~~~
I, [2014-11-14T22:28:04.695994 #83566]  INFO -- : Completed 200 OK in 7404ms (Views: 7231.1ms | ActiveRecord: 171.7ms)
I, [2014-11-14T22:29:58.455348 #83566]  INFO -- : Completed 200 OK in 6941ms (Views: 6749.0ms | ActiveRecord: 190.7ms)
I, [2014-11-14T22:30:22.522506 #83566]  INFO -- : Completed 200 OK in 6798ms (Views: 6692.7ms | ActiveRecord: 104.2ms)
I, [2014-11-14T22:30:41.590326 #83566]  INFO -- : Completed 200 OK in 8399ms (Views: 8293.4ms | ActiveRecord: 104.1ms)
I, [2014-11-14T22:31:16.102876 #83566]  INFO -- : Completed 200 OK in 6871ms (Views: 6739.3ms | ActiveRecord: 131.3ms)
~~~

> 我们可以认为与 haml 相比较，使用 erb 并没有提高性能


### Sinatra + Haml + ActiveRecord

- 启用 Haml 和 ActiveRecord

Gemfile

~~~ruby
gem 'sinatra'
gem 'sinatra-activerecord'
gem 'sqlite3'
gem 'haml'
~~~
- 连接数据库

app.rb

~~~ruby
+set :database, adapter: 'sqlite3', database: 'production.sqlite3'
+class User < ActiveRecord::Base
end
~~~

将 rails\_is\_slow 项目下的 production.sqlite3 数据库拷贝到项目下,

~~~
 $ cp rails_is_slow/db/production.sqlite3 rails_is_slow_sinatra/
~~~

- 实现 /users

app.rb

~~~ruby
+get '/users' do
+ @users = User.all.to_a
+ haml :index
+end
~~~

将 rails\_is\_slow 项目下的 users 视图文件拷贝到 rails\_is\_slow\_sinatra 项目下，

~~~
$ cp rails_is_slow/app/views/users/index.html.haml rails_is_slow_sinatra/views/index.haml
~~~

注释掉 index.haml 内的一些 url helper, 比如 link\_to, edit\_user\_path, new\_user\_path 等.

使用浏览器访问 http://localhost:4567/users, 访问 5 次， 每次的响应时间是 3s, 3s, 4s, 4s, 3s。

我们再将 rails\_is\_slow\_sinatra/views/index.haml 拷贝回 rails\_is\_slow/app/views/users/index.html.haml

使用浏览器访问 http://localhost:3000/users, 访问 5 次, 每次的响应时间是 4ms, 4ms, 5ms, 4ms, 6ms, 这时我们发现如果去掉视图里的各种 url helper, rails 的性能
大概提高了一倍, 因此 rails 中的 url helper 的性能是一个值得关注的问题。

> Rails 中的 url helper 对性能会有较大的负面影响


### 调整 Ruby GC

按照 [is-rails-slow](https://speakerdeck.com/a_matsuda/is-rails-slow) 这个 ppt 的步骤，我们对 Ruby 的 GC 做些试验。

Gemfile

~~~ruby
+gem 'gc_tracer'
~~~

UsersController.rb

~~~ruby
  def index
-    @users = User.all.to_a
+    GC::Tracer.start_logging(Rails.root.join("tmp/gc.txt").to_s) do
+      @users = User.all.to_a
+      render
+    end
  end
~~~

每次访问 http://localhost:3000/users 后，我们在 tmp/gc.txt 文件里能够得到一堆和 GC 相关的输出:

~~~
tick	type	count	heap_used	heap_length	heap_increment	heap_live_slot	heap_free_slot	heap_final_slot	heap_swept_slot	heap_eden_page_length	heap_tomb_page_length	total_allocated_object	total_freed_object	malloc_increase	malloc_limit	minor_gc_count	major_gc_count	remembered_shady_object	remembered_shady_object_limit	old_object	old_object_limit	oldmalloc_increase	oldmalloc_limit	major_by	gc_by	have_finalizer	immediate_sweep	ru_utime	ru_stime	ru_maxrss	ru_ixrss	ru_idrss	ru_isrss	ru_minflt	ru_majflt	ru_nswap	ru_inblock	ru_oublock	ru_msgsnd	ru_msgrcv	ru_nsignals	ru_nvcsw	ru_nivcsw
133232142907	gc_start	51	3532	4228	696	1381961	57670	0	758069	3532	0	7700718	6318757	31773504	31773459	41	9	24454	27704	652626	819366	31773504	24159190	oldmalloc	malloc	0	11922588	585390	289144832	0	0	0	73651	6	0	2	118	32	41	0	316	15767
133422656054	gc_end_m	51	3532	4228	696	1381961	57670	0	758069	3532	0	7700718	6318757	31773504	31773459	41	10	7958	27704	386214	819366	31773504	24159190	oldmalloc	malloc	0	11990883	587034	289153024	0	0	0	73653	6	0	2	118	32	41	0	316	16005
134303721618	gc_end_s	51	3532	4228	696	398669	1040973	11	1040962	2964	568	7700718	7302049	0	31773504	41	10	7958	15916	386214	772428	0	24159190	oldmalloc	malloc	0	1	12302598	600975	289153024	0	0	0	73656	6	0	2	118	32	41	0	316	16570
138024843100	gc_start	52	3532	4228	696	1112413	327218	0	1040973	2964	568	8414473	7302060	31773872	31773504	41	10	7958	15916	386214	772428	31773872	24159190	0	malloc	0	1	13581580	624087	289153024	0	0	0	74576	6	0	2	118	32	41	0	323	18719
138088825986	gc_end_m	52	3532	4228	696	1112413	327218	0	1040973	2964	568	8414473	7302060	31773872	31773504	42	10	14126	15916	410891	772428	31773872	24159190	0	malloc	0	1	13605305	624365	289153024	0	0	0	74576	6	0	2	118	32	41	0	323	18751
138538171872	gc_end_s	52	3532	4228	696	429503	1010128	0	778610	2964	568	8414473	7984970	0	31773824	42	10	14126	15916	410891	772428	2067344	28991028	0	malloc	0	1	13736593	625907	289153024	0	0	0	74576	6	0	2	118	32	41	0	323	18984

~~~

说实在话我没有看明白这些内容的意思。

现在我们停止 GC, 然后观察一下 rails\_is\_slow 的性能变化。我们把 GC.disable 这段代码加到某个配置文件中, 比如,

config/gc.rb

~~~ruby
GC.disable
~~~

重启应用后，我们会发现 GC 停止了, 可以通过观察 tmp/gc.txt 的内容得出这个结论,

~~~
tick	type	count	heap_used	heap_length	heap_increment	heap_live_slot	heap_free_slot	heap_final_slot	heap_swept_slot	heap_eden_page_length	heap_tomb_page_length	total_allocated_object	total_freed_object	malloc_increase	malloc_limit	minor_gc_count	major_gc_count	remembered_shady_object	remembered_shady_object_limit	old_object	old_object_limit	oldmalloc_increase	oldmalloc_limit	major_by	gc_by	have_finalizer	immediate_sweep	ru_utime	ru_stime	ru_maxrss	ru_ixrss	ru_idrss	ru_isrss	ru_minflt	ru_majflt	ru_nswap	ru_inblock	ru_oublock	ru_msgsnd	ru_msgrcv	ru_nsignals	ru_nvcsw	ru_nivcsw
~~~

我们可以看到上面的内容里已经没有 GC 相关的数据了。


关闭 GC 时的响应时间:

~~~
I, [2014-11-15T13:36:25.785793 #66181]  INFO -- : Completed 200 OK in 2079ms (Views: 1823.2ms | ActiveRecord: 73.8ms)
I, [2014-11-15T13:36:31.061513 #66181]  INFO -- : Completed 200 OK in 2735ms (Views: 2392.9ms | ActiveRecord: 88.8ms)
I, [2014-11-15T13:36:47.150416 #66181]  INFO -- : Completed 200 OK in 2100ms (Views: 1827.2ms | ActiveRecord: 72.0ms)
I, [2014-11-15T13:37:05.795685 #66181]  INFO -- : Completed 200 OK in 2235ms (Views: 1952.2ms | ActiveRecord: 74.0ms)
I, [2014-11-15T13:37:21.624737 #66181]  INFO -- : Completed 200 OK in 2106ms (Views: 1814.3ms | ActiveRecord: 103.8ms)
I, [2014-11-15T13:37:37.734350 #66181]  INFO -- : Completed 200 OK in 2357ms (Views: 2067.1ms | ActiveRecord: 101.9ms)
~~~

打开 GC 时的响应时间:

~~~
I, [2014-11-15T13:38:45.708158 #66845]  INFO -- : Completed 200 OK in 2762ms (Views: 2381.7ms | ActiveRecord: 134.9ms)
I, [2014-11-15T13:38:56.903308 #66845]  INFO -- : Completed 200 OK in 2640ms (Views: 2290.2ms | ActiveRecord: 100.2ms)
I, [2014-11-15T13:39:21.561863 #66845]  INFO -- : Completed 200 OK in 2419ms (Views: 2135.5ms | ActiveRecord: 87.9ms)
I, [2014-11-15T13:39:37.824292 #66845]  INFO -- : Completed 200 OK in 2170ms (Views: 1879.4ms | ActiveRecord: 66.2ms)
I, [2014-11-15T13:39:53.330387 #66845]  INFO -- : Completed 200 OK in 2312ms (Views: 2067.5ms | ActiveRecord: 63.8ms)
I, [2014-11-15T13:40:15.175491 #66845]  INFO -- : Completed 200 OK in 2197ms (Views: 1909.7ms | ActiveRecord: 77.6ms)
~~~

> 两者对比下，可以看到 GC 的关闭或者打开对 rails 应用的性能影响不大


### 试试 Rails 4.2

@tenderlove 说 "Rails 4.2 is going to be the fastest Rails ever.", 所以我们现在试试 Rails 4.2。

Gemfile

~~~ruby
-gem 'rails', '4.1.7'
# Bundle edge Rails instead: gem 'rails', github: 'rails/rails'
gem 'rails', github: 'rails/rails'
~~~

打开 GC， 访问 http://localhost:3000/users, 响应时间:

~~~
I, [2014-11-15T15:14:57.014377 #96328]  INFO -- : Completed 200 OK in 2926ms (Views: 2609.7ms | ActiveRecord: 99.3ms)
I, [2014-11-15T15:16:33.169118 #96328]  INFO -- : Completed 200 OK in 3026ms (Views: 2658.8ms | ActiveRecord: 162.1ms)
I, [2014-11-15T15:18:59.715360 #96328]  INFO -- : Completed 200 OK in 2934ms (Views: 2685.4ms | ActiveRecord: 73.7ms)
I, [2014-11-15T15:22:14.851652 #96328]  INFO -- : Completed 200 OK in 2987ms (Views: 2745.6ms | ActiveRecord: 66.0ms)
I, [2014-11-15T15:22:25.610507 #96328]  INFO -- : Completed 200 OK in 3831ms (Views: 3518.6ms | ActiveRecord: 110.0ms)
~~~

关闭 GC, GC.disable, 访问 http://localhost:3000/users, 响应时间:

~~~
I, [2014-11-15T15:24:15.153951 #97266]  INFO -- : Completed 200 OK in 2878ms (Views: 2604.2ms | ActiveRecord: 74.3ms)
I, [2014-11-15T15:24:24.363854 #97266]  INFO -- : Completed 200 OK in 4043ms (Views: 3607.7ms | ActiveRecord: 103.2ms)
I, [2014-11-15T15:24:34.652407 #97266]  INFO -- : Completed 200 OK in 3658ms (Views: 3268.5ms | ActiveRecord: 135.3ms)
I, [2014-11-15T15:24:48.085198 #97266]  INFO -- : Completed 200 OK in 4231ms (Views: 3863.1ms | ActiveRecord: 108.3ms)
I, [2014-11-15T15:24:57.836219 #97266]  INFO -- : Completed 200 OK in 3847ms (Views: 3443.1ms | ActiveRecord: 106.7ms)
~~~

> 从目前的试验来看, rails4.1 和 rails4.2 之间的性能差不多

那么 rails 4.2 的 ActiveRecord 到底发生了哪些变化呢?

### 使用 TracePoint 查看哪些方法被调用了

从 ruby2.0 起 TracePoint 已经被引入到了 ruby 核心。

app/controllers/users_controller.rb

~~~ruby
  def index
    calls = []
    trace = TracePoint.new(:call) do |tp|
      calls << [tp.defined_class, tp.method_id, tp.lineno]
    end
    trace.enable
    @users = User.all.to_a
    trace.disable
    pp calls.group_by(&:itself).map {|k, v| {k => v.length}}.sort_by {|h| -h.values.first }
  end
~~~

访问 http://localhost:3000/users, 有如下输出:

rails 4.2,

~~~
[{[ActiveRecord::AttributeMethods::ClassMethods,
   :define_attribute_methods,
   79]=>20000},
 {[ActiveSupport::Callbacks::CallbackChain, :empty?, 488]=>20000},
 {[ActiveSupport::Callbacks, :_run_callbacks, 86]=>20000},
 {[ActiveRecord::ModelSchema::ClassMethods, :inheritance_column, 186]=>20000},
 {[Object, :present?, 23]=>10001},
 {[#<Class:ActiveRecord::Base>, :_initialize_callbacks, 110]=>10000},
 {[ActiveRecord::Base, :_run_initialize_callbacks, 733]=>10000},
 {[ActiveRecord::Inheritance::ClassMethods,
   :discriminate_class_for_record,
   167]=>10000},
 {[#<Class:ActiveRecord::Base>, :_find_callbacks, 110]=>10000},
 {[ActiveRecord::Base, :_find_callbacks, 734]=>10000},
 {[ActiveRecord::Inheritance::ClassMethods,
   :using_single_table_inheritance?,
   175]=>10000},
 {[ActiveRecord::Base, :_initialize_callbacks, 734]=>10000},
 {[ActiveRecord::Persistence::ClassMethods, :instantiate, 66]=>10000},
 {[ActiveRecord::Base, :_run_find_callbacks, 733]=>10000},
 {[NilClass, :blank?, 54]=>10000},
 {[ActiveRecord::AttributeSet, :initialize, 5]=>10000},
 {[ActiveRecord::Core, :init_with, 297]=>10000},
 {[ActiveRecord::Persistence::ClassMethods,

~~~

ppt 上标注的是

~~~
[{[ActiveRecord::AttributeMethods::ClassMethods,
   :define_attribute_methods,
   79]=>140000},
~~~

而我们现在的试验结果是

~~~
[{[ActiveRecord::AttributeMethods::ClassMethods,
   :define_attribute_methods,
   79]=>20000},
~~~

当时 ppt 的作者认为这是一个问题，会消耗大量的 CPU 时间, 因为 140000 是 20000的 7倍，按目前的测试来看，rails4.2已经修复了这个问题, 即 ActiveRecord 的一些方法
调用回到了正常水平。

### 试验 Stackprof

stackprof 是 @tmm1 写的一个 gem, 可以用于 ruby2.1+, 这个 gem 可以用于查看方法调用所消耗的 CPU 时间。

app/controllers/users_controller.rb

~~~ruby
def index
  StackProf.run(mode: :cpu, out: 'tmp/stackprof-cpu-ar.dump') do
    @users = User.all.to_a
  end
end
~~~

可惜在我的试验中, tmp/stackprof-cpu-ar.dump 输出的是一堆乱码。

### hamlx

作者给出了 hamlx, 貌似是一个优化版本的 haml, 作者说使用 hamlx 比使用 haml 会有 10% ~ 20% 的性能提高。
由于作者还没有发布 hamlx, 我们就不做这个试验了。

### 试验用到的项目代码和 PPT

- 代码: [rails\_is\_slow](https://github.com/baya/rails_is_slow)

- 代码: [rails\_is\_slow\_42](https://github.com/baya/rails_is_slow_42-)

- 代码: [rails\_is\_slow\_sinatra](https://github.com/baya/rails_is_slow_sinatra)

- PPT: [Is Rails Slow?](https://speakerdeck.com/a_matsuda/is-rails-slow)
