---
layout: post
title: Ruby元编程阅读笔记
date: 2014-02-12
tags: ruby元编程
category: Programming
notebook: 工作日志
---

一直觉得代码本身才是有有效的文档，本文旨在做一些Ruby特性的展示。Ruby是懒人的Ruby，各种奇技淫巧跟黑科技。

##参数数组 Argument Array
也即不定参数，本质上行是将一组参数压入一个数组中。

```ruby
def my_method (*args)
   args.map {|arg| arg.reverse}
end
my_method ('abc', 'xyz', '123')  # => ['cba', 'zyx', '321']
```

其实一直不喜欢在语言里面用到下划线，一般来讲区分的作用可以由大小写不同来代替，而且可以缩短名字长度。可见的好处是对于反射来讲用`_`会更好的`grep`，还有类似`object#to_s`这种懒到极致的缩写。

##环绕别名 Around Alias
语言层面上支持的一种方法重构技术，其可允许在旧方法前后添加新功能，同时可以使用旧方法的名字来完成新功能的调用。

```ruby
class String
   alias :real_length :length
   def length
      real_length > 18 ? 'long' : 'short'
   end
end
"abc".length # =>  'short'
"abc".real_length # => 3
```

翻译这种事情大家都懂的。

重构就干重构的活，直接动手改改名字也不拉下多少时间。见长的补丁无论对阅读，还是IDE的代码分析都增加了难度。引入一个关键字就为了弥补取名字的过错，这是为哪般。

