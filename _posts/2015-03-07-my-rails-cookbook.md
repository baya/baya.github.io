---
layout: post
title: 我的 Rails Cookbook
---

## 构造联合查询

~~~ruby
def self.search(title, category_id, city_id)
    rel = scoped
    rel = rel.where('title    like ?', "%#{title}%") if(title)
    rel = rel.where('category_id = ?', category_id)  if(category_id)
    rel = rel.where('city_id     = ?', city_id)      if(city_id)
    rel
end
~~~

## 可用的回调

### Creating an Object

~~~ruby
before_validation
after_validation
before_save
around_save
before_create
around_create
after_create
after_save
after_commit/after_rollback
~~~

### Updating an Object

~~~ruby
before_validation
after_validation
before_save
around_save
before_update
around_update
after_update
after_save
after_commit/after_rollback
~~~

### Destroying an Object

~~~ruby
before_destroy
around_destroy
after_destroy
after_commit/after_rollback
~~~

## 调用save方法时的callback顺序

~~~ruby
before_validation
before_validation :on => :create
after_validation
after_validation  :on => :create
before_save
before_create
after_create
after_save
~~~

## callback方法之间的一些差别

- 参考: http://stackoverflow.com/questions/6422199/what-is-the-difference-between-after-create-and-after-save-and-when-to-use-w
- after_create only works once - just after the record is first created.
- after_save works every time you save the object - even if you're just updating it many years later

## Generator 例子

~~~text
bin/rails g model pa/bet --migration=false --fixture-replacement=fabrication
bin/rails g model pa/prebet --migration=false --fixture-replacement=fabrication
bin/rails g controller pa/term_notices --no-assets
bin/rails g controller SlsChannels --no-assets
bin/rails g model LotFarm --no-fixture --no-migration
bin/rails g model sls/activity_verify_charge --no-fixture --no-migration
~~~

## 自定义config

~~~ruby
# config/initializers/load_config.rb
APP_CONFIG = YAML.load_file("#{Rails.root}/config/config.yml")[Rails.env]

# application.rb
if APP_CONFIG['perform_authentication']
  # Do stuff
end
~~~

## Tabel.column migrations

~~~ruby
change_table :table do |t|
  t.column
  t.index
  t.timestamps
  t.change
  t.change_default
  t.rename
  t.references
  t.belongs_to
  t.string
  t.text
  t.integer
  t.float
  t.decimal
  t.datetime
  t.timestamp
  t.time
  t.date
  t.binary
  t.boolean
  t.remove
  t.remove_references
  t.remove_belongs_to
  t.remove_index
  t.remove_timestamps
end
~~~

## ActiveModel::Dirty

- 参考:http://stackoverflow.com/questions/5501334/how-to-properly-handle-changed-att

~~~ruby
person = Person.find_by_name('Uncle Bob')
person.changed?       # => false
person.name = 'Bob'
person.changed?       # => true
person.name_changed?  # => true
person.name_was       # => 'Uncle Bob'
person.name_change    # => ['Uncle Bob', 'Bob']
~~~

## Whenever

~~~text
whenever --set environment=development -i
whenever --set environment=staging -i
~~~

## 时区 Time Zone

~~~text
rake time:zones:all
config.time_zone = 'Beijing'
config.active_record.default_timezone = :local
Rails stores at +0000 Time zone by default
~~~

## HTML标签辅助方法 helper method

### select

~~~text
select '', 'lot_name', ['重庆时时彩', '双色球'], include_blank: false
select '', 'terminal_no', [['任意', nil]\]

options_for_contest_itype = [['photo', 0], ['video', 1], ['word', 2], ['audio', 4]]
select 'contest', 'itype', options_for_contest_itype, selected: @contest.itype

select '', 'contest[sponsoring[][sponsor]]', options_for_contest_sponsors, selected: sponsoring.sponsor.id
~~~

### radio button

~~~ text
radio_button '', 'u_depoted', true, checked: @pre_term.depoted

<%= radio_button 'contest', 'use_default_rule', true, checked: @preview_contest.use_default_rule, style: 'display:none;', class: 't' %>
<%= radio_button 'contest', 'use_default_rule', false, checked: !@preview_contest.use_default_rule, style: 'display:none;', class: 'f' %>
~~~

### check box

