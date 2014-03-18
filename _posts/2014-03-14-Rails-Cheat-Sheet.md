---
layout: post
title: Rails Cheet Sheet
date: 2014-03-14
tags: rails
category: Programming
notebook: 工作日志
---

本文旨在记录常用的rails相关的命令

## Concepts
Model Layer: 这一层由业务模型类组成。在Rails中有两种类型。1. 是继承自`ActiveRecord::Base`。2. 实现了`ActiveModel`中相应接口的普通ruby类。

Controller Layer: 这一层主要用来处理http请求。在Rails中由`ActionDispatch`和`ActionController`组成。

View Layer: `ActionView`通过模板生成各种资源。

## Basic Startup
初始化项目以及基本使用。

```bash
# Initialize a proejct
rails new demo
cd demo
rake about          # try to see if everything is OK
rails server        # start web server
```

## Rails Generate  
代码生成

```bash
rails generate controller Say hello goodbye   
rails g scaffold Product title:string  # generate module/action/view for Product, upper case seems not of usage
rails g migration add_quantity_to_line_items quantity:integer  # name of the migration will help rails to guess what's your meanning
rails g model payment_type name:string # generate model
rails destroy  model Oops     # opposite of generate
```

## Debug

```bash
rails console
rails dbconsole
rails runner script/load_orders.rb
```

``` ruby
prd = Product.new
prd.save # => false
prd.errors.full_messages # => view error message
```

## Database Console

```bash
mysqld --verbose --help # lookup mysql configuration
sqlite3 -line db/development.md
```

## Rake - Ruby Make

```bash
rake -T doc # view document related task
rake --describe task # get detail description of a task
rake --tasks         # list available task for this project
rake db:create       # create the database by database.yml
rake db:migrate      # make schema change
rake db:rollback     # revert schema change
rake db:seed         # load data defined in db/seeds.rb
rake db:migrate:status # track status of migrate
rake assets:clean      # remove compiled assets
rake assets:precompile # compile all assets
rake doc:app           # generates documentation
```

## Action Model

### Attribute Magic

```ruby
class Person
  include ActiveModel::AttributeMethods

  attribute_method_prefix 'clear_'
  define_attribute_methods :name, :age

  attr_accessor :name, :age

  def clear_attribute(attr)
    send("#{attr}=", nil)
  end
end

person = Person.new
person.clear_name
person.clear_age
```

### Callbacks

```ruby
class Person
  extend ActiveModel::Callbacks
  define_model_callbacks :create

  def create
    run_callbacks :create do
      # Your create action methods here
    end
  end
end
```

### Tracking value changes

```ruby
class Person
  include ActiveModel::Dirty

  define_attribute_methods :name

  def name
    @name
  end

  def name=(val)
    name_will_change! unless val == @name
    @name = val
  end

  def save
    # do persistence work
    changes_applied
  end
end

person = Person.new
person.name             # => nil
person.changed?         # => false
person.name = 'bob'
person.changed?         # => true
person.changed          # => ['name']
person.changes          # => { 'name' => [nil, 'bob'] }
person.save
person.name = 'robert'
person.save
person.previous_changes # => {'name' => ['bob, 'robert']}
```

### Adding `errors` inteface to objects

```ruby
class Person

  def initialize
    @errors = ActiveModel::Errors.new(self)
  end

  attr_accessor :name
  attr_reader   :errors

  def validate!
    errors.add(:name, "cannot be nil") if name.nil?
  end

  def self.human_attribute_name(attr, options = {})
    "Name"
  end
end

person = Person.new
person.name = nil
person.validate!
person.errors.full_messages
# => ["Name cannot be nil"]

```

### Model name introspection

```ruby
class NamedPerson
  extend ActiveModel::Naming
end

NamedPerson.model_name        # => "NamedPerson"
NamedPerson.model_name.human  # => "Named person"
```

### making objects serializable

