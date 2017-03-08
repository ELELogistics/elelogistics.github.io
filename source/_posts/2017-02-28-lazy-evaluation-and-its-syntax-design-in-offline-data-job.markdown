---
title: 离线数据任务中的惰性求值语法设计
date: 2017-03-08 15:50:58
author : 陈大伟
tags: 离线数据
---

>  概要: 对惰性求值在离线数据任务中的好处做了定义和如何实现的分析，并从有向无环图中偷师如何解耦复杂计算中的数据逻辑依赖，然后分别从
>        Python, Java, Scala, Ruby 这四个语言如何表达惰性求值中去考察 这几个编程语言的特点，最后做了表格对比分析和总结。


在处理离线数据相关任务时，项目的主要目的是为了解决某个数据需求，然而其本质也是一个
软件工程问题。所以也和通常的软件开发一样，都需要遵循 [DRY(Don't repeat yourself)][DRY],
[SOLID][SOLID] 和 [KISS principle(Keep it simple, stupid)][KISS_principle] 等黄金原则。
而和一般软件开发不太一样的地方在于，离线数据处理的重点在于数据，一个是数据量大，另一个是数据逻辑复杂。

本文不探讨用具体某种算法或数据结构去优化这些数据任务，就单单从编程语言这一层面来看
也许哪些地方我们还可以做得更好，一个是做到 简单易懂好维护，另一个是做到 程序优雅。
考察的编程语言主要是我个人用过的 Scala, Python, Java 和 Ruby。


### 惰性求值
[Lazy Evaluation][Lazy_Evaluation] 是指 定义了一个变量和其对应的计算表达式，
等到实际需要去访问这个值时，才对该表达式进行求值。这种方式会带来以下两个好处:

1. 加快程序处理速度。在进行某个操作时，通常我们需要先预备好相关的资源，
   不管最后用不用得上，根据主流编程语言的及早求值([Eager evaluation][Eager_evaluation])
   策略，这些资源都是需要被马上载入的，那这就在一定程度上使程序响应时间变慢了。
   所以一般都是程序员得小心翼翼地用 IF-ELSE 维护好每一处逻辑。

2. 一次计算，多次使用。既然是惰性求值了，那就意味着不被求值时，变量被随意多次引用
   也不会引起耗时计算或空间浪费。而如果被求值了，那该值也会以某种形式(比如
   Memoization) 被保存下来，以避免重复计算。

而如何 **表达** 惰性计算呢？根据上面的定义，惰性计算需要满足以下三个部件:

1. 变量的声明。
2. 表达式的声明。
3. 变量和表达式如何绑定。

所以我们应该是需要:

1. 创造之。声明一个惰性的对象，并直接把表达式和它绑定。
2. 使用之。在我们需要访问其值的时候，就根据表达式计算出来结果，并把结果保存下来，不再更改，以备下次访问时不再重复计算，直接快速地返回该结果值。

这两个操作，应该最好是一体的，可以写在 **一处地方**，方便理解和修改。所以关键点在于如何把 一个变量 和 一个表达式 糅合到一起变成一个惰性求值。

而通常一般的编程语言里对属性和方法（函数）的访问形式有括号和没括号的区别，比如 object.attribute 和 object.method() ，那两者就统一不到一块了。

我们先不急着看各个编程语言如何表达惰性求值，而是先从我们要解决的离线数据任务看起。


### 离线数据任务 的宏观和微观
从宏观上来说，离线数据任务，尤其是算法模型，往往由多个任务组成。最简单的都有三个流程:

1. 数据采集
2. 数据模型（包括 预处理 + 训练 + 测试验证 等）
3. 数据推送

这些任务都会有它们各自的输入输出和运算逻辑，而且它们之间有执行上的前后依赖关系，那这就
形成了一个有向无环图，简称 DAG ([Directed acyclic graph][Directed_acyclic_graph])。
在任务调度的解决方案里，像 Python 社区里主要有 [Luigi][Luigi] 和 [Airflow][Airflow] 两大解决方案，限于主题原因这里就不具体展开了。

DAG 任务调度 在实际的数据工作中时间占比极大，可以说团队中的数据合作都是依靠它才能顺利运作。

DAG 的设计标准 在本质上也是现实企业中其项目任务分解在软件架构上的映射:

1. 清晰的树结构。是近似于树形的有向无环图，确保分解的任务都有专业的人去执行，并可以有条不紊地收集执行结果。
2. 易懂的平铺性。得益于树结构，所有拓扑关系完全可以在一个二维视野里表达，而不会发生意大利面条式缠绕难解的现象。
3. 可替换性。每个节点的职能都是定义好了的，只需要能执行出符合需求规范的结果即可，所以每个节点都是可以被替换的，

那这就是说 DAG 具有对应于软件工程上解耦的特点，给软件和数据维护带来了极大的便利。

所以我们能否在微观层面，也即是在具体某个软件模块编写的层面上，是否也能引入 解耦和易懂的 DAG 呢？答案是可以的。


### DAG 结构 和 惰性求值 的显性关系

要形成 数据任务调度的 DAG，必须有如下三个要素:

1. 数据结构固定。输入输出的数据 schema 必须是固定的，或者说上下游的节点必须是能自动处理该 schema 的。
2. 重复性。这个重复性通常是时间属性，比如按天粒度运行的任务。
3. 具体实现可变。只要满足上下游节点接受的数据结构，那么该节点的数据内容，算法，软件库，编程语言 等其实都是可以变换或自由选择的。

而这三个要素，在经典的面向对象软件设计里，完全是直接映射过来的。这里拿 类(class) 来举例子:

1. 数据结构固定。不管静态或动态类型，一个类里的不同方法接受的参数总是某一个指定类型（或某个模式）的，否则就会提示异常。
2. 重复性。既然抽象成类了，那么应该是会被多次初始化，以来处理多个相同类型的数据对象需求的。
3. 具体实现可变。这是软件开发最大的好处，内部对外部来说应该都是透明的，只要保持接口不变就好。

再进一步想，类其实是编程语言里的基本元素，而且一个类里也会调用其他类的实例(instance)，所以我们整个程序设计都是可以遵照
DAG 结构来设计得更解耦和易懂。

可能有些读者会发出这样的疑问，那么 DAG 和 惰性求值 有什么关系呢？

如果一个类只有一个地方使用到了惰性求值，那么确实关系不大。而如果一个类里使用了多个惰性求值（我相信需求总是复杂的:P)，然后这几个惰性求值之间有一些引用依赖，
那么其实就相当于隐式声明了 DAG 依赖图，它们是通过直接呼叫对方变量名实现的。相当于在执行一个惰性求值的时候，如果这个惰性求值也调用了其他惰性求值，并以此类推，
那么它们就会形成一个 **自动的调用链**，其实这就形成了一个有向无环图 DAG。

与原来的在类似于在 main 函数里去写几行到十几行关于各种变量的赋值，并注意执行顺序上的前后依赖。
现在使用惰性求值后，我们只需要直接呼叫 **最终** 要的那个变量名(以惰性求值形式)就可以了，它依赖的所有其他值都会以
DAG 的形式被自动调用起来。

最后对参数做一个说明，既然是惰性求值了，那就意味着这个属性是不能带参数了，参数只能以上下文的形式绑定，比如 instance attribute/variable/member 。
如果一个数据真的需要参数，那就说明还需要对这个类做进一步拆解成更小的粒度。


### 举一个常见的例子

来看一个很简单和正常的程序设计，我们设计了一个入口 Project，它会调用 Task 来处理一些事情。

```python
class Project:

  def __init__(self):
    self.task = Task()

  def process(self, request):
    params = extract_from(request)
    return self.task.compute(params)


class Task:

  def __init__(self):
    self.data = self.load_complex_data()

  def compute(self, params):
    return func(params, self.data)
```

粗看一下好像并没有什么问题，但是这个程序的启动时间其实是可以加快的，因为在
初始化 Project 的时候，同时也跟着初始化了 Task，而 Task 里的有一个 data 属性是需要花长时间来载入的。
如果我们想赶快进入调试，那么必须得把 Task 的 data 延迟加载。那么很多人都会接着这么设计的:

```python
class Task:

  def compute(self, params):
    if self.data is None:
      self.data = self.load_complex_data()
    return func(params, self.data)
```

然后如果程序里有很多地方都需要延迟加载，那就会出现很多个 IF 判断，程序的可读性因此大大降低。


### Python 的正确解法

```python
from cached_property import cached_property

class Project:

  def process(self, request):
    params = extract_from(request)
    return self.task.compute(params)

  @cached_property
  def task(self):
    return Task()

class Task:

  def compute(self, params):
    return func(params, self.data)

  @cached_property
  def data(self):
    // load complex data
```

这个写法比上面好的地方在于:

1. 对于定义方来说，直接把 变量名 data 和 load_complex_data 函数 统一为
   **同一个** instance method 的名字和内容了，只需要在头部加上一个
   @cached_property 装饰器即可。
2. 对于调用方来说，只需要像访问一个属性一样去访问就好了，而不需要加额外的括号来显式执行它。
3. 对于两者来说，定义方的参数和执行过程无论怎样改变，对于调用方来说，只需要知道名字就可以了，什么都不需要修改。

而 Python 是如何做到这些的呢? 是因为在 Python 的对象系统里有两个特性:

1. `__get__` 。如果一个属性使用 [property][Python_property] 包装了 getter (可选的还有 setter 和 deleter)，
   那么就可以直接以属性形式(不加括号)访问了。但是注意, 这个 property 是没有缓存功能的。
2. `__dict__` 。这个属性是继承了 Python object 根类就会有的，其作用可以用来存放当前对象的各种缓存。在我们上面用的
   [cached_property][cached_property] 里有具体的[实现][cached_property_source_file]，它弥补了 Python 内置 property
   不支持缓存的现状，是一个接近完美的解决方案，可惜不是 Python 内置的。

另外还有一个原因就是 Python 是一个动态类型语言，所以我们这里一个 class 里面的同名 method 可以在运行时被动态的替换成
property 类型。同时因为 Python 的 method 是必须显式声明至少一个参数，即是 self 当前 instance 自身，
所以就可以保证在替换的 property 里还可以继续保持对 当前对象的引用，然后可以利用这个上下文来做对当前对象的各种访问。

这种惰性求值 property 在 Python 这个动态语言里还有另一个好处，就是用于测试时 mock 数据。
实际运行时用函数去调用各种真实的资源来计算结果，而测试时，只需要把这个 property 替换成一个静态值就可以很方便测试了，比如 JSON 或列表之类的。

最后总结一下, @cached_property 在使用上是简单和没有歧义的，但是理解它的实现还是有点复杂的，主要是因为
Python 自身语法的简单性，和需要去协调函数对象和类系统之间的统一。



### Java 的解决方案

Java 作为静态语言，它是没有那些支持动态函数的东西可以灵活地去实现 名字和表达式的统一 这种高级特性。
但是它有一个 Annotation (注解) 功能可以实现和 Python 装饰器 类似的功能。

Lombok 是 Java 社区常用的库（虽然我个人没有用过。。。），它的其中一个
Annotation @Getter(lazy=true) 就是专门用来惰性求值的，我从它 [官网][Lombok_getter_lazy] 的例子拷贝如下：

```java
import lombok.Getter;

public class GetterLazyExample {
  @Getter(lazy=true) private final double[] cached = expensive();

  private double[] expensive() {
    double[] result = new double[1000000];
    for (int i = 0; i < result.length; i++) {
      result[i] = Math.asin(i);
    }
    return result;
  }

}
```

而它的访问方式就是 Java 一般约定的访问属性方式，即执行 `getCached()` ，来返回 double[] 类型的结果。
所以可以看出，表达语法是简单和易懂的，但是属性名和访问名字不能统一，不过这也不算大碍。



### Scala 的语言级别解法


```scala
class Project {
  def process(request: Request) = {
    val params = extract_from(request)
    task.compute(params)
  }

  lazy val task = {
    new Task()
  }
}

class Task {
  def compute(params: Params) = {
    ... compute params with data
  }

  lazy val data = {
    .. load complex data
  }

}
```

Scala 作为 JVM 语言，其都会编译生成对应的 Java 代码，所以可想而知其代码逻辑应该是相仿的。
但是它的书写方式却是异常简洁，完爆 Python 和 Java，直接把一个表达式赋予给一个 lazy val 就可以了。


### Ruby 的解决方案

```ruby
class Project
  def process(request)
    params = extract_from(request)
    task.compute(params)
  end

  def task
    @_task ||= new Task()
  end
end

class Task
  def compute(params)
    ... compute params with data
  end

  def data
    @_data ||= .. load complex data
  end
end
```

Ruby 的解法也是内置的，

1. 无参数的方法访问是可以不加括号就可以调用的。这点和 Scala 是一样的。因此可以完成属性和方法调用的无缝转换。
2. 相比其他语言需要先初始化一个空变量，然后再判断赋予真实值。在 Ruby 里
   只要简单的 `||=` 就可以把一个 instance attribute 初始化和赋值直接搞定。

缺点是 `@_data` 不兼容 false，导致 `||=` 就会再次进行计算。不过这个可以通过一个第三方库 [memoizable][memoizable] 来解决。


```ruby
require 'memoizable'

class Planet
  include Memoizable
  def spherical?
    4 * Math::PI * radius ** 2 == area
  end
  memoize :spherical?
end
```

memoizable 的原理是用一个 字典 去代理缓存，和 Python 里 cached_property 的解法是一样的，这样就兼容了 false 值的情况。
从书写上来说，和 Java 的 @Getter(lazy=true) 复杂度相当。都需要引入一个第三方库，用一个专门的声明去表示这是一个惰性求值。

另外，Ruby 里还可以根据其对象系统里的 [method_missing][Ruby_method_missing] 特性来做一个
[代理的惰性求值][lazy_object_pattern_in_ruby]。主要是这其实是一个相当于 mock 的做法，而不是返回一个真实的值。

### 最后的总结比较

最后我们来做一个表格来对比分析惰性求值在四个编程语言里使用上各种优劣，其中以 ABC 来表示从高到低的不同支持级别。

| 惰性求值    | 不需要括号|  定义在同一个地方 | 属性名字和表达式名字相同 | 语言内置 | 线程安全 |
|:-----------:|:---------:|:-----------------:|:------------------------:|:--------:|:--------:|
| Scala       |         A |                 A |                        A |        A |        A |
| Python      |         A |                 B |                        A |        B |        A |
| Java        |         C |                 C |                        B |        B |        A |
| Ruby        |         A |                 B |                        A |        B |        A |

综合排序是: Scala > Python/Ruby > Java。其实可以发现，这个排序也对应了编程语言的复杂程度的由高到低。
其中线程安全问题这四个语言的实现都是大同小异的，无非就是加把锁就搞定了，各个语言里都有内置标准库的实现。

通过本文分析，惰性求值解决了 数据量大 时程序初始化慢的问题，同时其平铺性和静态化表达式的特点 解耦了 数据逻辑之间依赖复杂 的问题。


> 注: 本文其实是从个人在 2015 年为自己用 Python 写的 [Luiti 离线任务调度框架][Luiti-homepage] 做的一篇
>     [演讲稿][luiti-an-offline-task-management-framework] 而延承过来的。虽然不同公司用的任务调度系统都各不相同，有用开源的，也有自研的，但是
>     DAG 和惰性求值的核心思想应该都是一致的。我只是表达了个人的经验，希望得到同行的不吝指正。


[Lazy_Evaluation]: https://en.wikipedia.org/wiki/Lazy_evaluation
[DRY]: https://en.wikipedia.org/wiki/Don%27t_repeat_yourself
[SOLID]: https://en.wikipedia.org/wiki/SOLID_(object-oriented_design)
[KISS_principle]: https://en.wikipedia.org/wiki/KISS_principle
[Eager_evaluation]: https://en.wikipedia.org/wiki/Eager_evaluation
[Directed_acyclic_graph]: https://en.wikipedia.org/wiki/Directed_acyclic_graph
[Luigi]: https://github.com/spotify/luigi
[Airflow]: https://github.com/apache/incubator-airflow
[Python_property]: https://docs.python.org/2/library/functions.html#property
[cached_property]: https://github.com/pydanny/cached-property
[cached_property_source_file]: https://github.com/pydanny/cached-property/blob/master/cached_property.py#L12
[Lombok_getter_lazy]: https://projectlombok.org/features/GetterLazy.html
[memoizable]: https://github.com/dkubb/memoizable
[lazy_object_pattern_in_ruby]: http://greyblake.com/blog/2014/10/05/lazy-object-pattern-in-ruby/
[Ruby_method_missing]: http://ruby-doc.org/core-2.1.0/BasicObject.html#method-i-method_missing
[luiti-an-offline-task-management-framework]: http://speakerdeck.com/mvj3/luiti-an-offline-task-management-framework
[Luiti-homepage]: https://luiti.github.io/