~~~text
<%= check_box_tag 'contest[use_default_rule]', @preview_contest.use_default_rule, @preview_contest.use_default_rule, style: 'display:none;' %>

~~~


## API 设计

### Versioned Controller

~~~ruby
Applications.routes.draw do
  namespace :v1 do
    resources :orders # V1::OrdersController, /v1/orders
  end
  namespace :v2 do
    resources :orders # V2::OrdersController, /v2/orders
  end
end

Application.router.draw do
  constraints(:version => 1) do # namespace V1
    resources :orders # V1::OrdersController, /orders
  end
  constraints(:version => 2)  do # namespace V2
    resources :orders # V2::OrdersController, /orders
  end
end
~~~

## cycle helper 单双循环

~~~erb
<tr class=<%= cycle('even', 'odd') %>>
~~~

## HTTP Basic Auth

~~~ruby
class Admin::ApplicationController < ApplicationController

  USER_NAME, PASSWORD = "zhou", "111111"

  before_filter :admin_authenticate


  private

  def admin_authenticate
    authenticate_or_request_with_http_basic do |user_name, password|
      user_name == USER_NAME && password == PASSWORD
    end
  end

end

~~~

## Rake任务在生产环境输出日志

~~~ruby
Rails.logger.auto_flushing = true
~~~

## Rake migrate task

~~~ text
$ rake db:version 查看迁移版本
# 恢复迁移(我需要修改某个历史的migration文件的迁移逻辑，比如20080930121212)
$ rake db:migrate:down VERSION=20080930121212
$ rake db:migrate
~~~

## Helper Method

~~~ruby
#application_controller.rb
def current_user
  @current_user ||= User.find_by_id!(session[:user_id])
end
helper_method :current_user

~~~

## Asset Path Helper

~~~text
.css.erb
background-image:url(<%=asset_path "admin/logo.png"%>);
- 实例
首先将美工给的common.css文件改名为common.css.erb
然后利用asset_path来引入image
.header{width:960px;height:75px;background:url(<%= asset_path 'main_bg.png' %>) 0 0 repeat-x;overflow:hidden;}
~~~

## Mime Types

~~~text
"*/*"                      => :all
"text/plain"               => :text
"text/html"                => :html
"application/xhtml+xml"    => :html
"text/javascript"          => :js
"application/javascript"   => :js
"application/x-javascript" => :js
"text/calendar"            => :ics
"text/csv"                 => :csv
"application/xml"          => :xml
"text/xml"                 => :xml
"application/x-xml"        => :xml
"text/yaml"                => :yaml
"application/x-yaml"       => :yaml
"application/rss+xml"      => :rss
"application/atom+xml"     => :atom
"application/json"         => :json
"text/x-json"              => :json
~~~


## ORM Associations 关系

### Singular associations (one-to-one)

~~~text

                                  |            |  belongs_to  |
generated methods                 | belongs_to | :polymorphic | has_one
----------------------------------+------------+--------------+---------
other                             |     X      |      X       |    X
other=(other)                     |     X      |      X       |    X
build_other(attributes={})        |     X      |              |    X
create_other(attributes={})       |     X      |              |    X
create_other!(attributes={})      |     X      |              |    X

~~~

### Collection associations (one-to-many / many-to-many)

~~~text
                                  |       |          | has_many
generated methods                 | habtm | has_many | :through
----------------------------------+-------+----------+----------
others                            |   X   |    X     |    X
others=(other,other,...)          |   X   |    X     |    X
other_ids                         |   X   |    X     |    X
other_ids=(id,id,...)             |   X   |    X     |    X
others<<                          |   X   |    X     |    X
others.push                       |   X   |    X     |    X
others.concat                     |   X   |    X     |    X
others.build(attributes={})       |   X   |    X     |    X
others.create(attributes={})      |   X   |    X     |    X
others.create!(attributes={})     |   X   |    X     |    X
others.size                       |   X   |    X     |    X
others.length                     |   X   |    X     |    X
others.count                      |   X   |    X     |    X
others.sum(*args)                 |   X   |    X     |    X
others.empty?                     |   X   |    X     |    X
others.clear                      |   X   |    X     |    X
others.delete(other,other,...)    |   X   |    X     |    X
others.delete_all                 |   X   |    X     |    X
others.destroy(other,other,...)   |   X   |    X     |    X
others.destroy_all                |   X   |    X     |    X
others.find(*args)                |   X   |    X     |    X
others.exists?                    |   X   |    X     |    X
others.distinct                   |   X   |    X     |    X
others.uniq                       |   X   |    X     |    X
others.reset                      |   X   |    X     |    X

