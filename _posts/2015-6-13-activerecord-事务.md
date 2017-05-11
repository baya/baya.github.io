---
layout: post
title: ActiveRecord 事务的一些试验和验证
---

> Transactions are protective blocks where SQL statements are only permanent if they can all succeed as one atomic action

我们平时说的 一手交钱，一手交货 就是一种事务。

ActiveRecord 中的事务是通过 `transaction` 方法实现的。

我们首先试验一个基础的例子:

## ActiveRecord::Base.transaction

~~~ruby

ActiveRecord::Base.transaction do
  david.withdrawal(100)
  mary.deposit(100)
end

~~~

在试验之前，我们先用 Rails 创建一个 demo, 并创建好相关的 model。

~~~bash
$ rails new ar-transaction-demo
~~~

建立 Account 模型:

~~~bash
$ bundle exe rails g model Account
~~~

~~~ruby
# 修改 db/migrate/20150614031226_create_accounts.rb
class CreateAccounts < ActiveRecord::Migration
  def change
    create_table :accounts do |t|
   +  t.string :name
   +  t.float :money, default: 0

      t.timestamps null: false
    end
  end
end
~~~

执行 migrate:

~~~bash
$ bundle exe rake db:migrate
~~~

创建种子数据:

~~~ruby
# db/seeds.rb

+ Account.create([{name: 'david', money: 1999},
+                {name: 'mary', money: 899}
+               ])

~~~

加载种子数据:

~~~bash
$ bundle exe rake db:seed
~~~

实现 Account#withdrawal 和 Account#deposit 两个方法:

~~~ruby
# app/models/account.rb

class Account < ActiveRecord::Base

+  def withdrawal(money_num)
+     decrement!(:money, money_num)
+  end

+  def deposit(money_num)
+    increment!(:money, money_num)
+  end
  
end

~~~

建立一个 rake task: debug\_transaction:ar\_base 来运行试验:

~~~ruby
# lib/tasks/debug_transaction.rake

namespace :debug_transaction do

+  task ar_base: ['environment'] do
    
+    david = Account.find_by(name: 'david')
+    mary = Account.find_by(name: 'mary')

+    puts "before: david:#{david.money}, mary:#{mary.money}"
    
+    ActiveRecord::Base.transaction do
+      david.withdrawal(100)
+      mary.deposit(100)
+    end

+    puts "after: david:#{david.money}, mary:#{mary.money}"
    
+  end

end

~~~

运行 debug\_transaction:ar\_base task,

~~~bash
$ bundle exe rake debug_transaction:ar_base

before: david:1999.0, mary:899.0
after: david:1899.0, mary:999.0
~~~

在正常情况下, david 的钱增加了 100, mary 的钱减少了 100, 现在我们给 Account#deposit 方法

引入一个异常,

~~~ruby

class Account < ActiveRecord::Base

  def withdrawal(money_num)
    increment!(:money, money_num)
  end

  def deposit(money_num)
    decrement!(:money, money_num)
+	raise 'deposit fail'
  end
  
end

~~~

运行 debug\_transaction:ar\_base task,

~~~bash
$ bundle exe rake debug_transaction:ar_base

before: david:1899.0, mary:999.0
rake aborted!
deposit fail
~~~

进入 rails console 查看 david 和 mary 的钱是否发生了变化,

~~~ruby
david = Account.find_by(name: 'david')
mary = Account.find_by(name: 'mary')

david.money #=> 1899.0
mary.money #=> 999.0
~~~

我们看到 david 和 mary 的钱没有发生变化，说明 ActiveRecord::Base.transaction 起到了
事务的作用。

我们也可以在日志文件 log/development.log 里找到和 transaction 相关的记录,


~~~text
(0.1ms)  begin transaction
 SQL (0.5ms)  UPDATE "accounts" SET "money" = ?, "updated_at" = ? WHERE "accounts"."id" = ?  [["money", 2199.0], ["updated_at", "2015-06-14 14:30:45.901253"], ["id", 1]]
 SQL (0.4ms)  UPDATE "accounts" SET "money" = ?, "updated_at" = ? WHERE "accounts"."id" = ?  [["money", 699.0], ["updated_at", "2015-06-14 14:30:45.904365"], ["id", 2]]
(2.7ms)  rollback transaction
~~~

如果我们在 Account#withdrawal 方法里引入异常，david 和 mary 的钱也不会发生变化，


~~~ruby

class Account < ActiveRecord::Base

  def withdrawal(money_num)
    decrement!(:money, money_num)
+   raise 'withdrawal fail'
  end

  def deposit(money_num)
    increment!(:money, money_num)
+ # raise 'deposit fail'
  end
  
end

~~~


## Different Active Record classes in a single transaction

在单个事务里包含不同的 Active Record 类, 这时候事务机制会发生作用吗? 为了验证这一想法，我们做
下面的 3 个试验。

1. 由 ActiveRecord::Base 的子类调用 transaction 方法

	~~~ruby
	Account.transaction do
	  balance.save!
	  account.save!
	end
	~~~

2.  由 ActiveRecord::Base 的子类的实例调用 transaction 方法

	~~~ruby
	balance.transaction do
	  balance.save!
	  account.save!
	end
	~~~

3. 由 ActiveRecord::Base 类调用 transaction 方法

~~~ruby
ActiveRecord::Base.transaction do
balance.save!
account.save!
end
~~~


