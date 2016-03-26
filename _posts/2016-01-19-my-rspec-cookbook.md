---
layout: post
title: 我的 Rspec Cookbook
---

## 与 Rails 的集成

参考: http://www.webascender.com/Blog/ID/629/Testing-Rails-4-Apps-With-RSpec-3-Part-II

1. Gemfile

~~~ruby
group :development, :test do
  gem 'rspec-rails', '~> 3.0.0'
  gem 'database_cleaner'
end
~~~

2. Run generators

~~~bash
bin/rails generate rspec:install
~~~

3. Configure database_cleaner

~~~ruby
# spec/rails_helper.rb

RSpec.configure do |config|

  config.before(:suite) do
    DatabaseCleaner.clean_with(:truncation)
  end

  config.before(:each) do
    DatabaseCleaner.strategy = :transaction
  end

  config.before(:each, :js => true) do
    DatabaseCleaner.strategy = :truncation
  end

  config.before(:each) do
    DatabaseCleaner.start
  end

  config.after(:each) do
    DatabaseCleaner.clean
  end

end
~~~

4. 配置 Rails 生成器使用 rspec

~~~ruby
# config/application.rb
config.generators do |g|
  g.test_framework :rspec
end
~~~

## Rspec 相关的生成器

- 参考: https://www.relishapp.com/rspec/rspec-rails/docs/generators

~~~bash
$ bin/rails generate rspec:model widget
~~~

* scaffold
* model
* controller
* helper
* view
* mailer
* observer
* integration
* feature
* job

## 单独测试一个文件

~~~bash
$ rspec spec/models/pretty_day_spec.rb 
~~~