##白板 Blank Slate
移除一个对象中的所有方法，以便把它们转换成[幽灵方法](#)

```ruby
# 从Ruby 1.9开始，增加了一个新的白板类。BasicObject
class BlankSlate < BasicObject
end
BlankSlate.instance_methods 
# => [:==, :equal?, :!, :!=, :instance_eval, :instance_exec, :__send__, :__id__]

# 旧方式
class C
  def method_missing (name, *args)
      "a Ghost Method"
   end
end
obj = C.new
obj.to_s         # => "#<C:0x3452356>"

class C
   instance_methods.each do |m|
      undef_method m unless m.to_s =~ /method_missing|respond_to?|^__/
   end
end
obj.to_s       # => "a Ghost Method"
```

##类扩展 Class Extension
通过向类的eigenclass中引入模块来定义类方法，是[对象扩展]()的一个特例。

```ruby
class C; end

module M
   def my_method
      'a class method'
   end 
end

class << C
  include M
end
C.my_method # => "a class method"
```

##类扩展混入 Class Extension Mixin
通过实现[钩子]()使一个模块扩展包含者。

```ruby
module M
    # The following method will be called when module m is included into anther module/class
    def self.included (base)
        base.extend (ClassMethods)
    end
    module ClassMethods
        def my_method
            'a class method'
        end
    end
end

puts C.my_method # =>"a class method"
```

##类实例变量 Class Instance Variable
在一个`Class`的实例变量中存储类级别的状态。

```ruby
class C
	@my_class_instance_variable = "some value"
	@@my_class_variable = "any value"

	def self.class_instance_attribute
		@my_class_instance_variable
	end

	def instance_attribute
		@my_class_instance_variable
	end

	def class_attribute
		@@my_class_variable
	end
end

puts C.class_instance_attribute             # => "some value"
obj = C.new
puts "instance: #{obj.instance_attribute}"  # => "instance:"
puts "class: #{obj.class_attribute}"        # => "class: any value"
```

由上代码可以看出此处类实例变量跟类变量还是有区别的。另外类变量在ruby中海油些奇怪的是地方

```ruby
@@v = 1
class C
    @@v = 2
end
@@v     # => 2    
```

在这里面@@v其实是定义于main的上下文，属于main的类object，属于所有Object的后代。这种共享经常带来意外的情况。

##类宏 Class Macro
在类的定义中调用当前Class instance的类方法

```ruby
class C
	def self.first_macro (arg)
		puts "first macro #{arg} called"
	end
end

class << C
	def second_macro (arg)
		puts "second macro #{arg} called"
	end
end

class C
	first_macro :test	# => first macro test called
	second_macro :test 	# => second macro called
end
```

此处展示了两种定义类方法的方式，另外常用的还有使用模块扩展。

##洁净室 Clean Room
使用对象作为执行块的上下文

```ruby
class CleanRoom
    def useful_method (x); x * 2; end
end.
CleanRoom.new.instance_eval { useful_method(3) } # => 6
```
此种方法常用语多个方法之间需要共享某些临时变量时使用。这列临时变量将只能在CleanRoom的范围内被共享。

##代码处理器 Code Processor
处理从外部获得的字符串代码

```ruby
File.readlines ("a_file_has_ruby_code.txt").each do |line|
    # 展示 eval 的使用
    puts "#{line.chomp} => #{eval (line) }" 
end
```

##上下文探针 Context Probe
执行块来获得对象上下文的信息。

```ruby
class D
	def initialize
		@x = 'inner class envrionment'
	end
end
D.new.instance_eval{@x} # => 'inner class environment'

def fred(param)
  proc {}
end

b = fred(99)
eval("param", b.binding)   #=> 99
```

此处instance_eval 中就可以访问到类D内部的变量。

##延迟执行 Deferred Evaluation
在Proc或者lambda中存储一段代码及其上下文，用以以后执行。

```ruby
class E
    def store(&block)
        @stored = block
    end
    def exec(*arg)
        @stored.call(*arg)
    end
end
obj = E.new
obj.store {|x,y| puts x * y}
obj.exec(4, 2)  # => 8
```
疑点：多个参数的时候就不知道怎么保存再调用了。...[FIX]
解决：不要再方法名称之后留多余空格，会误认为空参数的方法调用。

##动态派发 Dynamic Dispatch
在运行时决定调用方法

```ruby
method_to_call = :reverse
"123".send (method_to_call) # => "321"
```

##动态方法 Dynamic method
在运行定义方法

```ruby
class F
	class << self
		# Define method in class level
		define_method (:bar) { puts 'in F.class.bar'}
	end

	# Define instance method
	define_method (:foo) {puts 'in F.foo'}

	def create_method (name, &block)
		# CAUTION!!!
		# BY ADDING A SPACE AFTER SEND, THE METHOD WILL BE ILLEGAL
		# self.class.send (:define_method, name, &block)
		self.class.send(:define_method, name, &block)
	end
end

class << F
   define_method (:hello) { |guy| puts "say hello to #{guy}"}
end

F.class_eval ( "define_method (:apple) {puts 'like apple'}" )

F.new.foo   # => "in F.foo"
F.new.apple # => "like apple"

F.hello 'alice' # => "say hello to alice"
F.bar           # => "in F.class.bar"

obj = F.new
obj.create_method (:better) {puts self}
obj.better  # => #<F:0x23435352>
```

##动态代理 Dynamic Proxy
将不能处理的方法动态的转发给其他对象

```ruby
class Proxy
	def initialize(target)
		@target = target
	end

	def method_missing(name, *arg, &block)
		"result:#{@target.send(name, *arg, &block)}"
	end
end

obj = Proxy.new("line")
p obj.reverse   # => "result: enil"
```

注意在作为参数调用中不定参数以及block的写法。

##扁平作用域 Flat Scope
使用闭包在两个作用域之间共享变量

```ruby
class G
	def set_att(att)
		@att = att
	end
	def att
		@att
	end
end
obj = G.new
obj.set_att 10
p obj.att   # => 10
out_attr = 100

# Flat scope
obj.instance_eval {
	@att = out_attr
}
p obj.att   # => 100

```

##幽灵方法 Ghost Method
响应一个没有关联方法的消息。参见动态代理的示例。

##钩子方法 Hook Method
通过覆写特定方法来截获对象模型的事件。

```ruby
$INHERITORS=[]
class H
	def self.inherited(subclass)
		$INHERITORS << subclass
	end
end

class I < H; end
class J < H; end
class K < J; end

p $INHERITORS   # => [I,J,K]
```

##内核方法 Kernel Method
在Module Kernel中定义方法，使得方法能够在所有对象中使用。

```ruby
class Resource
    def dispose
		puts 'disposed!'
   	end
    def do_something; end
end
module Kernel
    def using(resource)
        begin
            yield
		ensure
            resource.dispose
        end
    end
end
obj = Resource.new
using(obj){
    obj.do_something
	raise "Exception!"
}
# disposed!
# test_kernel.rb:19:in `block in <main>': Exception! (RuntimeError)
#         from test_kernel.rb:10:in `using'
#         from test_kernel.rb:17:in `<main>'

```

由于Object类的定义中引入了module Kernel，所以在Obejct以及其子类均可以以调用module Kernel的方法，长用此方法来模拟关键字

##惰性实例变量 Lazy Instance Variable
这是ruby中惯用法，只有当第一次方位实例变量的时候才对其进行初始化

```ruby
class C
	def foo
		@thing ||= "some value"
	end
end
```

之所以会有这种方法的原因在于ruby的对象模型，参见类实例变量一节我们可以看到ruby其实并没有定义实例变量的地方。因此如果需要初始化一次变量就只能在类的构造函数中赋值，或者使用这种技巧。


##拟态方法 Mimic Method
将一个方法伪装成另一种语言构件。

```ruby
def BaseClass(name)
	name == "string" ? String : Object
end

class_name = "string"
class D < BaseClass class_name
end

```

Ruby中方法调用可以省略小括号，使得方法有了类似关键字的行为。需要注意的是，多个参数的时候，关键字通常只是并列参数，比如 `alias :new_method :old_method`，而方法则需要使用`,`分开。

##猴子补丁 Monkey Patch
修改已有类的特性

```ruby
class String
    def reverse
        "override"
    end
end
"abc".reverse   # => "override"
```

Ruby打开类的特性，赋予了开发人员莫大的权利。再带来便利的同时也增加了出现问题的可能。慎用。

##有名参数 Named Arguments
把方法参数收集到哈希列表中，以通过名字访问

```ruby
def foo(arg)
	arg[:test]
end

puts foo(:test => 'do it', :make => 2)  # => 'do it'
puts foo(test: 'also do it')            # => 'also do it'
```

在动态语言中，参数的类型得到了隐藏，因此在方法调用的过程中，容易造成参数混淆。有名参数正是解决这个问题而产生。

##命名空间 Namespace
在一个模块中定义常量，以防止命名冲突。

```ruby
module MyModule
	class Array
		def to_s
			"my array"
		end
	end
end
p Array.new             # => []
p MyModule::Array.new   # => 'my array'
```

##空指针保护 Nil Guard
使用‘或’操作符覆写变量

```ruby
x = nil
y = x || "value"
```

##对象扩展 Object Extension
通过给一个对象的eigenclass混入模块的方法来定义单件方法。

```ruby
module Mod
  def hello
    "Hello from Mod.\n"
  end
end

class Klass
  def hello
    "Hello from Klass.\n"
  end
end

# 方法一
k = Klass.new
k.hello         #=> "Hello from Klass.\n"
k.extend(Mod)   #=> #<Klass:0x401b3bc8>
k.hello         #=> "Hello from Mod.\n"

m = Klass.new
m.hello         #=> "Hello from Klass.\n"

# 方法二
class << m
	include Mod
end
m.hello         #=> "Hello from Mod.\n"

# 方法三
o = Klass.new
o.define_singleton_method (:hello) { "#{self}: Hello" }
o.hello         #=> "#<Klass:0x269d588>: Hello"

```

##打开类 Open Class
修改已有类

```ruby
class String
    def good_length
        'tall'
    end
end.
"abc".good_length   # => 'tall'
```

##沙盒 Sandbox
在一个安全的环境中执行未授信的代码。

```
def sandbox(&code)
	proc {
		$SAFE = 2
		yield
	}.call
end

begin
	p $SAFE         # => 0
	sandbox{File.delete "a_file"}
rescue Exception => ex
	p ex            # => #<SecurityError: Insecure operation `delete' at level 2>
	p $SAFE         # => 0
end

```

也还没搞清楚为什么`$SAFE`的值会回滚。跟`proc`块有关。

##作用域门 Scope Gate
用class, module, def关键字可以隔离作用域

```ruby
a = 1
defined? a  #=>"local-variabl"
module M
    b = 1
    defined? a #=> nil
    defined? b #=>"local-variabl"
end
defined? a #=>"local-variabl"
defined? b #=> nil
```
##Self Yield
把self块传给自身

```ruby
class Person
    attr_accessor :name, :surename
    def initialize
        yield self
    end
end
joe = Person.new do |p|
    p.name = 'joe'
    p.surename = 'smith'
end
```

##共享作用域 Shared Scope
在一个扁平作用域的多个上下文中共享变量

```ruby
proc {
	shared = 10
	self.class.class_eval {
		define_method(:counter) { shared}
		define_method(:down) { shared -=1}
	}
}.call

p counter       # => 10
3.times{down}   
p counter       # => 7

```

##单件方法 Singleton Method
在对象上定义方法，参见扩展对象

##符号对Proc Symbol To Proc
把一个符号转换为调用单个方法的代码块

```ruby
["abc", "def"].map(&:length)    # => [3,3]
```