为了进行上面的 3 个试验，我们需要增加一个新的模型: Balance

~~~bash
$ bundle exe rails g model Balance
~~~

~~~ruby
# db/migrate/20150620022444_create_balances.rb

class CreateBalances < ActiveRecord::Migration
  def change
    create_table :balances do |t|
      t.integer :account_id
      t.float :amount

      t.timestamps null: false
    end
  end
end

~~~

~~~bash
$ bundle exe rake db:migrate
~~~

我们首先进行第 1 个试验:

~~~ruby
# rails console

account = Account.find_by_name('david')

balance = Balance.create(account_id: account.id, amount: 100)
balance.id #=> 1
~~~


~~~ruby
# lib/tasks/debug_transaction.rake

namespace :debug_transaction do

+  task single_transaction_01: ['environment'] do
+     account = Account.find_by_name('david')
+     balance = Balance.find(1)

+     puts "before balance.amount: #{balance.amount}"
+     puts "before account.money: #{account.money}"

+     Account.transaction do
+       balance.amount = 200
+       balance.save!
+       account.money = 300
+       account.save!
+       raise 'debug transaction 01'
     end

+  end

end
~~~


~~~bash
$ bundle exe rake debug_transaction:single_transaction_01

before balance.amount: 100.0
before account.money: 1899.0
rake aborted!
~~~


进入 rails console 查看,

~~~ruby
# bundle exe rails c
account = Account.find_by_name('david')
account.money #=> 1899.0

balance = Balance.find(1)
balance.amount #=> 100.0

~~~

从结果来看, 可以验证 ActiveRecord::Base 的子类调用 transaction 方法，也可以达到事务的作用。

试验 2 和 试验 3的过程和试验 1 相近，这里不再赘述, 只提供代码如下:

试验 2 相关代码,

~~~ruby
lib/tasks/debug_transaction.rake

namespace :debug_transaction do

+  task single_transaction_02: ['environment'] do
+    account = Account.find_by_name('david')
+    balance = Balance.find(1)

+    puts "before balance.amount: #{balance.amount}"
+    puts "before account.money: #{account.money}"

+    balance.transaction do
+      balance.amount = 200
+      balance.save!
+      account.money = 300
+      account.save!
+      raise 'debug transaction 01'
+    end

+  end

end
~~~

试验 3 相关代码,


~~~ruby
namespace :debug_transaction do

+  task single_transaction_03: ['environment'] do
+    account = Account.find_by_name('david')
+    balance = Balance.find(1)

+    puts "before balance.amount: #{balance.amount}"
+    puts "before account.money: #{account.money}"

+    ActiveRecord::Base.transaction do
+      balance.amount = 200
+      balance.save!
+      account.money = 300
+      account.save!
+      raise 'debug transaction 01'
+    end

+  end

end

~~~


## Transactions are not distributed across database connections

> 一个事务只能在一个数据库连接上起作用

我们先建立 Student, Course 两个模型,

~~~bash
$ bundle exe rails g model Student

$ bundle exe rails g model Course
~~~

Student 和 Course 之间是多对多的关系，所以我们还需要建立一个中间模型: StudentCourse

~~~bash
$ bundle exe rails g model StudentCourse
~~~

~~~ruby
# db/migrate/20150622080544_create_students.rb

class CreateStudents < ActiveRecord::Migration
  def change
    create_table :students do |t|
+     t.string :name
+     t.integer :units, default: 0
      t.timestamps null: false
    end
  end
end

~~~

~~~ruby
db/migrate/20150622080604_create_courses.rb

class CreateCourses < ActiveRecord::Migration
  def change
    create_table :courses do |t|
+     t.string :name
+     t.integer :units, default: 0
      t.timestamps null: false
    end
  end
end

~~~

~~~ruby
# db/migrate/20150622080837_create_student_courses.rb

class CreateStudentCourses < ActiveRecord::Migration
  def change
    create_table :student_courses do |t|
+     t.integer :student_id
+     t.integer :course_id
      
      t.timestamps null: false
    end
  end
end

~~~


~~~ruby
# app/models/student.rb

class Student < ActiveRecord::Base
 + has_many :student_courses
 + has_many :courses, through: 'student_courses'
end
~~~


~~~ruby
class StudentCourse < ActiveRecord::Base
+  belongs_to :student
+  belongs_to :course
end

~~~


~~~ruby
# app/models/course.rb

class Course < ActiveRecord::Base
+  has_many :student_courses
+  has_many :students, through: 'student_courses'
end
~~~

我们将给 Student, StudentCourse, Course 分配不同的数据库连接, 为此我们需要建立三个数据库,

~~~bash
$ cp config/environments/development.rb config/environments/development1.rb 
$ cp config/environments/development.rb config/environments/development2.rb
$ cp config/environments/development.rb config/environments/development3.rb 
~~~

~~~yaml
# SQLite version 3.x
#   gem install sqlite3
#
#   Ensure the SQLite 3 gem is defined in your Gemfile
#   gem 'sqlite3'
#
default: &default
  adapter: sqlite3
  pool: 5
  timeout: 5000

development:
  <<: *default
  database: db/development.sqlite3
  
+ development1:
+  <<: *default
+  database: db/development1.sqlite3


+ development2:
+  <<: *default
+  database: db/development2.sqlite3