```ruby
class SerialPerson
  include ActiveModel::Serialization

  attr_accessor :name

  def attributes
    {'name' => name}
  end
end

s = SerialPerson.new
s.serializable_hash   # => {"name"=>nil}

class SerialPerson
  include ActiveModel::Serializers::JSON
end

s = SerialPerson.new
s.to_json             # => "{\"name\":null}"

class SerialPerson
  include ActiveModel::Serializers::Xml
end

s = SerialPerson.new
s.to_xml              # => 
```

### I18n support

```ruby
class Person
  extend ActiveModel::Translation
end

Person.human_attribute_name('my_attribute')
# => "My attribute"
```

### Validation

```ruby
class Person
  include ActiveModel::Validations

  attr_accessor :first_name, :last_name

  validates_each :first_name, :last_name do |record, attr, value|
    record.errors.add attr, 'starts with z.' if value.to_s[0] == ?z
  end
end

person = Person.new
person.first_name = 'zoolander'
person.valid?  # => false
```

### Custom Validators

```ruby
class HasNameValidator < ActiveModel::Validator
  def validate(record)
    record.errors[:name] = "must exist" if record.name.blank?
  end
end

class ValidatorPerson
  include ActiveModel::Validations
  validates_with HasNameValidator
  attr_accessor :name
end

p = ValidatorPerson.new
p.valid?                  # =>  false
p.errors.full_messages    # => ["Name must exist"]
p.name = "Bob"
p.valid?                  # =>  true
```


## ActionRecord

### Validation

```ruby
class Product < ActiveRecord::Base
  validates :title, presence: true
  validates :terms, acceptance: true
  validates :password, confirmation: true
  validates :username, exclusion: { in: %w(admin superuser) }
  validates :email, format: { with: /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\Z/i, on: :create }
  validates :age, inclusion: { in: 0..9 }
  validates :first_name, length: { maximum: 30, message: 'must be less than 30' }
  validates :age, numericality: true
  validates :username, presence: true
  validates :username, uniqueness: true
end
```

### Associations

```ruby
class Firm < ActiveRecord::Base
  has_many   :clients
  has_one    :account
  belongs_to :conglomerate
end
```

### Aggregations of value objects

```ruby
class Account < ActiveRecord::Base
  composed_of :balance, class_name: 'Money',
              mapping: %w(balance amount)
  composed_of :address,
              mapping: [%w(address_street street), %w(address_city city)]
end
```

### Transactions

```ruby
# Database transaction
Account.transaction do
  david.withdrawal(100)
  mary.deposit(100)
end
```

### Create Records
TBD:

### Find Records/Where Clauses
TBD:

### Serialized Object to Database

```ruby
class User < ActiveRecord::Base
serialize :preferences
end

user = User.create(preferences: { "background" => "black", "display" => large })
User.find(user.id).preferences # => { "background" => "black", "display" => large }
```

### Attribute Query Method

```ruby
user = User.new(name: "David")
user.name? # => true

anonymous = User.new(name: "")
anonymous.name? # => false
```

### Apply Migration
Rails defines a [TableDefinition][1] class to describe the schema. It could be init like below: (p.64)

```ruby
create_table :product do |t|
  t.text :title, limit: 50,
  t.decimal :price, precision:8, scale:2
end
```

[1]: http://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/TableDefinition.html#method-i-column "Link to Definition"

### Add Test Data
Make full use of seeds.rb to add test data. Notice `ActiveRecord::Base#Create` and `ActiveRecord::Base#Create!`  are differenct. The second one will raise an exception if the record is invalid. (p. 69)

## Action View

## Basic Helper Functions

```html
<%= link_to "Goodbye", say_goodbye_path %>!
<%= link_to "Destroy', product, method: :delete, data: { confirm: 'Are you sure?' } %>
<tr class = "<%= cycle('list_line_odd', 'list_line_even') %>" >
```

### Using link_to method in program

Need more knowledge on url routine (p7x)

### Modify Unit Test

Not known about `products(:one)` (p. 81)

### Resize image with image_tag

we could pass size as option `size: '30x30'`