~~~

## Validate Errors

~~~ruby
# ActiveRecord对象的错误模型类似下面:

user.errors #=>
#<ActiveModel::Errors:0x007f8fe8288370
 @base=
  #<User id: nil, name: nil, birthday: nil, gender: nil, mobile: nil, email: nil, avatar: nil, password_hash: nil, password_salt: nil, sn: nil, status: nil, level: 1, point_count: 0, vote_count: 0, created_at: nil, updated_at: nil, marry: 0, location: nil, role: "user", created_ref: 0>,
 @messages=
  {:email=>["can't be blank", "has already been taken"],
   :mobile=>["can't be blank", "has already been taken"]}>


user.errors.messages #=>
{:email=>["can't be blank", "has already been taken"],
 :mobile=>["can't be blank", "has already been taken"]}

user.errors.sort  #=>
[[:email, "can't be blank"],
 [:email, "has already been taken"],
 [:mobile, "can't be blank"],
 [:mobile, "has already been taken"]]

user.errors.to_a #=>
["Email can't be blank",
 "Email has already been taken",
 "Mobile can't be blank",
 "Mobile has already been taken"]

user.errors.keys #=>
[:email, :mobile]

~~~
## 用户，关注，粉丝的三者关系的在activerecord中的快速实现

### 表结构

~~~text
follows
Column       | Type     | Modifiers | Comment
------------ | -------- | --------- | --------
following_id | integer  |           |  被关注者的id
follower_id  | integer  |           |  关注者的id
created_at   | datetime |           |
updated_at   | datetime |           |

users
Column        | Type     | Modifiers    | Comment
------------- | -------- | ------------ |------------------
id            | integer  |              |
sn            | string   |              |
gender        | integer  |              |  性别, 0 女, 1 男

~~~

### 模型

~~~ruby
class Follow < ActiveRecord::Base
  belongs_to :following, class_name: 'User'
  belongs_to :follower, class_name: 'User'
end

class User < ActivieRecord::Base

  # 有许多偶像
  has_many :following_relationships, class_name: 'Follow', foreign_key: 'follower_id'
  has_many :followings, through: :following_relationships

  # 有许多粉丝
  has_many :follower_relationships, class_name: 'Follow', foreign_key: 'following_id'
  has_many :followers, through: :follower_relationships

end
~~~

## 获取 user agent

~~~ruby
request.env["HTTP_USER_AGENT"]
#or
request.user_agent
~~~

## Test Assert

~~~ruby
 assert( test, [msg] ), Ensures that test is true
 assert_not( test, [msg] ), Ensures that test is false.
 assert_equal( expected, actual, [msg] ), Ensures that expected == actual is true.
 assert_raises( exception1, exception2, ... ) { block }, Ensures that the given block raises one of the given exceptions.

    assert_raise NoEnoughVoteOrCreditError do
      credit_log = FchkCredits::Vote << {user: user, entity: entity}
    end

~~~

## 文件上传 carrierwave

~~~text
$ spring rails generate uploader RuleFile  #=> create app/uploaders/rule_file_uploader.rb
$ spring rails g migration add_rule_file_to_contests
$ rake db:migrate
~~~

~~~ruby
class Contest < ActiveRecord::Base
  mount_uploader :rule_file, RuleFileUploader
end

contest = Contest.new
contest.rule_file = params[:rule_file]
contest.save!
contest.rule_file.url
contest.rule_file.current_path
contest.rule_file.identifier
~~~

## 解析客户端gzip压缩数据

~~~ruby
  ActiveSupport::Gzip.decompress(request.raw_post)
~~~

## http status code