+ development3:
+  <<: *default
+  database: db/development3.sqlite3
  
~~~

~~~bash
$ RAILS_ENV=development1 bundle exe rake db:create

$ RAILS_ENV=development2 bundle exe rake db:create

$ RAILS_ENV=development3 bundle exe rake db:create
~~~

现在我们有 db/development1.sqlite3, db/development2.sqlite3, db/development3.sqlite3 三个数据库。

Student 和 db/development1.sqlite3 数据库连接,

~~~ruby
# app/models/student.rb

class Student < ActiveRecord::Base
+  establish_connection YAML.load_file("#{Rails.root}/config/database.yml")['development1']
end
~~~

StudentCourse 和 db/development2.sqlite3 数据库连接,

~~~ruby
# app/models/student_course.rb

class StudentCourse < ActiveRecord::Base
+  establish_connection YAML.load_file("#{Rails.root}/config/database.yml")['development2']
end
~~~

Course 和 db/development2.sqlite3 数据库连接,

~~~ruby
class Course < ActiveRecord::Base
+  establish_connection YAML.load_file("#{Rails.root}/config/database.yml")['development3']
end
~~~

数据迁移,

~~~bash
$ RAILS_ENV=development1 bundle exe rake db:migrate

$ RAILS_ENV=development2 bundle exe rake db:migrate

$ RAILS_ENV=development3 bundle exe rake db:migrate
~~~

实现 Course#enroll 方法,

~~~ruby
class Course < ActiveRecord::Base

+  def enroll(student)
+    self.students << student
+    save
+  end
  
end

~~~

首先我们只使用 Student.transaction 方法，


~~~ruby
# lib/tasks/debug_transaction.rake

namespace :debug_transaction do

  task distribute_01: ['environment'] do
  
    student = Student.find_by_name('Jim')
    course = Course.find_by_name('数学课')

    puts "before student.units: #{student.units}"
    puts "before student.units: #{student.units}"
    puts "before StudentCourse.all.to_a: #{StudentCourse.all.to_a}"

    Student.transaction do
      course.enroll(student)
      student.units += course.units
      student.save

      raise 'debug distribute 01 transactions'
    end
    
  end

end
~~~

准备一些验证数据,

~~~ruby
# bundle exe rails c

Student.create(name: 'Jim')
Course.create(name: '数学课', units: 20)

~~~


执行验证,

~~~bash
$ bundle exe rake debug_transaction:distribute_01

before student.units: 0
before StudentCourse.all.to_a: []
rake aborted!
debug distribute 01 transactions
~~~

我们查看下数据库是否发生变化,

~~~
# bundle exe rails c

student = Student.find_by_name('Jim')

student.units #=> 0
StudentCourse.all.to_a #=>  [#<StudentCourse id: 6, student_id: 6, course_id: 2, created_at: "2015-06-22 13:45:35", updated_at: "2015-06-22 13:45:35">]
~~~


我们看到 student.units 的值回滚到了 0, 但是 student_courses 表增加了一条记录:

~~~ruby
#<StudentCourse id: 6, student_id: 6, course_id: 2, created_at: "2015-06-22 13:45:35", updated_at: "2015-06-22 13:45:35">
~~~

这说明 Student.transaction 没有对 StudentCourse 起到事务的作用。


现在我们加入 StudentCourse.transaction 方法,

~~~ruby
# lib/tasks/debug_transaction.rake

  task distribute_02: ['environment'] do
    student = Student.find_by_name('Jim')
    course = Course.find_by_name('数学课')

    puts "before student.units: #{student.units}"
    puts "before StudentCourse.all.to_a: #{StudentCourse.all.to_a}"

    Student.transaction do
      StudentCourse.transaction do
        course.enroll(student)
        student.units += course.units
        student.save

        raise 'debug distribute 02 transactions'
      end
    end
    
  end

~~~

在试验前，我们将 student_courses 表清空,

~~~ruby
# bundle exe rails c

StudentCourse.destroy_all
~~~

~~~bash
$ bundle exe rake debug_transaction:distribute_02

before student.units: 0
before StudentCourse.all.to_a: []
rake aborted!
debug distribute 02 transactions
~~~

查看数据库是否发生变化,

~~~ruby
# bundle exe rails c

student = Student.find_by_name('Jim')

student.units #=> 0

StudentCourse.all.to_a #=> []
~~~

我们看到 student.units 的值没有发生变化还是 0, student_courses 表也没有产生新的记录， 

从而验证了 ActiveRecord 的一个事务只能在一个数据库连接上起作用。


## save and destroy are automatically wrapped in a transaction

为了验证这条规则，我们建立一个叫 AccountOpLog 的模型, 每次 account 调用 save 或者 destroy

方法时, 都会创建一条 account\_op\_log 记录.


~~~bash
$ bundle exe rails g model AccountOpLog
~~~

~~~ruby
# db/migrate/20150709144706_create_account_op_logs.rb

class CreateAccountOpLogs < ActiveRecord::Migration
  def change
    create_table :account_op_logs do |t|
+     t.integer :account_id
+     t.string  :action 
      t.timestamps null: false
    end
  end
end

~~~

~~~bash
$ bundle exe rake db:migrate
~~~

为 Account 模型设置相关的 callback,

~~~ruby
# app/models/account.rb
class Account < ActiveRecord::Base

+  after_save :create_op_log_for_save
+  after_destroy :create_op_log_for_destroy

  private
  
