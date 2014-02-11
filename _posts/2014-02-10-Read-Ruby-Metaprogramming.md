---
layout: post
title: Ruby元编程读书笔记
date: 2014-02-12 00:35:14 +08:00
category: Ruby
tags: "Ruby 元编程"
---

#Ruby元编程读书笔记

一直觉得代码本身才是有有效的文档，本文旨在做一些Ruby特性的展示。Ruby是懒人的Ruby，各种奇技淫巧跟黑科技。

##参数数组 Argument Array
也即不定参数，本质上行是将一组参数压入一个数组中。

	```Ruby
	def my_method (*args)
	   args.map {|arg| arg.reverse}
	end
	my_method ('abc', 'xyz', '123')  # => ['cba', 'zyx', '321']
	```

*槽点* 其实一直不喜欢在语言里面用到下划线，一般来讲区分的作用可以由大小写不同来代替，而且可以缩短名字长度。可见的好处是对于反射来讲用`_`会更好的`grep`，还有类似`object#to_s`这种懒到极致的缩写。

##环绕别名 Around Alias
语言层面上支持的一种方法重构技术，其可允许在旧方法前后添加新功能，同时可以使用旧方法的名字来完成新功能的调用。

	```Ruby
	class String
	   alias :real_length :length
	   def length
	      real_length > 18 ? 'long' : 'short'
	   end
	end
	"abc".length # =>  'short'
	"abc".real_length # => 3
	```

*槽点* 翻译这种事情大家都懂的。

*槽点* 重构就干重构的活，直接动手改改名字也不拉下多少时间。见长的补丁无论对阅读，还是IDE的代码分析都增加了难度。引入一个关键字就为了弥补取名字的过错，这是为哪般。

##白板 Blank Slate
移除一个对象中的所有方法，以便把它们转换成[幽灵方法](#)

	```Ruby
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

	```Ruby
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

	```Ruby
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

*槽点* 有`Mixin`的地方翻译就是个旦。

##类实例变量 Class Instance Variable
在一个`Class`的实例变量中存储类级别的状态。

