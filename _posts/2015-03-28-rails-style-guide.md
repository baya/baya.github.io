---
layout: post
title: Rails Style Guide 注解
---

原贴在此: https://github.com/bbatsov/rails-style-guide

## Configuration

* <a name="config-initializers"></a>
  Put custom initialization code in `config/initializers`. The code in
  initializers executes on application startup.
<sup>[同意]</sup>

* <a name="gem-initializers"></a>
  Keep initialization code for each gem in a separate file with the same name
  as the gem, for example `carrierwave.rb`, `active_admin.rb`, etc.
<sup>[同意]</sup>

* <a name="dev-test-prod-configs"></a>
  Adjust accordingly the settings for development, test and production
  environment (in the corresponding files under `config/environments/`)
<sup>[同意]</sup>

  * Mark additional assets for precompilation (if any):

    ~~~ruby
    # config/environments/production.rb
    # Precompile additional assets (application.js, application.css,
    #and all non-JS/CSS are already added)
    config.assets.precompile += %w( rails_admin/rails_admin.css rails_admin/rails_admin.js )
    ~~~

* <a name="app-config"></a>
  Keep configuration that's applicable to all environments in the
  `config/application.rb` file.
<sup>[同意]</sup>

* <a name="staging-like-prod"></a>
  Create an additional `staging` environment that closely resembles the
  `production` one.
<sup>[同意]</sup>

* <a name="yaml-config"></a>
  Keep any additional configuration in YAML files under the `config/` directory.
<sup>[同意]</sup>

  Since Rails 4.2 YAML configuration files can be easily loaded with the new `config_for` method:

  ~~~ruby
  Rails::Application.config_for(:yaml_file)
  ~~~

## Routing

* <a name="member-collection-routes"></a>
  When you need to add more actions to a RESTful resource (do you really need
  them at all?) use `member` and `collection` routes.
<sup>[同意]</sup>

  ~~~ruby
  # bad
  get 'subscriptions/:id/unsubscribe'
  resources :subscriptions

  # good
  resources :subscriptions do
    get 'unsubscribe', on: :member
  end

  # bad
  get 'photos/search'
  resources :photos

  # good
  resources :photos do
    get 'search', on: :collection
  end
  ~~~

* <a name="many-member-collection-routes"></a>
  If you need to define multiple `member/collection` routes use the
  alternative block syntax.
<sup>[同意]</sup>

  ~~~ruby
  resources :subscriptions do
    member do
      get 'unsubscribe'
      # more routes
    end
  end

  resources :photos do
    collection do
      get 'search'
      # more routes
    end
  end
  ~~~

* <a name="nested-routes"></a>
  Use nested routes to express better the relationship between ActiveRecord
  models.
<sup>[同意]</sup>

  ~~~ruby
  class Post < ActiveRecord::Base
    has_many :comments
  end

  class Comments < ActiveRecord::Base
    belongs_to :post
  end

  # routes.rb
  resources :posts do
    resources :comments
  end
  
  ~~~

* <a name="namespaced-routes"></a>
  Use namespaced routes to group related actions.
<sup>[同意]</sup>

  ~~~ruby
  namespace :admin do
    # Directs /admin/products/* to Admin::ProductsController
    # (app/controllers/admin/products_controller.rb)
    resources :products
  end
  ~~~

* <a name="no-wild-routes"></a>
  Never use the legacy wild controller route. This route will make all actions
  in every controller accessible via GET requests.
<sup>[同意]</sup>

  ~~~ruby
  # very bad
  match ':controller(/:action(/:id(.:format)))'
  ~~~

* <a name="no-match-routes"></a>
  Don't use `match` to define any routes unless there is need to map multiple request types among `[:get, :post, :patch, :put, :delete]` to a single action using `:via` option.
<sup>[同意]</sup>

## Controllers

* <a name="skinny-controllers"></a>
  Keep the controllers skinny - they should only retrieve data for the view
  layer and shouldn't contain any business logic (all the business logic
  should naturally reside in the model).
<sup>[同意]</sup>

保持 controller 苗条的手段除了将业务逻辑相关的代码转移到 model, 还有,