+  def create_op_log_for_save
+    AccountOpLog.create(action: 'save', account_id: self.id)
+  end

+  def create_op_log_for_destroy
+    AccountOpLog.create(action: 'destroy', account_id: self.id)
+  end
  
end

~~~

增加一个叫 debug\_transaction:for\_save 的 task:

~~~ruby
# lib/tasks/debug_transaction.rake
namespace :debug_transaction do
+  task for_save: ['environment'] do
+    Account.create(name: 'kyk01')
+  end
end
~~~

执行:

~~~bash
$ bundle exe rake debug_transaction:for_save
~~~

我们会发现数据库创建了:

~~~ruby
#<Account id: 7, name: "kyk01", money: 0.0, created_at: "2015-07-09 15:31:55", updated_at: "2015-07-09 15:31:55">

#<AccountOpLog id: 1, account_id: 7, action: "save", created_at: "2015-07-09 15:31:55", updated_at: "2015-07-09 15:31:55">
~~~

这说明 Account 模型的 after_save 回调起作用了。

现在我们给 after_save 回调引入一个异常:

~~~ruby
# app/models/account.rb

class Account < ActiveRecord::Base

  after_save :create_op_log_for_save

  private
  
  def create_op_log_for_save
    AccountOpLog.create(action: 'save', account_id: self.id)
+   raise 'boom!'
  end

end

~~~

同时我们在 debug\_transaction:for\_save task 里创建一个新的 account,

~~~ruby
# lib/tasks/debug_transaction.rake
namespace :debug_transaction do
  task for_save: ['environment'] do
+   # Account.create(name: 'kyk01')
+   Account.create(name: 'kyk02')
  end
end
~~~

执行:

~~~bash
$ bundle exe rake debug_transaction:for_save

rake aborted!
boom!

~~~

有异常抛出，我们查看下日志，有事务回滚,

~~~bash
 (0.1ms)  begin transaction
  SQL (0.5ms)  INSERT INTO "accounts" ("name", "created_at", "updated_at") VALUES (?, ?, ?)  [["name", "kyk02"], ["created_at", "2015-07-09 15:38:55.350623"], ["updated_at", "2015-07-09 15:38:55.350623"]]
  SQL (0.2ms)  INSERT INTO "account_op_logs" ("action", "account_id", "created_at", "updated_at") VALUES (?, ?, ?, ?)  [["action", "save"], ["account_id", 8], ["created_at", "2015-07-09 15:38:55.360222"], ["updated_at", "2015-07-09 15:38:55.360222"]]
   (2.6ms)  rollback transaction
~~~

并且我们发现数据库里没有创建出 name 为 'kyk02' 的 account  以及相关的 account\_op\_log,

~~~ruby
Account.find_by(name: 'kyk02') #=> nil
AccountOpLog.find_by(id: 2) #=> nil
~~~

这验证了,

> save 方法被包裹在事务中

现在我们验证 destroy 方法.


~~~ruby
# app/models/account.rb

class Account < ActiveRecord::Base
  def create_op_log_for_save
    AccountOpLog.create(action: 'save', account_id: self.id)
+   # raise 'boom!'
  end
end
~~~


~~~ruby
# lib/tasks/debug_transaction.rake
namespace :debug_transaction do 
+ task for_destroy: ['environment'] do
+   account = Account.find_by(name: 'kyk03')
+   account ||= Account.create(name: 'kyk03')
+   account.destroy
+  end
end
~~~

执行:

~~~bash
$ bundle exe rake debug_transaction:for_destroy
~~~

~~~ruby
Account.find_by(name: 'kyk03') #=> nil
Account.last #=> <AccountOpLog id: 3, account_id: 8, action: "destroy", created_at: "2015-07-11 00:35:49", updated_at: "2015-07-11 00:35:49">
~~~

引入异常,

~~~ruby
class Account < ActiveRecord::Base

  def create_op_log_for_destroy
    AccountOpLog.create(action: 'destroy', account_id: self.id)
+   raise 'boom!'
  end

end
~~~

执行:

~~~bash
$ bundle exe rake debug_transaction:for_destroy

rake aborted!
boom!
~~~

~~~ruby
Account.find_by(name: 'kyk03') #=> #<Account id: 9, name: "kyk03", money: 0.0, created_at: "2015-07-11 00:40:44", updated_at: "2015-07-11 00:40:44">

AccountOpLog.last #=> #<AccountOpLog id: 4, account_id: 9, action: "save", created_at: "2015-07-11 00:40:44", updated_at: "2015-07-11 00:40:44">
~~~

并且我们在日志中查到:

~~~bash
(0.1ms)  begin transaction
  SQL (0.5ms)  DELETE FROM "accounts" WHERE "accounts"."id" = ?  [["id", 9]]
  SQL (0.1ms)  INSERT INTO "account_op_logs" ("action", "account_id", "created_at", "updated_at") VALUES (?, ?, ?, ?)  [["action", "destroy"], ["account_id", 9], ["created_at", "2015-07-11 00:40:44.879227"], ["updated_at", "2015-07-11 00:40:44.879227"]]
   (0.6ms)  rollback transaction
~~~

我们发现 id 为 9 的 account 没有被删除，并且也没有创建和 destroy 相关的 account\_op\_log, 这说明:

> destroy 方法被包裹在事务中

