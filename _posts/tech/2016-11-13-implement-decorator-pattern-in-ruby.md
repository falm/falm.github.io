---
layout: post
title: Ruby 实现装饰器模式
category: 技术
tags: [ruby]
keywords: ruby, design pattern 
---


了解过Python的同学都知道，Python从语言层面支持装饰器模式，并配以语法糖 『@xx』的形式来提供更好的开发体验。本文的目标就是在Ruby中的实现对标Python的装饰器。

在开始之前我们还是来重温一下，Python中装饰器这个概念：

装饰器本质上是一个函数，它可以让其他函数在不需要做任何代码变动的前提下增加额外功能，装饰器的返回值也是一个函数对象。它经常用于有切面需求的场景，比如：插入日志、性能测试、事务处理、缓存、权限校验等场景。装饰器是解决这类问题的绝佳设计，有了装饰器，我们就可以抽离出大量与函数功能本身无关的雷同代码并继续重用。

总的来说，就是实现面向切面的AOP编程。

### Python装饰器

既然是要对标Python，那我们来看看Pyhton的装饰器，装饰器在Python中分类两大类，一类是较为常见的函数装饰器，另一类是类装饰器，文本只针对函数装饰器。

#### 函数装饰器

```python
def logger(func):
    def wrapper(*args):
        print('execute %s function' % func.__name__)
        return func(*args)
    return wrapper

@logger
def talkto(someone):
    print('talk to %s' % someone)

talkto('jam')

''' 输出
execute talkto function
talk to jam
'''
```

上面的Python代码就是装饰器的例子，可以看出来，其实利用了闭包去是先携带上下文中的func函数对象，再返回包装函数，然后让包装函数接受原来的参数，并在包装函数最后返回原函数的结果。

### Ruby装饰器

我们相应在Ruby中实现完全相同的装饰器模式，恐怕是不行了，Python使用的语法糖『@』在Ruby中是实例变量的关键字，无法重载，所以要使用其他的方法。

要达到的效果：

```ruby
class Demo

  extend MyDecorator

  def logger(method, *args)
    puts "before execute #{method.original_name}"
    yield
    puts 'after execute'
  end

  wrap :logger
  def talk_to(someone)
    puts 'talk to ' + someone
  end
end

Demo.new.talk_to '特朗普'

# 输出：
# before execute talk_to
# talk to 特朗普
# after execute
```

上面的代码，在**Demo**类的**talk_to**方法外面包裹了装饰器，装饰方法是 **logger**。 包装是通过类宏**wrap**方法提供的，它会将传入的symbol方法名，作为装饰方法。

在装饰方法**logger** 使用统一的接口：

- 至少接受两个参数，一个是被包装的方法的Method对象，另一个是该方法的参数。
- 通过**yield**关键字执行被包装方法，（这样在yield前后的代码就是前置后置执行的了）。

在上面的例子中虽然没有写出来的，其实可以通过想yield中传递参数来覆盖，原有方法的参数。

因为装饰方法的第一个参数是Method对象，所以如果你愿意的话，可以不通过yield而使用method.call 去执行被包装的**talk_to**方法。

### 实现

上面的装饰器例子的实现，全都是基于扩展了**MyDecorator**这个模块，那么**MyDecorator**中究竟做了什么呢，我们来看看：

```ruby
module MyDecorator

  def method_added(method_name)

    unless decorator_methods.empty?

      decorator_method = decorator_methods.pop

      new_name = "#{method_name}_without_decorator"

      alias_method new_name, method_name

      define_method method_name do |*args|
        method(decorator_method).call method(new_name), *args do |p = args|
          method(new_name).call(*p)
        end
      end

    end

  end

  def wrap(decorator)
    decorator_methods << decorator
  end

  def decorator_methods
    @decorator_methods ||= []
  end

end
```

**MyDecorator** 模块目前非常的简单之定义了三个方法。

- **method_added** 挂在类中方法定义是的钩子方法
- **wrap** 定义装饰的类宏，将装饰方法压入 装饰器队列中。
- **decorator_methods** 存储装饰器方法

**method_added** 中通过每次弹出一个队里中的装饰方法，再将原方法重命名，然后通过动态定义方法 **define_mehtod** 在原方法外包裹一层，这就实现了装饰器。

### 总结

装饰器是在Python原生支持，并且可以通过语法糖提供便捷的实用方法。Ruby中虽然原生不支持，但是因为语言本身的DSL能力非常强所以我们可以做出一个类Python的装饰器。因为Ruby是通过block来模拟闭包的，所以在Ruby实现中我们也使用了block，并且因为Ruby是纯OOP的，**函数**非一等公民，只能通过**函数对象**去模拟函数的传递和调用。