- 提炼出 service object
- 多建立 controller

* <a name="one-method"></a>
  Each controller action should (ideally) invoke only one method other than an
  initial find or new.
<sup>[同意]</sup>

* <a name="shared-instance-variables"></a>
  Share no more than two instance variables between a controller and a view.
<sup>[同意]</sup>

## Models

* <a name="model-classes"></a>
  Introduce non-ActiveRecord model classes freely.
<sup>[同意]</sup>

* <a name="meaningful-model-names"></a>
  Name the models with meaningful (but short) names without abbreviations.
<sup>[同意]</sup>

* <a name="activeattr-gem"></a>
  If you need model objects that support ActiveRecord behavior (like validation)
  without the ActiveRecord database functionality use the
  [ActiveAttr](https://github.com/cgriego/active_attr) gem.
<sup>[不建议使用]</sup>

  ~~~ruby
  class Message
    include ActiveAttr::Model

    attribute :name
    attribute :email
    attribute :content
    attribute :priority

    attr_accessible :name, :email, :content

    validates :name, presence: true
    validates :email, format: { with: /\A[-a-z0-9_+\.]+\@([-a-z0-9]+\.)+[a-z0-9]{2,4}\z/i }
    validates :content, length: { maximum: 500 }
  end
  ~~~

  For a more complete example refer to the
  [RailsCast on the subject](http://railscasts.com/episodes/326-activeattr).

### ActiveRecord

* <a name="keep-ar-defaults"></a>
  Avoid altering ActiveRecord defaults (table names, primary key, etc) unless
  you have a very good reason (like a database that's not under your control).
<sup>[同意]</sup>

  ~~~ruby
  # bad - don't do this if you can modify the schema
  class Transaction < ActiveRecord::Base
    self.table_name = 'order'
    ...
  end
  ~~~

* <a name="macro-style-methods"></a>
  Group macro-style methods (`has_many`, `validates`, etc) in the beginning of
  the class definition.
<sup>[同意]</sup>

  ~~~ruby
  class User < ActiveRecord::Base
    # keep the default scope first (if any)
    default_scope { where(active: true) }

    # constants come up next
    COLORS = %w(red green blue)

    # afterwards we put attr related macros
    attr_accessor :formatted_date_of_birth

    attr_accessible :login, :first_name, :last_name, :email, :password

    # followed by association macros
    belongs_to :country

    has_many :authentications, dependent: :destroy

    # and validation macros
    validates :email, presence: true
    validates :username, presence: true
    validates :username, uniqueness: { case_sensitive: false }
    validates :username, format: { with: /\A[A-Za-z][A-Za-z0-9._-]{2,19}\z/ }
    validates :password, format: { with: /\A\S{8,128}\z/, allow_nil: true}

    # next we have callbacks
    before_save :cook
    before_save :update_username_lower

    # other macros (like devise's) should be placed after the callbacks

    ...
  end
  ~~~

* <a name="has-many-through"></a>
  Prefer `has_many :through` to `has_and_belongs_to_many`. Using `has_many
  :through` allows additional attributes and validations on the join model.
<sup>[同意]</sup>

  ~~~ruby
  # not so good - using has_and_belongs_to_many
  class User < ActiveRecord::Base
    has_and_belongs_to_many :groups
  end

  class Group < ActiveRecord::Base
    has_and_belongs_to_many :users
  end

  # prefered way - using has_many :through
  class User < ActiveRecord::Base
    has_many :memberships
    has_many :groups, through: :memberships
  end

  class Membership < ActiveRecord::Base
    belongs_to :user
    belongs_to :group
  end

  class Group < ActiveRecord::Base
    has_many :memberships
    has_many :users, through: :memberships
  end
  ~~~

* <a name="read-attribute"></a>
  Prefer `self[:attribute]` over `read_attribute(:attribute)`.
<sup>[同意]</sup>

  ~~~ruby
  # bad
  def amount
    read_attribute(:amount) * 100
  end

  # good
  def amount
    self[:amount] * 100
  end
  ~~~

* <a name="write-attribute"></a>
  Prefer `self[:attribute] = value` over `write_attribute(:attribute, value)`.
<sup>[同意]</sup>

  ~~~ruby
  # bad
  def amount
    write_attribute(:amount, 100)
  end

  # good
  def amount
    self[:amount] = 100
  end
  ~~~

* <a name="sexy-validations"></a>
  Always use the new ["sexy"
  validations](http://thelucid.com/2010/01/08/sexy-validation-in-edge-rails-rails-3/).
<sup>[同意]</sup>

  ~~~ruby
  # bad
  validates_presence_of :email
  validates_length_of :email, maximum: 100

  # good
  validates :email, presence: true, length: { maximum: 100 }
  ~~~

* <a name="custom-validator-file"></a>
  When a custom validation is used more than once or the validation is some
  regular expression mapping, create a custom validator file.
<sup>[同意]</sup>

  ~~~ruby
  # bad
  class Person
    validates :email, format: { with: /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i }
  end

  # good
  class EmailValidator < ActiveModel::EachValidator
    def validate_each(record, attribute, value)
      record.errors[attribute] << (options[:message] || 'is not a valid email') unless value =~ /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i
    end
  end

  class Person
    validates :email, email: true
  end
  ~~~

* <a name="app-validators"></a>
  Keep custom validators under `app/validators`.
<sup>[同意]</sup>

* <a name="custom-validators-gem"></a>
  Consider extracting custom validators to a shared gem if you're maintaining
  several related apps or the validators are generic enough.
<sup>[同意]</sup>

* <a name="named-scopes"></a>
  Use named scopes freely.
<sup>[同意]</sup>

```ruby
class User < ActiveRecord::Base
scope :active, -> { where(active: true) }
scope :inactive, -> { where(active: false) }

scope :with_orders, -> { joins(:orders).select('distinct(users.id)') }
end
```

* <a name="named-scope-class"></a>
  When a named scope defined with a lambda and parameters becomes too
  complicated, it is preferable to make a class method instead which serves the
  same purpose of the named scope and returns an `ActiveRecord::Relation`
  object. Arguably you can define even simpler scopes like this.

<sup>[同意]</sup>

~~~ruby
class User < ActiveRecord::Base
def self.with_orders
joins(:orders).select('distinct(users.id)')
end
end
~~~

  Note that this style of scoping cannot be chained in the same way as named scopes. For instance:

  这里应该不是 unchainable 而是 chainable,
  
  ```ruby
  # unchainable
 class User < ActiveRecord::Base
   def User.old
     where('age > ?', 80)
   end

   def User.heavy
     where('weight > ?', 200)
   end
 end
```
  
In this style both `old` and `heavy` work individually, but you cannot call `User.old.heavy`, to chain these scopes use:

~~~ruby
# chainable
class User < ActiveRecord::Base
scope :old, -> { where('age > 60') }
scope :heavy, -> { where('weight > 200') }
end
~~~

* <a name="beware-update-attribute"></a>
  Beware of the behavior of the
  [`update_attribute`](http://api.rubyonrails.org/classes/ActiveRecord/Persistence.html#method-i-update_attribute)
  method. It doesn't run the model validations (unlike `update_attributes`) and
  could easily corrupt the model state.
<sup>[同意]</sup>

* <a name="user-friendly-urls"></a>
  Use user-friendly URLs. Show some descriptive attribute of the model in the URL
  rather than its `id`.  There is more than one way to achieve this:
<sup>[同意]</sup>

  * Override the `to_param` method of the model. This method is used by Rails
    for constructing a URL to the object.  The default implementation returns
    the `id` of the record as a String.  It could be overridden to include another
    human-readable attribute.

~~~ruby
class Person
def to_param
"#{id} #{name}".parameterize
end
end
~~~

  In order to convert this to a URL-friendly value, `parameterize` should be
  called on the string. The `id` of the object needs to be at the beginning so
  that it can be found by the `find` method of ActiveRecord.

  * Use the `friendly_id` gem. It allows creation of human-readable URLs by
    using some descriptive attribute of the model instead of its `id`.

~~~ruby
class Person
extend FriendlyId
friendly_id :name, use: :slugged
end
~~~

  Check the [gem documentation](https://github.com/norman/friendly_id) for more
  information about its usage.

* <a name="find-each"></a>
  Use `find_each` to iterate over a collection of AR objects. Looping through a
  collection of records from the database (using the `all` method, for example)
  is very inefficient since it will try to instantiate all the objects at once.
  In that case, batch processing methods allow you to work with the records in
  batches, thereby greatly reducing memory consumption.
<sup>[同意]</sup>


~~~ruby
# bad
Person.all.each do |person|
person.do_awesome_stuff
end

Person.where('age > 21').each do |person|
person.party_all_night!
end

# good
Person.find_each do |person|
person.do_awesome_stuff
end

Person.where('age > 21').find_each do |person|
person.party_all_night!
end
~~~

* <a name="before_destroy"></a>
  Since [Rails creates callbacks for dependent
  associations](https://github.com/rails/rails/issues/3458), always call
  `before_destroy` callbacks that perform validation with `prepend: true`.
<sup>[同意]</sup>

  ~~~ruby
  # bad (roles will be deleted automatically even if super_admin? is true)
  has_many :roles, dependent: :destroy

  before_destroy :ensure_deletable

  def ensure_deletable
    fail "Cannot delete super admin." if super_admin?
  end

  # good
  has_many :roles, dependent: :destroy

  before_destroy :ensure_deletable, prepend: true

  def ensure_deletable
    fail "Cannot delete super admin." if super_admin?
  end
  ~~~

### ActiveRecord Queries

* <a name="avoid-interpolation"></a>
  Avoid string interpolation in
  queries, as it will make your code susceptible to SQL injection
  attacks.
<sup>[同意]</sup>

  ~~~ruby
  # bad - param will be interpolated unescaped
  Client.where("orders_count = #{params[:orders]}")

  # good - param will be properly escaped
  Client.where('orders_count = ?', params[:orders])
  ~~~

* <a name="named-placeholder"></a>
  Consider using named placeholders instead of positional placeholders
  when you have more than 1 placeholder in your query.
<sup>[同意]</sup>

  ~~~ruby
  # okish
  Client.where(
    'created_at >= ? AND created_at <= ?',
    params[:start_date], params[:end_date]
  )

  # good
  Client.where(
    'created_at >= :start_date AND created_at <= :end_date',
    start_date: params[:start_date], end_date: params[:end_date]
  )
  ~~~

* <a name="find"></a>
  Favor the use of `find` over `where`
when you need to retrieve a single record by id.
<sup>[同意]</sup>

  ~~~ruby
  # bad
  User.where(id: id).take

  # good
  User.find(id)
  ~~~

* <a name="find_by"></a>
  Favor the use of `find_by` over `where`
when you need to retrieve a single record by some attributes.
<sup>[同意]</sup>

  ~~~ruby
  # bad
  User.where(first_name: 'Bruce', last_name: 'Wayne').first

  # good
  User.find_by(first_name: 'Bruce', last_name: 'Wayne')
  ~~~

* <a name="find_each"></a>
  Use `find_each` when you need to process a lot of records.
<sup>[同意]</sup>

  ~~~ruby
  # bad - loads all the records at once
  # This is very inefficient when the users table has thousands of rows.
  User.all.each do |user|
    NewsMailer.weekly(user).deliver_now
  end

  # good - records are retrieved in batches
  User.find_each do |user|
    NewsMailer.weekly(user).deliver_now
  end
  ~~~

* <a name="where-not"></a>
  Favor the use of `where.not` over SQL.
<sup>[同意]</sup>

  ~~~ruby
  # bad
  User.where("id != ?", id)

  # good
  User.where.not(id: id)
  ~~~

## Migrations

* <a name="schema-version"></a>
  Keep the `schema.rb` (or `structure.sql`) under version control.
<sup>[同意]</sup>

* <a name="db-schema-load"></a>
  Use `rake db:schema:load` instead of `rake db:migrate` to initialize an empty
  database.
<sup>[同意]</sup>

* <a name="default-migration-values"></a>
  Enforce default values in the migrations themselves instead of in the
  application layer.
<sup>[同意]</sup>

  ~~~ruby
  # bad - application enforced default value
  def amount
    self[:amount] or 0
  end
  ~~~

  While enforcing table defaults only in Rails is suggested by many
  Rails developers, it's an extremely brittle approach that
  leaves your data vulnerable to many application bugs.  And you'll
  have to consider the fact that most non-trivial apps share a
  database with other applications, so imposing data integrity from
  the Rails app is impossible.

* <a name="foreign-key-constraints"></a>
  Enforce foreign-key constraints. As of Rails 4.2, ActiveRecord
  supports foreign key constraints natively.
  <sup>[同意]</sup>

* <a name="change-vs-up-down"></a>
  When writing constructive migrations (adding tables or columns),
  use the `change` method instead of `up` and `down` methods.
  <sup>[同意]</sup>

  ~~~ruby
  # the old way
  class AddNameToPeople < ActiveRecord::Migration
    def up
      add_column :people, :name, :string
    end

    def down
      remove_column :people, :name
    end
  end

  # the new prefered way
  class AddNameToPeople < ActiveRecord::Migration
    def change
      add_column :people, :name, :string
    end
  end
  ~~~

* <a name="no-model-class-migrations"></a>
  Don't use model classes in migrations. The model classes are constantly
  evolving and at some point in the future migrations that used to work might
  stop, because of changes in the models used.
<sup>[同意]</sup>

## Views

* <a name="no-direct-model-view"></a>
  Never call the model layer directly from a view.
<sup>[同意]</sup>

* <a name="no-complex-view-formatting"></a>
  Never make complex formatting in the views, export the formatting to a method
  in the view helper or the model.
<sup>[同意]</sup>

* <a name="partials"></a>
  Mitigate code duplication by using partial templates and layouts.
<sup>[同意]</sup>

## Internationalization

* <a name="locale-texts"></a>
  No strings or other locale specific settings should be used in the views,
  models and controllers. These texts should be moved to the locale files in the
  `config/locales` directory.
<sup>[同意]</sup>

* <a name="translated-labels"></a>
  When the labels of an ActiveRecord model need to be translated, use the
  `activerecord` scope:
<sup>[同意]</sup>

  ~~~
  en:
    activerecord:
      models:
        user: Member
      attributes:
        user:
          name: 'Full name'
  ~~~

  Then `User.model_name.human` will return "Member" and
  `User.human_attribute_name("name")` will return "Full name". These
  translations of the attributes will be used as labels in the views.


* <a name="organize-locale-files"></a>
  Separate the texts used in the views from translations of ActiveRecord
  attributes. Place the locale files for the models in a folder `models` and the
  texts used in the views in folder `views`.
<sup>[同意]</sup>

  * When organization of the locale files is done with additional directories,
    these directories must be described in the `application.rb` file in order
    to be loaded.

      ~~~ruby
      # config/application.rb
      config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.{rb,yml}')]
      ~~~

* <a name="shared-localization"></a>
  Place the shared localization options, such as date or currency formats, in
  files under the root of the `locales` directory.
<sup>[同意]</sup>

* <a name="short-i18n"></a>
  Use the short form of the I18n methods: `I18n.t` instead of `I18n.translate`
  and `I18n.l` instead of `I18n.localize`.
<sup>[同意]</sup>

* <a name="lazy-lookup"></a>
  Use "lazy" lookup for the texts used in views. Let's say we have the following
  structure:
<sup>[同意]</sup>

  ~~~
  en:
    users:
      show:
        title: 'User details page'
  ~~~

  The value for `users.show.title` can be looked up in the template
  `app/views/users/show.html.haml` like this:

  ~~~ruby
  = t '.title'
  ~~~

* <a name="dot-separated-keys"></a>
  Use the dot-separated keys in the controllers and models instead of specifying
  the `:scope` option. The dot-separated call is easier to read and trace the
  hierarchy.
<sup>[同意]</sup>

  ~~~ruby
  # bad
  I18n.t :record_invalid, :scope => [:activerecord, :errors, :messages]

  # good
  I18n.t 'activerecord.errors.messages.record_invalid'
  ~~~

* <a name="i18n-guides"></a>
  More detailed information about the Rails I18n can be found in the [Rails
  Guides](http://guides.rubyonrails.org/i18n.html)

## Assets

Use the [assets pipeline](http://guides.rubyonrails.org/asset_pipeline.html) to leverage organization within
your application.

* <a name="reserve-app-assets"></a>
  Reserve `app/assets` for custom stylesheets, javascripts, or images.
<sup>[同意]</sup>

* <a name="lib-assets"></a>
  Use `lib/assets` for your own libraries that don’t really fit into the
  scope of the application.
<sup>[同意]</sup>

* <a name="vendor-assets"></a>
  Third party code such as [jQuery](http://jquery.com/) or
  [bootstrap](http://twitter.github.com/bootstrap/) should be placed in
  `vendor/assets`.
<sup>[同意]</sup>

* <a name="gem-assets"></a>
  When possible, use gemified versions of assets (e.g.
  [jquery-rails](https://github.com/rails/jquery-rails),
  [jquery-ui-rails](https://github.com/joliss/jquery-ui-rails),
  [bootstrap-sass](https://github.com/thomas-mcdonald/bootstrap-sass),
  [zurb-foundation](https://github.com/zurb/foundation)).
<sup>[同意]</sup>

## Mailers

* <a name="mailer-name"></a>
  Name the mailers `SomethingMailer`. Without the Mailer suffix it isn't
  immediately apparent what's a mailer and which views are related to the
  mailer.
<sup>[同意]</sup>

* <a name="html-plain-email"></a>
  Provide both HTML and plain-text view templates.
<sup>[同意]</sup>

* <a name="enable-delivery-errors"></a>
  Enable errors raised on failed mail delivery in your development environment.
  The errors are disabled by default.
<sup>[同意)]</sup>

  ~~~ruby
  # config/environments/development.rb

  config.action_mailer.raise_delivery_errors = true
  ~~~

* <a name="local-smtp"></a>
  Use a local SMTP server like
  [Mailcatcher](https://github.com/sj26/mailcatcher) in the development
  environment.
<sup>[同意]</sup>

  ~~~ruby
  # config/environments/development.rb

  config.action_mailer.smtp_settings = {
    address: 'localhost',
    port: 1025,
    # more settings
  }
  ~~~

* <a name="default-hostname"></a>
  Provide default settings for the host name.
<sup>[同意]</sup>

  ~~~ruby
  # config/environments/development.rb
  config.action_mailer.default_url_options = { host: "#{local_ip}:3000" }

  # config/environments/production.rb
  config.action_mailer.default_url_options = { host: 'your_site.com' }

  # in your mailer class
  default_url_options[:host] = 'your_site.com'
  ~~~

* <a name="url-not-path-in-email"></a>
  If you need to use a link to your site in an email, always use the `_url`, not
  `_path` methods. The `_url` methods include the host name and the `_path`
  methods don't.
<sup>[同意]</sup>

  ~~~ruby
  # bad
  You can always find more info about this course
  <%= link_to 'here', course_path(@course) %>

  # good
  You can always find more info about this course
  <%= link_to 'here', course_url(@course) %>
  ~~~

* <a name="email-addresses"></a>
  Format the from and to addresses properly. Use the following format:
<sup>[同意]</sup>

  ~~~ruby
  # in your mailer class
  default from: 'Your Name <info@your_site.com>'
  ~~~

* <a name="delivery-method-test"></a>
  Make sure that the e-mail delivery method for your test environment is set to
  `test`:
<sup>[同意]</sup>

  ~~~ruby
  # config/environments/test.rb

  config.action_mailer.delivery_method = :test
  ~~~

* <a name="delivery-method-smtp"></a>
  The delivery method for development and production should be `smtp`:
<sup>[同意]</sup>

  ~~~ruby
  # config/environments/development.rb, config/environments/production.rb

  config.action_mailer.delivery_method = :smtp
  ~~~

* <a name="inline-email-styles"></a>
  When sending html emails all styles should be inline, as some mail clients
  have problems with external styles. This however makes them harder to maintain
  and leads to code duplication. There are two similar gems that transform the
  styles and put them in the corresponding html tags:
  [premailer-rails](https://github.com/fphilipe/premailer-rails) and
  [roadie](https://github.com/Mange/roadie).
<sup>[同意]</sup>

* <a name="background-email"></a>
  Sending emails while generating page response should be avoided. It causes
  delays in loading of the page and request can timeout if multiple email are
  sent. To overcome this emails can be sent in background process with the help
  of [sidekiq](https://github.com/mperham/sidekiq) gem.
<sup>[同意]</sup>

## Time

* <a name="tz-config"></a>
  Config your timezone accordingly in `application.rb`.
<sup>[同意]</sup>

  ~~~ruby
  config.time_zone = 'Eastern European Time'
  # optional - note it can be only :utc or :local (default is :utc)
  config.active_record.default_timezone = :local
  ~~~

* <a name="time-parse"></a>
  Don't use `Time.parse`.
<sup>[同意]</sup>

  ~~~ruby
  # bad
  Time.parse('2015-03-02 19:05:37') # => Will assume time string given is in the system's time zone.

  # good
  Time.zone.parse('2015-03-02 19:05:37') # => Mon, 02 Mar 2015 19:05:37 EET +02:00
  ~~~

* <a name="time-now"></a>
  Don't use `Time.now`.
<sup>[同意]</sup>

  ~~~ruby
  # bad
  Time.now # => Returns system time and ignores your configured time zone.

  # good
  Time.zone.now # => Fri, 12 Mar 2014 22:04:47 EET +02:00
  Time.current # Same thing but shorter.
  ~~~

## Bundler

* <a name="dev-test-gems"></a>
  Put gems used only for development or testing in the appropriate group in the
  Gemfile.
<sup>[同意]</sup>

* <a name="only-good-gems"></a>
  Use only established gems in your projects. If you're contemplating on
  including some little-known gem you should do a careful review of its source
  code first.
<sup>[同意]</sup>

* <a name="os-specific-gemfile-locks"></a>
  OS-specific gems will by default result in a constantly changing
  `Gemfile.lock` for projects with multiple developers using different operating
  systems.  Add all OS X specific gems to a `darwin` group in the Gemfile, and
  all Linux specific gems to a `linux` group:
<sup>[同意]</sup>

  ~~~ruby
  # Gemfile
  group :darwin do
    gem 'rb-fsevent'
    gem 'growl'
  end

  group :linux do
    gem 'rb-inotify'
  end
  ~~~

  To require the appropriate gems in the right environment, add the
  following to `config/application.rb`:

  ~~~ruby
  platform = RUBY_PLATFORM.match(/(linux|darwin)/)[0].to_sym
  Bundler.require(platform)
  ~~~

* <a name="gemfile-lock"></a>
  Do not remove the `Gemfile.lock` from version control. This is not some
  randomly generated file - it makes sure that all of your team members get the
  same gem versions when they do a `bundle install`.
<sup>[同意]</sup>

## Flawed Gems

This is a list of gems that are either problematic or superseded by
other gems. You should avoid using them in your projects.

* [rmagick](http://rmagick.rubyforge.org/) - this gem is notorious for its
  memory consumption. Use
  [minimagick](https://github.com/minimagick/minimagick) instead.

* [autotest](http://www.zenspider.com/ZSS/Products/ZenTest/) - old solution for
  running tests automatically. Far inferior to
  [guard](https://github.com/guard/guard) and
  [watchr](https://github.com/mynyml/watchr).

* [rcov](https://github.com/relevance/rcov) - code coverage tool, not compatible
  with Ruby 1.9. Use [SimpleCov](https://github.com/colszowka/simplecov)
  instead.

* [therubyracer](https://github.com/cowboyd/therubyracer) - the use of this gem
  in production is strongly discouraged as it uses a very large amount of
  memory. I'd suggest using `node.js` instead.

This list is also a work in progress. Please, let me know if you know other
popular, but flawed gems.

## Managing processes

* <a name="foreman"></a>
  If your projects depends on various external processes use
  [foreman](https://github.com/ddollar/foreman) to manage them.
<sup>[同意]</sup>