~~~text
Response Class	HTTP Status Code	Symbol
Informational	100	:continue
101	:switching_protocols
102	:processing
Success	200	:ok
201	:created
202	:accepted
203	:non_authoritative_information
204	:no_content
205	:reset_content
206	:partial_content
207	:multi_status
208	:already_reported
226	:im_used
Redirection	300	:multiple_choices
301	:moved_permanently
302	:found
303	:see_other
304	:not_modified
305	:use_proxy
306	:reserved
307	:temporary_redirect
308	:permanent_redirect
Client Error	400	:bad_request
401	:unauthorized
402	:payment_required
403	:forbidden
404	:not_found
405	:method_not_allowed
406	:not_acceptable
407	:proxy_authentication_required
408	:request_timeout
409	:conflict
410	:gone
411	:length_required
412	:precondition_failed
413	:request_entity_too_large
414	:request_uri_too_long
415	:unsupported_media_type
416	:requested_range_not_satisfiable
417	:expectation_failed
422	:unprocessable_entity
423	:locked
424	:failed_dependency
426	:upgrade_required
423	:precondition_required
424	:too_many_requests
426	:request_header_fields_too_large
Server Error	500	:internal_server_error
501	:not_implemented
502	:bad_gateway
503	:service_unavailable
504	:gateway_timeout
505	:http_version_not_supported
506	:variant_also_negotiates
507	:insufficient_storage
508	:loop_detected
510	:not_extended
511	:network_authentication_required
~~~

## log filter params 日志过滤敏感参数

- 参考: http://stackoverflow.com/questions/7232554/how-to-filter-parameters-in-rails

~~~ruby
config.filter_parameters += [:password, :password_confirmation, :credit_card]

filters = Rails.application.config.filter_parameters
f = ActionDispatch::Http::ParameterFilter.new filters
f.filter :password => 'haha' # => {:password=>"[FILTERED]"}
~~~

###  在控制器里实施

~~~ruby

# app/controllers/api/base_api_controller.rb
class Api::BaseApiController < ApplicationController
  filter_parameter_logging :password, :password_confirmation, :card_number
end

# app/controllers/mobile/mobile_api_controller.rb
class Mobile::BaseMobileController < ApplicationController
  filter_parameter_logging :password, :password_confirmation
end

~~~

## Console 里调用 view helpers 方法
- 参考: http://stackoverflow.com/questions/151030/how-do-i-call-controller-view-methods-from-the-console-in-rails

~~~ruby
helper.number_to_currency('123.45')
helper.send :h, '<script>alert("see you")</script>'
~~~

## 样例: 使用 SQL 创建一个带 id 的 表 

需要这样声明 id: `id serial primary key`

~~~sql
create table batch_order_invoice_tasks(
  id serial primary key,
  total_invoice_count integer NOT NULL,
  completed_invoice_count integer NOT NULL default 0,
  user_id integer NOT NULL,
  name character varying(255),
  state character varying(255),
  created_at timestamp without time zone,
  updated_at timestamp without time zone
)
~~~


## ActiveRecord 优化 tips

- http://hashrocket.com/blog/posts/rails-quick-tips-easy-activerecord-optimizations

### blank? VS empty? 使用 empty?

~~~ruby
# Using `blank?`
User.where(screen_name: ['user1','user2']).blank?

# 1. Queries database for all user data
#   SELECT "users".* FROM "users" WHERE "users"."screen_name" IN ('user1','user2')

# 2. Loads users into an array
#   [<#User:0x007fbf6413c510>,<#User:0x007fbf65ab1c70>]

# 3. Checks to see if the array size is zero
#   => true
~~~


~~~ruby
# Using `empty?`
User.where(screen_name: ['user1','user2').empty?

# 1. Queries database for ONLY a count
#   SELECT COUNT(*) FROM "users" WHERE "users"."screen_name" IN ('user1','user2')

# 2. Checks to see if the count is zero
# => true
~~~

### map? VS pluck? 使用 pluck?


~~~ruby
# Using `map?`
User.where(email: ['jane@example.com', 'john@example.com']).map(&:screen_name)

# 1. Queries database for all user data
#   SELECT "users".* FROM "users" WHERE "users"."email" IN ('jane@example.com','john@example.com')

# 2. Loads users into an array
#   [<#User:0x007fbf6413c510>,<#User:0x007fbf65ab1c70>]

# 3. Iterates over users to collect screen_names
#   ['user1','user2']
~~~

~~~ruby
# Using `pluck?`
User.where(email: ['jane@example.com', 'john@example.com']).pluck(:screen_name)

# 1. Queries database for only screen_names
#   SELECT "users"."screen_name" FROM "users" WHERE "users"."email" IN ('jane@example.com','john@example.com')

# 2. Returns those screen_names in an array
#   ['user1','user2']
~~~


~~~ruby
User.where(screen_name: users.select(:screen_name)).empty?
~~~