## Exception handling and rolling back

我们需要注意三点:

1. 普通的异常在 transaction block 里触发 ROLLBACK 后会向 block 外传播;

2. ActiveRecord::Rollback 触发 ROLLBACK 后不会向 block 外传播;

3. transaction block 中捕捉不到 ActiveRecord::RecordInvalid 异常, 即使你加了

rescue 语句;

第 1 点在前面的试验中，已经多次验证过了，我们现在验证第 2 点。

~~~ruby
# lib/tasks/debug_transaction.rake
namespace :debug_transaction do
+ task for_rollback_exception: ['environment'] do
+  begin
+    Number.transaction do
+      Number.create(i: 0)
+      Number.create(i: 1)
+      raise ActiveRecord::Rollback
+    end
+  rescue Exception => e
+    puts e.message
+  end

+  end
end

~~~

建立 Number 模型,

~~~bash
$ bundle exe rails g model Number
~~~

~~~ruby
# db/migrate/20150711055433_create_numbers.rb

class CreateNumbers < ActiveRecord::Migration
  def change
    create_table :numbers do |t|
+     t.integer :i
      t.timestamps null: false
    end
  end
end

~~~

~~~ruby
# app/models/number.rb
class Number < ActiveRecord::Base

+ validates :i, uniqueness: true
  
end

~~~

数据迁移,

~~~bash
$ bundle exe rake db:migrate
~~~

执行,

~~~bash
$ bundle exe rake debug_transaction:for_rollback_exception
~~~

我们看到没有异常信息的输出，这说明 ActiveRecord::Rollback 异常没有传递出 transaction block, 并且,

~~~ruby
Number.find_by(i: 0) #=> nil
Number.find_by(i: 1) #=> nil
~~~

这说明 ActiveRecord::Rollback 触发了 transaction rollback.

接下来我们抛出一个普通的异常试试,

~~~ruby
# lib/tasks/debug_transaction.rake
namespace :debug_transaction do
  task for_rollback_exception: ['environment'] do
    begin
      Number.transaction do
        Number.create(i: 0)
        Number.create(i: 1)
+       # raise ActiveRecord::Rollback
+		raise 'boom!'
      end
    rescue Exception => e
      puts e.message
    end

  end
end  
~~~

执行,

~~~bash
bundle exe rake debug_transaction:for_rollback_exception

boom!
~~~

此时输出了异常消息: boom!, 这说明普通异常传递出了 transaction block, 并且,

~~~ruby
Number.find_by(i: 0) #=> nil
Number.find_by(i: 1) #=> nil
~~~

试验到了这一步，我们再顺手做一个试验,

~~~ruby
# lib/tasks/debug_transaction.rake
namespace :debug_transaction do

+  task for_create: ['environment'] do

+   Number.transaction do
+      Number.create(i: 0)
+      Number.create(i: 0)
+    end
    
+  end

end
~~~

我们在 Account 模型中对 i 做了唯一性验证,


~~~ruby
class Number < ActiveRecord::Base

+  validates :i, uniqueness: true
  
end
~~~

执行,

~~~bash
$ bundle exe rake debug_transaction:for_create
~~~

没有异常抛出，并且,

~~~ruby
Number.where(i: 0).to_a #=> [#<Number id: 1, i: 0, created_at: "2015-07-11 06:28:00", updated_at: "2015-07-11 06:28:00">]
~~~

> 这说明只有引发异常，才能触发 transaction rollback.

我们验证第 3 点。我们首先需要配置 PostgreSQL,

~~~yaml
# config/database.yml

+ development_pg:
+  adapter: postgresql
+  encoding: unicode
+  pool: 5
+  username: transaction_tester
+  password:
+  host: localhost
+  database: transaction_demo
~~~

~~~bash
# 创建 非超级用户，可登陆，需要密码，可创建数据库 的用户 transaction_tester
# 当提示需要输入密码时，我们按 enter 键就行
$ createuser transaction_tester -S -l -P -d

# 创建数据库 transaction_demo, 并且此数据库的拥有者是 transaction_tester
$ createdb transaction_demo -O transaction_tester

# 连接数据库
$ psql -d transaction_demo -U transaction_tester

# 创建 numbers 表

psql (9.3.5)
Type "help" for help.

transaction_demo=> create table numbers(
  id serial primary key,
  i integer NOT NULL UNIQUE,
  created_at timestamp without time zone,
  updated_at timestamp without time zone
);

~~~

在 Gemfile 里加入 pg,

~~~ruby
+ gem 'pg'
~~~

执行,

~~~bash
$ bundle install
~~~

更改 Number 的数据库连接,

~~~ruby
# app/models/number.rb

class Number < ActiveRecord::Base

+  pg_db = YAML.load_file("#{Rails.root}/db/database.yml")['development_pg']
+  self.establish_connection(pg_db)

  validates :i, uniqueness: true
  
end

~~~

新建任务,

~~~ruby
# lib/tasks/debug_transaction.rake

namespace :debug_transaction do

+  task for_pg: ['environment'] do
+    # Suppose that we have a Number model with a unique column called 'i'.
+    Number.transaction do
+      Number.create(i: 2)
+      begin
+        # This will raise a unique constraint error...
+        Number.create!(i: 2)
+      rescue ActiveRecord::StatementInvalid => e
+        puts e.message
+      end