### Cache in ruby
Rails 4 supports key based cache which could be referenced [here](http://guides.rubyonrails.org/caching_with_rails.html)   (p.104)
We have following steps to enable cache in rails
1. set enable in config
2. change view (*.erb) for fragments that needs to cache
3. set storage method for cache

### Dependency for controller code in rails
The following code could run in rails, so what is in the running environment. How could the ActiveSupport, ActiveRecord and Cart be found? We could also pay attention to the private mark. This would prevent rails to treat the `set_cart` method as controller method.

Answers: 
1. modules in `app/controllers/concern/` could be accessed in all controllers

```ruby
module CurrentCart
extend ActiveSupport::Concern
private
  def set_cart
    @cart = Cart.find(session[:cart_id])
    rescue ActiveRecord::RecordNotFound
    @cart = Cart.create
    session[:cart_id] = @cart
  end
end
```

### How to add connection table in rails
Table LineItem is connection table for product and cart.

```bash
rails generate scaffold LineItem product:references cart:belongs_to
```

```ruby
class LineItem < ActiveRecord::Base
  belongs_to :product
  belongs_to :cart
end
```

To add method for reverse finding from Cart we should modify Cart's model
```ruby
class Cart < ActiveRecord::Base
  has_many :line_items, dependent: :destroy
end
```

### How to choose Get/Post methods
`links_to` method will generate `Http Get` method. `button_to` will generate `POST` method.

### Parameters of method in ActionPack
There are always kind of function has the following signature `(name = nil, options = nil, html_options = nil, &block)`. I cann't see there is a need for such parameters.

### Passing objects between controllers
`params` is used to pass values between controllers.

### Error when using session
`cookieOverflow` exception throws when add item to cart. By searching the web, it is because session data is too large to store by cookie.  Change it to use database to store the value. A new gem `activerecord-session_store` is introduced.

Q: I still don't know there is the document for 'cookieOverflow'. Not in the api doc.
`++` operator is not support in Ruby
`activerecord#destroy` will trigger callback, however `activerecord#delete` won't.

### Flash in rails
Flash is hash-like object, which could pass information to following requests.

### Test in Rails
Assertions.
assert_equal, assert_not_equal
`assert_select` usage for controller auto testing

### Ruby js
ruby js may generate code with not exists node, in this time, it will be ignored.

### Run ruby scripts under rails

```ruby
Order.transaction do
  (1..100) do |i|
    Order.create (...) 
  end
end
```

```bash
rails runner script/load_orders.rb
```

### Rails send email

```bash
rails generate mailer Notifier order_received order_shipped
# create app/mailers/notifier.rb
# create ap/views/notifier/order_received.text.erb
# create ap/views/notifier/order_shipped.text.erb
# create test/functional/notifier_test.rb
```

usage
```ruby
Notifer.order_received(@order).deliver
```

### Rails integration test

```bash
rails g integration_test user_stories
# create test/integration/user_stories_test.rb
```

### Deploy ruby program on environment
apache used as front-server to process http request, and passenger will dispatch ruby reqest to rails processes.

### CRUD With ActiveRecord

```ruby
name = params[:name]
pos = Order.where(["name = ? and pay_type = 'po'", name])

name = params[:name]
pay_type = params[:pay_type]
pos = Order.where("name = :name and pay_type = :pay_type",
pay_type: pay_type, name: name)

pos = Order.where("name = :name and pay_type = :pay_type",
params[:order])

pos = Order.where(params[:order])

pos = Order.where(name: params[:name],
pay_type: params[:pay_type])

# Doesn't work
User.where("name like '?%'", params[:name])

# Works
User.where("name like ?", params[:name]+"%")

```

### Subsetting the Records found

```ruby
# The view wants to display orders grouped into pages,
# where each page shows page_size orders at a time.
# This method returns the orders on page page_num (starting
# at zero).
def Order.find_on_page(page_num, page_size)
  order(:id).limit(page_size).offset(page_num*page_size)
end

orders = Order.where(name: 'Dave').
order("pay_type, shipped_at DESC").
limit(10)
```

### Lock of Active Record

```ruby
Account.transaction do
  ac = Account.where(id: id).lock("LOCK IN SHARE MODE").first
  ac.balance -= amount if ac.balance > amount
  ac.save
end
```

### Statics of Result

```ruby
average = Order.average(:amount) # average amount of orders
max = Order.maximum(:amount)
min = Order.minimum(:amount)
total = Order.sum(:amount)
number = Order.count
```

### Scopes : to reuse simple data.

```ruby
class Order < ActiveRecord::Base
  scope :last_n_days, lambda { |days| where('updated < ?' , days) }
  scope :checks, -> { where(pay_type: :check) }
end

orders = Order.last_n_days(7)
orders = Order.checks.last_n_days(7)
```

### Update on ActiveRecord

```ruby
# fetch and update record, id is used as first parameter
order = Order.update(12, name: "Barney", email: "barney@bedrock.com")
# where clause is the second parameter
# return result depends on database adapter
result = Product.update_all("price = 1.1*price", "title like '%Java%'")
```

### Delete Rows
```ruby
# delete method bypass the validation and ActiveRecord callback
Order.delete(123)
User.delete([2,3,4,5])

# when no condition is given, all rows are deleted.
Product.delete_all(["price > ?", @expensive_price])
```

### Active Record Callback

```ruby
class Order < ActiveRecord::Base
  before_validation :normalize_credit_card_number
  after_create do |order|
     logger.info "Order #{order.id} created"
  end
  protected
  def normalize_credit_card_number

   # Notice we have to use self. since this is class method???/
    self.cc_number.gsub!(/[-\s]/, '')
  end
end
```

### Action Pack

Action Dispatch helps to find controller and method for processing request. There are two ways to defined the routes, pre-defined and self-defined.

```ruby
Depot::Application.routes.draw do
  # pre-defined, 
  # This assumes there is a controller named 'ProductController' and several methods with given names
  resources :products
  resources :comments, except: [:update, :destroy]  # update, destroy is not in need.
  # sef-defined
  resources :products do
    get :who_bought, on: :member  # :collection ~
  end
end
```

We could use `rake routes` to review the generated routes.
Notes: 'GET    /products(.:format) products#index'  the `/products(.:format)` stands for '/product/1.html` or '/product/1.xml'

There are 7 methods that rails generate for us. `index`,`create`,`new`,`show`,`destroy`,`edit`,`update`

Diff: `create` and `new`, `create` receives a bunch of attributes from POST request and create a new source and save it. `new` method is only create a empty resource used to gather inputs from client, resource is not saved. It is like `initialized` method in Base Module

`edit` and `update`, `edit` returns the content of a resource identified by `params[:id]`, `update` will update the resource by `params[:id]` with data associated with request.

We could fire a request on client by ruby methods

```html
<% link_to 'Show', product %>
# link_to generate a HTTP GET by default, so need to specify using DELETE instead.
<% link_to 'Destroy', product, method: :delete %>
```

Rails uses one controller method to handle all types of output(xml/json/html). This may increase difficulty for error handling.

### Process request in controller
TBD

### Q&A

* Where to find the document for phantom method, since they are created dynamically? e.g. _find*_
* Rails method has return type, however it is hard to find them.
* Help function `content_tag` is no use in my sample
* Duplicate item is added when click image.
* How to create class method like before_validate, it could use a class instance as parameter, which method it knows to call.
* It is not safe to store primary key in the url. The default route of rails needs to changed? p309
* In RESTful design, the server only works on CRUD of resources. Is there any concern about the validation?? p310
* Nested routes will provide all resource information on the full path? p318
* Error handling when user specify illegal URI. `http://192.168.33.15:3000/products/1/who_bought.sfas` 
* What is a permanent/temporary redirect? what's the differences?
* How we could build a custom program avoid using ActionPack
* How to deploy with passenger? There is no content under public directory. p237.