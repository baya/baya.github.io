---
layout: post
title:  Rails Respond Format 应用
---

现在摆在我面前有这样一个需求: 用户需要一个报表, 首先这个报表需要在网页上以 table 的形式展示, 然后用户可以将此报表以
CSV, Excel, PDF 的形式下载到本地, 最后用户还希望可以将报表转换为 JSON, XML 等数据以供其他程序使用。
这是一个很繁琐的需求，但是如果我们利用好 Rails 提供的 view 模版以及 respond_to 方法，我们可以很优雅的完成这个需求。
在实际需求中，报表可能会很复杂，但是今天我们的重点不是如何生成报表，而是如何优雅的响应用户的请求，所以我会建立一个简单的 demo
来叙述这一过程。

demo 代码: [respond-format-demo](https://github.com/baya/respond-format-demo)

## format.html

这个响应很容易实现, 我们直接看代码。

路由的代码,

~~~ruby
# config/routes.rb

Rails.application.routes.draw do
+ resources :users
end
~~~

users 控制器的代码,

~~~ruby
# app/controllers/users_controller.rb

class UsersController < ApplicationController

+ def index
+   @users = User.order(:name)
+ end
  
end

~~~

相关视图的代码,

~~~html
# app/views/users/index.html.erb

<h1>Users</h1>
<table class="table">
  <thead>
    <tr>
      <th>#</th>
      <th>名字</th>
      <th>性别</th>
      <th>年龄</th>
    </tr>
  </thead>
  <tbody>
    <% @users.each do |user| %>
      <tr>
	<td><%= user.id %></td>
	<td><%= user.name %></td>
	<td><%= user.human_gender %></td>
	<td><%= user.age %></td>
      </tr>
    <% end %>
  </tbody>
</table>

~~~

响应结果,

![users](/images/Snip20150912_11.png)


## format.csv

为了实现 csv 的输出，我们不需要增加控制器或者 action, 也不需要增加判断逻辑，只需要为 respond_to
方法引入 format.csv 即可。

users 控制器代码,

~~~ruby
# app/controllers/users_controller.rb

class UsersController < ApplicationController

  def index
    @users = User.order(:name)

+   respond_to do |format|
+     format.html
+     format.csv
+   end
  end
  
end
~~~

相关视图的代码, 此时我们增加了一个 index.csv.erb 文件,

~~~text
# app/views/users/index.csv.erb

id,name,gender,age
<% @users.each do |user| %>
<%= [user.id, user.name, user.human_gender, user.age].join(',') %>
<% end %>

~~~

现在我们使用浏览器访问 http://localhost:3000/users.csv, 注意 csv 后缀，这表明我们的
请求格式是 csv, 然后服务器会识别此格式，并且返回给我们一个 csv 文件。

返回的 users.csv 内容,

~~~text
id,name,gender,age
3,小军,男,25
1,小明,男,22
2,小红,女,21
4,小芸,女,24
~~~

## format.xls

为了实现 format.xls 我们首先需要将 xls 注册到 Mime Type 中去。

~~~ruby
# config/initializers/mime_types.rb

+ Mime::Type.register "application/vnd.ms-excel", :xls

~~~

然后在 users 控制器里增加 format.xls,

~~~ruby
# app/controllers/users_controller.rb

class UsersController < ApplicationController

  def index
    @users = User.order(:name)

    respond_to do |format|
      format.html
      format.csv
+     format.xls
    end
  end
  
end
~~~

最后实现 xls 视图, 其实这个视图的内容是一个 xml 文档，浏览器收到此 xml 文档时会将其转换为 xls 文件。

~~~xml
# app/views/users/index.xls.erb

<?xml version="1.0"?>
<Workbook xmlns="urn:schemas-microsoft-com:office:spreadsheet"
  xmlns:o="urn:schemas-microsoft-com:office:office"
  xmlns:x="urn:schemas-microsoft-com:office:excel"
  xmlns:ss="urn:schemas-microsoft-com:office:spreadsheet"
  xmlns:html="http://www.w3.org/TR/REC-html40">
  <Worksheet ss:Name="Sheet1">
    <Table>
      <Row>
        <Cell><Data ss:Type="String">ID</Data></Cell>
        <Cell><Data ss:Type="String">Name</Data></Cell>
        <Cell><Data ss:Type="String">Gender</Data></Cell>
        <Cell><Data ss:Type="String">Age</Data></Cell>
      </Row>
    <% @users.each do |user| %>
      <Row>
        <Cell><Data ss:Type="Number"><%= user.id %></Data></Cell>
        <Cell><Data ss:Type="String"><%= user.name %></Data></Cell>
        <Cell><Data ss:Type="String"><%= user.human_gender %></Data></Cell>
        <Cell><Data ss:Type="Number"><%= user.age %></Data></Cell>
      </Row>
    <% end %>
    </Table>
  </Worksheet>
</Workbook>
~~~

使用浏览器访问 http://localhost:3000/users.xls, 我们将下载到一个 xls 文件: users.xls。

![users.xls](/images/Snip20150912_16.png)


## format.pdf

我们使用 [prawn](https://github.com/prawnpdf/prawn) 和 [prawn-table](https://github.com/prawnpdf/prawn-table) 两个 gem 共同工作来生成 PDF。

~~~ruby
# Gemfile

+ gem 'prawn'
+ gem 'prawn-table'

~~~

users 控制器代码,

~~~ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController

  def index
    @users = User.order(:name)

    respond_to do |format|
      format.html
      format.csv
      format.xls
+     # 将响应头的 Content-Disposition 设置为 attachment
+     # 这样可以确保浏览器会主动下载 PDF 文档
+     format.pdf {response.headers['Content-Disposition'] = 'attachment'}
    end
  end
  
end
~~~

我们注意到在 format.pdf 的 block 中我们将响应头的 Content-Disposition 设置为了 attachment 这样可以确保浏览器会主动下载 PDF 文档。

建立视图文件 index.pdf.ruby, 这个视图文件的内容其实就是一段 ruby 代码, 并且这个视图最后会直接由 ruby 解释器处理。


~~~ruby
# app/views/users/index.pdf.ruby

pdf = Prawn::Document.new

pdf.font "#{Rails.root}/public/fonts/Arial-Unicode.ttf"

pdf.text "用户列表", size: 30

table_data = []
headers = ['ID', '姓名', '性别', '年龄']
table_data << headers
items = @users.map {|user|
  table_data << [user.id, user.name, user.human_gender, user.age]
}


pdf.table table_data

pdf.render

~~~

我们需要用到字体文件 Arial-Unicode.ttf, 所以我们从字体库中将该字体文件拷贝到我们项目
的 public/fonts 目录下:

~~~bash
cp /Library/Fonts/Arial\ Unicode.ttf public/fonts/Arial-Unicode.ttf 
~~~

users.pdf 的内容,

![users.pdf](/images/Snip20150913_17.png)


## format.json

我们使用 [jbuilder](https://github.com/rails/jbuilder) 来渲染 json 模版。


users 控制器代码,

~~~ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController

  def index
    @users = User.order(:name)

    respond_to do |format|
      format.html 
      format.csv
      format.xls
      # 将响应头的 Content-Disposition 设置为 attachment
      # 这样浏览器会主动下载 PDF 文档
      format.pdf {response.headers['Content-Disposition'] = 'attachment'}
+     format.json
    end
  end
  
end
~~~

建立 index.json.jbuilder 模版,

~~~ruby
# app/views/users/index.json.jbuilder

json.users @users, :id, :name, :human_gender, :age
~~~

使用浏览器访问 http://localhost:3000/users.json, 我们将得到:

~~~javascript
{
    users: [
	{
	    id: 3,
	    name: "小军",
	    human_gender: "男",
	    age: 25
	},
	{
	    id: 1,
	    name: "小明",
	    human_gender: "男",
	    age: 22
	},
	{
	    id: 2,
	    name: "小红",
	    human_gender: "女",
	    age: 21
	},
	{
	    id: 4,
	    name: "小芸",
	    human_gender: "女",
	    age: 24
	}
    ]
}
~~~


## format.xml

我们使用 [builder](https://github.com/jimweirich/builder) 来渲染 xml 模版。


users 控制器代码,

~~~ruby
# app/controllers/users_controller.rb

class UsersController < ApplicationController

  def index
    @users = User.order(:name)

    respond_to do |format|
      format.html 
      format.csv
      format.xls
      # 将响应头的 Content-Disposition 设置为 attachment
      # 这样浏览器会主动下载 PDF 文档
      format.pdf {response.headers['Content-Disposition'] = 'attachment'}
      format.json
+     format.xml
    end
  end
  
end

~~~


建立 index.xml.builder,

~~~ruby
# app/views/users/index.xml.builder

xml.instruct!
xml.users do
  @users.each do |user|
    xml.user do
      xml.id     user.id
      xml.name   user.name
      xml.gender user.human_gender
      xml.age    user.age
    end
  end
end
~~~

使用浏览器访问 http://localhost:3000/users.xml, 我们可以得到,

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<users>
  <user>
    <id>3</id>
    <name>小军</name>
    <gender>男</gender>
    <age>25</age>
  </user>
  <user>
    <id>1</id>
    <name>小明</name>
    <gender>男</gender>
    <age>22</age>
  </user>
  <user>
    <id>2</id>
    <name>小红</name>
    <gender>女</gender>
    <age>21</age>
  </user>
  <user>
    <id>4</id>
    <name>小芸</name>
    <gender>女</gender>
    <age>24</age>
  </user>
</users>
~~~


## 小结

1. 视图模版的命名规则是, :action.:format.:handler, 比如 index.html.erb 表示
   模版对应的 action 是 index, 响应格式是 html, 模版处理器是 erb。
   
2. 利用好 Rails 提供的 MVC(Model-View-Controller) 架构能够让程序变的优雅且易于维护。   