+      # On PostgreSQL, the transaction is now unusable. The following
+      # statement will cause a PostgreSQL error, even though the unique
+      # constraint is no longer violated:
+      Number.create(i: 3)
+      # => "PGError: ERROR:  current transaction is aborted, commands
+      #     ignored until end of transaction block"
+    end
+  end

end

~~~

我们在 numbers 表中已经对 'i' 做了唯一性约束，

~~~sql
i integer NOT NULL UNIQUE,
~~~

同时我们将 Number 模型里的 validates 去掉, 否则试验的时候会抛出 ActiveRecord::RecordInvalid 异常而不是 ActiveRecord::StatementInvalid 异常。

~~~ruby
# app/models/number.rb

class Number < ActiveRecord::Base

  pg_db = YAML.load_file("#{Rails.root}/config/database.yml")['development_pg']
  mysql_db = YAML.load_file("#{Rails.root}/config/database.yml")['development_mysql']
  self.establish_connection(pg_db)
  # self.establish_connection(mysql_db)

+  # validates :i, uniqueness: true
  
end

~~~

执行任务,

~~~bash
$ bundle exe rake debug_transaction:for_pg

PG::UniqueViolation: ERROR:  duplicate key value violates unique constraint "numbers_i_key"
DETAIL:  Key (i)=(3) already exists.
: INSERT INTO "numbers" ("i", "created_at", "updated_at") VALUES ($1, $2, $3) RETURNING "id"
rake aborted!
ActiveRecord::StatementInvalid: PG::InFailedSqlTransaction: ERROR:  current transaction is aborted, commands ignored until end of transaction block
~~~

我们看到当加了 rescue 语句后捕捉 ActiveRecord::StatementInvalid 时，会导致 PG::UniqueViolation: ERROR。

我们再试验下使用 rescue Exception 看捕捉到 ActiveRecord::InvalidStatement 异常后会发生什么错误

~~~ruby
lib/tasks/debug_transaction.rake

namespace :debug_transaction do

+  task for_pg_2: ['environment'] do
    # Suppose that we have a Number model with a unique column called 'i'.
+    Number.transaction do
+      Number.create(i: 6)
+      begin
        # This will raise a unique constraint error...
+        Number.create!(i: 6)
+      rescue Exception => e
+        puts "+++++++++++++++"
+        puts e.message
+      end

      # On PostgreSQL, the transaction is now unusable. The following
      # statement will cause a PostgreSQL error, even though the unique
      # constraint is no longer violated:
+      Number.create(i: 7)
      # => "PGError: ERROR:  current transaction is aborted, commands
      #     ignored until end of transaction block"
+    end
+  end

end

~~~

执行,

~~~bash
$ bundle exe rake debug_transaction:for_pg_2

+++++++++++++++
PG::UniqueViolation: ERROR:  duplicate key value violates unique constraint "numbers_i_key"
DETAIL:  Key (i)=(6) already exists.
: INSERT INTO "numbers" ("i", "created_at", "updated_at") VALUES ($1, $2, $3) RETURNING "id"
rake aborted!
ActiveRecord::StatementInvalid: PG::InFailedSqlTransaction: ERROR:  current transaction is aborted, commands ignored until end of transaction block
: INSERT INTO "numbers" ("i", "created_at", "updated_at") VALUES ($1, $2, $3) RETURNING "id"
~~~

结果和前面使用 rescue ActiveRecord::StatementInvalid 一样会导致错误。

我们把数据库换为 sqlite 试验下,

~~~ruby
# app/models/number.rb
class Number < ActiveRecord::Base

  pg_db = YAML.load_file("#{Rails.root}/config/database.yml")['development_pg']
 + # self.establish_connection(pg_db)

 + # validates :i, uniqueness: true
  
end

~~~

将 debug_transaction:for_pg task 改动下,

~~~ruby
  task for_pg: ['environment'] do
    # Suppose that we have a Number model with a unique column called 'i'.
    Number.transaction do
 +     Number.create(i: 6)
      begin
        # This will raise a unique constraint error...
 +        Number.create!(i: 6)
      rescue ActiveRecord::StatementInvalid => e
 +       puts "+++++++++++++++"
        puts e.message
      end

      # On PostgreSQL, the transaction is now unusable. The following
      # statement will cause a PostgreSQL error, even though the unique
      # constraint is no longer violated:
 +     Number.create(i: 7)
      # => "PGError: ERROR:  current transaction is aborted, commands
      #     ignored until end of transaction block"
    end
  end

~~~

执行,

~~~bash
bundle exe rake debug_transaction:for_pg


+++++++++++++++
SQLite3::ConstraintException: column i is not unique: INSERT INTO "numbers" ("i", "created_at", "updated_at") VALUES (?, ?, ?)
~~~

也就是说在 sqlite 中是可以捕捉 ActiveRecord::StatementInvalid 异常的。

我们再把数据库设置为 mysql 试验下, 试验前先配置下 mysql,

~~~yaml
# config/database.yml

development_mysql:
  adapter: mysql2
  encoding: utf8
  pool: 5
  username: tr_tester
  password: 
  host: localhost
  database: transaction_demo
  
~~~

创建相关用户和数据库,

~~~bash
$ mysql -h localhost -u root

mysql> CREATE USER 'tr_tester'@'localhost';
mysql> GRANT ALL PRIVILEGES ON * . * TO 'tr_tester'@'localhost';
mysql> CREATE DATABASE transaction_demo CHARACTER SET utf8;
mysql> use transaction_demo;
mysql> CREATE TABLE numbers (
  id int(11) auto_increment PRIMARY KEY ,
  i int,
  created_at datetime NOT NULL,
  updated_at datetime NOT NULL 
);

mysql> ALTER TABLE numbers ADD UNIQUE(i);
~~~


增加 gem mysql2,

~~~
# Gemfile

+ gem 'mysql2'

~~~

~~~bash
$ bundle install
~~~

将 number 模型连接到 mysql, 并且注释掉掉 validates 验证,

~~~ruby
# app/models/number.rb

class Number < ActiveRecord::Base

  pg_db = YAML.load_file("#{Rails.root}/config/database.yml")['development_pg']
+  mysql_db = YAML.load_file("#{Rails.root}/config/database.yml")['development_mysql']
  # self.establish_connection(pg_db)
+  self.establish_connection(mysql_db)

+ # validates :i, uniqueness: true
  
end

~~~

执行,

~~~bash
bundle exe rake debug_transaction:for_pg


+++++++++++++++
Mysql2::Error: Duplicate entry '6' for key 'i': INSERT INTO `numbers` (`i`, `created_at`, `updated_at`) VALUES (6, '2015-07-19 04:40:23.121202', '2015-07-19 04:40:23.121202')
~~~

这说明在 mysql 中可以捕捉到 ActiveRecord::StatementInvalid 类的异常。

## Nested transactions

嵌套的事务之间会形成一种父子关系, 子事务的异常如果没有传递到父事务，则整个事务会失效，我们可以通过下面的一些试验来加深对此处的理解。

我们使用 mysql 数据库进行试验, 首先创建相关的环境文件:

~~~bash
$ cp cp config/environments/development.rb config/environments/development_mysql.rb 
~~~

创建 User 模型,

~~~bash
$ bundle exe rails g model User
~~~

~~~ruby
# db/migrate/20150719124654_create_users.rb

class CreateUsers < ActiveRecord::Migration
  def change
    create_table :users do |t|
      t.string   :name
      t.timestamps null: false
    end
  end
end

~~~

数据迁移, 因为我们前面通过 sql 直接创建了 numbers 表，所以我们先将 numbers 表删除,

~~~bash
mysql > drop table numbers;
~~~

~~~bash
$ RAILS_ENV=development_mysql bundle exe rake db:migrate
~~~

### 试验1:

~~~ruby
# lib/tasks/debug_transaction.rake

namespace :debug_transaction do
+  task for_nested_1: ['environment'] do
+    User.transaction do
+      User.create(name: 'Kotori')
+      User.transaction do
+        User.create(name: 'Nemu')
+        raise ActiveRecord::Rollback
+      end
+    end
+  end
end
~~~

~~~bash
$ RAILS_ENV=development_mysql bundle exe rake debug_transaction:for_nested_1
~~~

查询 users,

~~~bash
mysql> select * from users;
+----+--------+---------------------+---------------------+
| id | name   | created_at          | updated_at          |
+----+--------+---------------------+---------------------+
|  1 | Kotori | 2015-07-19 12:55:45 | 2015-07-19 12:55:45 |
|  2 | Nemu   | 2015-07-19 12:55:45 | 2015-07-19 12:55:45 |
+----+--------+---------------------+---------------------+
2 rows in set (0.00 sec)

~~~

从查询结果来看，事务没有被触发，这是因为 ActiveRecord::Rollback 没有传递到最外层的父事务中去。

### 试验 2

~~~ruby
# lib/tasks/debug_transaction.rake

namespace :debug_transaction do
+  User.transaction do
+    User.create(name: 'Kotori2')
+    User.transaction do
+      User.create(name: 'Nemu2')
+      raise 'boom!'
+    end
+ end
end

~~~

~~~bash
$ RAILS_ENV=development_mysql bundle exe rake debug_transaction:for_nested_2

rake aborted!
boom!
~~~

查询 users,

~~~bash
mysql> select * from users;
+----+--------+---------------------+---------------------+
| id | name   | created_at          | updated_at          |
+----+--------+---------------------+---------------------+
|  1 | Kotori | 2015-07-19 12:55:45 | 2015-07-19 12:55:45 |
|  2 | Nemu   | 2015-07-19 12:55:45 | 2015-07-19 12:55:45 |
+----+--------+---------------------+---------------------+
2 rows in set (0.00 sec)

~~~

从查询结果来看，事务被触发，这是因为 raise 'boom!' 传递到最外层的父事务中去了。

### 试验 3

~~~ruby
# lib/tasks/debug_transaction.rake

namespace :debug_transaction do
+  task for_nested_3: ['environment'] do
+    User.transaction do
+      User.create(name: 'Kotori3')
+      User.transaction(requires_new: true) do
+        User.create(name: 'Nemu3')
+        raise ActiveRecord::Rollback
+      end
+    end
+  end
end

~~~

我们在子事务中加了一个 `requires_new: true` 这有什么作用呢? 我们测试一下就知道了。

~~~bash
$ RAILS_ENV=development_mysql bundle exe rake debug_transaction:for_nested_3
~~~

查询 users,

~~~bash
mysql> select * from users;
+----+---------+---------------------+---------------------+
| id | name    | created_at          | updated_at          |
+----+---------+---------------------+---------------------+
|  1 | Kotori  | 2015-07-19 12:55:45 | 2015-07-19 12:55:45 |
|  2 | Nemu    | 2015-07-19 12:55:45 | 2015-07-19 12:55:45 |
|  5 | Kotori3 | 2015-07-19 13:07:12 | 2015-07-19 13:07:12 |
+----+---------+---------------------+---------------------+
3 rows in set (0.00 sec)

~~~

我们发现 Kotori3 被创建了, Nemu3 没有被创建, 也就是说在没有回滚到父事务的情况下子事务被触发了, `requires_new: true` 
的作用就在于此。

### 试验 4

~~~ruby
# lib/tasks/debug_transaction.rake

namespace :debug_transaction do
+  task for_nested_4: ['environment'] do
+    User.transaction do
+      User.create(name: 'Kotori4')
+      begin
+        User.transaction do
+          User.create(name: 'Nemu4')
+          raise 'boom!'
+        end
+      rescue Exception => e
+        puts "!!!!!!!!!!!!!"
+        puts e.message
+      end
+    end
+  end
end

~~~

~~~bash
RAILS_ENV=development_mysql bundle exe rake debug_transaction:for_nested_4

!!!!!!!!!!!!!
boom!
~~~

查询 users,

~~~bash
mysql> select * from users;
+----+---------+---------------------+---------------------+
| id | name    | created_at          | updated_at          |
+----+---------+---------------------+---------------------+
|  1 | Kotori  | 2015-07-19 12:55:45 | 2015-07-19 12:55:45 |
|  2 | Nemu    | 2015-07-19 12:55:45 | 2015-07-19 12:55:45 |
|  5 | Kotori3 | 2015-07-19 13:07:12 | 2015-07-19 13:07:12 |
|  7 | Kotori4 | 2015-07-19 13:31:58 | 2015-07-19 13:31:58 |
|  8 | Nemu4   | 2015-07-19 13:31:58 | 2015-07-19 13:31:58 |
+----+---------+---------------------+---------------------+
5 rows in set (0.00 sec)
~~~

和试验 1的结果一样，事务没有被触发, Kotori4 和 Nemu4 都被创建了, 这说明异常如果没有到达
父事务，那么整个父事务和子事务都不会被触发。

### 试验 5

~~~ruby
# lib/tasks/debug_transaction.rake

namespace :debug_transaction do
+  task for_nested_5: ['environment'] do
+    User.transaction do
+      User.create(name: 'Kotori5')
+      begin
+        User.transaction(requires_new: true) do
+          User.create(name: 'Nemu5')
+          raise 'boom!'
+        end
+      rescue Exception => e
+        puts "!!!!!!!!!!!!!"
+        puts e.message
+      end
+    end
+  end
end

~~~

~~~bash
$ RAILS_ENV=development_mysql bundle exe rake debug_transaction:for_nested_5

!!!!!!!!!!!!!
boom!
~~~

查询 users,

~~~bash
mysql> select * from users;
+----+---------+---------------------+---------------------+
| id | name    | created_at          | updated_at          |
+----+---------+---------------------+---------------------+
|  1 | Kotori  | 2015-07-19 12:55:45 | 2015-07-19 12:55:45 |
|  2 | Nemu    | 2015-07-19 12:55:45 | 2015-07-19 12:55:45 |
|  5 | Kotori3 | 2015-07-19 13:07:12 | 2015-07-19 13:07:12 |
|  7 | Kotori4 | 2015-07-19 13:31:58 | 2015-07-19 13:31:58 |
|  8 | Nemu4   | 2015-07-19 13:31:58 | 2015-07-19 13:31:58 |
|  9 | Kotori5 | 2015-07-19 13:36:29 | 2015-07-19 13:36:29 |
+----+---------+---------------------+---------------------+
6 rows in set (0.00 sec)
~~~

Kotori5 创建成功， Nemu5 没有被创建说明即使异常没有传播到父事务，由于子事务有 `requires_new: true` 参数, 子事务也被触发了。

## 小结

1. ActiveRecord 的事务不是分布式的，必须一个连接对应一个事务, 简单的来说就是一个连接使用一个 transaction 方法;

2. 必须有异常才能触发事务，如果纪录创建失败，但是没有异常，那就不会触发事务;

   ~~~ruby
     User.transaction do
	   # 假设名字有唯一性验证，虽然第二条纪录创建失败，但是因为没有异常抛出，所以不会触发事务，第一条纪录可以创建成功
	   User.create(name: 'test01')
	   User.create(name: 'test01')
	 end
   ~~~

3. 如果使用 PostgresSQL 数据库，在事务中捕捉 ActiveRecrod::InvalidStatement 异常会导致错误;

4. ActiveRecord::Rollback 可以触发自己所在事务块中的事务，但是事务块外面的代码接收不到 ActiveRecord::Rollback;

5. 嵌套的事务(nested transactions) 之间形成父子关系, 如果子事务没有携带 `requires_new: true` 参数，那么子事务的异常必须传播到父事务中，才能触发父事务和子事务, 否则父事务和子事务都不会被触发，如果子事务携带了 `requires_new: true` 参数, 那么子事务中的异常不必传到父事务中, 也能触发子事务;

代码在 [https://github.com/baya/ar-transaction-demo](https://github.com/baya/ar-transaction-demo)

