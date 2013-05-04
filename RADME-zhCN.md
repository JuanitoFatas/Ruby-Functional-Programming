# Ruby 函数式编程 by Arnau Sanchez

本文档翻译自 Arnau Sanchez (tokland)所编译的这份文档 [RubyFunctionalProgramming](http://code.google.com/p/tokland/wiki/RubyFunctionalProgramming)。

同时也有[日文版本](http://www.h6.dion.ne.jp/~machan/misc/FPwithRuby.html)。

## 目录

* [简介](#-1)
* [理论部分](#-2)
* [Ruby的函数式编程](#ruby)
	* [不要更新变量](#-3)
	* [用 Blocks 作为高阶函数](#-blocks-)
	* [面向对象与函数式编程](#-6)
	* [万物皆表达式](#-7)
	* [递归](#-8)
	* [惰性枚举器](#-9)
	* [一个实际的例子](#-11)
* [结论](#-12)
* [简报](#-13)
* [延伸阅读](#-14)

## 简介

> 命令式编程比较牛吗？
> 不！不！不！只是比较快，比较简单，比较诱人而已。

	x = x + 1

在以前上小学的美好回忆裡，我们可能都曾对上面这个式子感到困惑。这个 `x` 到底是什么呢？为什么加了一之后，`x` 仍然还是 `x`。

不知道为什么，我们就开始写程序了，也就不在乎这是为什么了。心想：“嗯”，“这不是什么大问题，编程就是事情做完最重要，没有必要去挑剔数学的纯粹性 （让大学裡的大鬍子教兽们去烦恼就好）” 。但我们错了，也因此付出极高的代价，只因我们不了解它。

## 理论部分

[维基百科](http://en.wikipedia.org/wiki/Functional_programming)的解释：“函数式编程是一种写程序的范式，将计算视为对数学函数的求值，并避免使用状态及可变的数据” 换句话说，函数式编程提倡没有副作用的代码，不改变变量的值。这与命令式编程相反，命令式编程强调改变状态。

令人惊讶的是，函数式编程就这样而已。那…有什么好处呢？

* 更简洁的代码：“变量”一旦定义之后就不再改动，所以我们不需要追踪变量的状态，就可以理解一个函数、方法、类别、甚至是整个项目是怎么工作的。

* 引用透明：表达式可以用本身的值换掉。如果我们用同样的参数调用一个函数，我们确信输出会是一样的结果（没有其它的状态可改变它的值）。这也是为什么爱因斯坦说：“重复做一样的事却期望不同的结果”是疯狂的理由。

引用透明打开了前往某些美妙事物的大门

* 并行化：如果调用函数是各自独立的，则他们可以在不同的进程甚至是机器裡执行，而不会有竞态条件的问题。“平常” 写并发程序讨厌的细节（锁、semaphore…等）在函数式编程裡面通通消失不见了。

* 记忆化：由于函数调用的结果等于它的返回值，我们可以把这些值缓存起来。

* 模组化：代码裡不存有状态，所以我们可以将项目用小的黑箱连结起来，函数式编程提倡自底向上的编程风格。

* 容易调试：函数彼此互相隔离，只依赖输入与输出，所以很容易调试。

## Ruby的函数式编程

一切都是这么美好，但怎样才能将函数式编程，应用到每天写 Ruby（Ruby 不是个函数式语言）的程序开发裡呢？函数式编程广义来说，是一种风格，可以用在任何语言。当然啦，用在特别为这种范式打造的语言裡显得更自然，但某种程度上来说，可以应用到任何语言。

让我们先釐清这一点：本文没有要提倡古怪的风格，比如仅仅为了要延续理论函数式编程的纯粹性所带来的古怪风格。反之，我想说的重点是，我们应该 **当可以提昇代码的品质的时候，才使用函数式编程** ，不然这只不过是个糟糕的解决办法。

### 不要更新变量

别更新它们，创造新的变量。

#### 不要对数组或字串做 `append`

No:

```Ruby
indexes = [1, 2, 3]
indexes << 4
indexes # [1, 2, 3, 4]
```

Yes：

```Ruby
indexes = [1, 2, 3]
all_indexes = indexes + [4] # [1, 2, 3, 4]
```

#### 不要更新 hash

No:

```Ruby
hash = {:a => 1, :b => 2}
hash[:c] = 3
hash
```

Yes:

```Ruby
hash = {:a => 1, :b => 2}
new_hash = hash.merge(:c => 3)
```

#### 牵扯到内存位置的地方，不要使用破坏性方法。

No:

```Ruby
string = "hello"
string.gsub!(/l/, 'z')
string # "hezzo"
```

Yes:

```Ruby
string = "hello"
new_string =  string.gsub(/l/, 'z') # "hezzo"
```

#### 如何累积值

No:

```Ruby
output = []
output << 1
output << 2 if i_have_to_add_two
output << 3
```

Yes:

```Ruby
output = [1, (2 if i_have_to_add_two), 3].compact
```

### 用 Blocks 作为高阶函数

如果一个语言要搞函数式，会需要高阶函数。高阶函数是什么？函数可以接受别的函数作为参数，并可以返回函数，就这么简单。

Ruby (与 Smalltalk 还有其它语言）在这个方面上非常特别，语言本身就内置这个功能： **blocks** 区块。区块是一段匿名的代码，你可以随意的传来传去或是执行它。让我们看区块的典型用途，来构造函数式编程的构造子。

#### init-empty + each + push = map

No:

```Ruby
dogs = []
["milu", "rantanplan"].each do |name|
  dogs << name.upcase
end
dogs # => ["MILU", "RANTANPLAN"]
```

Yes:

```Ruby
dogs = ["milu", "rantanplan"].map do |name|
  name.upcase
end # => ["MILU", "RANTANPLAN"]
```
#### init-empty + each + conditional push -> select/reject


No:

```Ruby
dogs = []
["milu", "rantanplan"].each do |name|
  if name.size == 4
    dogs << name
  end
end
dogs # => ["milu"]
```

Yes:

```Ruby
dogs = ["milu", "rantanplan"].select do |name|
  name.size == 4
end # => ["milu"]
```

#### initialize + each + accumulate -> inject

No:

```Ruby
length = 0
["milu", "rantanplan"].each do |dog_name|
  length += dog_name.length
end
length # => 14
```

Yes:

```Ruby
length = ["milu", "rantanplan"].inject(0) do |accumulator, dog_name|
  accumulator + dog_name.length
end # => 14
```

在这个特殊情况下，当累积器与元素之间有操作进行时，我们不需要区块，只要将操作传给符号即可。

```Ruby
length = ["milu", "rantanplan"].map(&:length).inject(0, :+) # 14
```

#### empty + each + accumulate + push -> scan

想像一下，你不仅想要摺迭(fold)的结果，也想要过程中产生的部分数值。用命令式编程风格，你可能会这么写：

```Ruby
lengths = []
total_length = 0
["milu", "rantanplan"].each do |dog_name|
  lengths << total_length
  total_length += dog_name.length
end
lengths # [0, 4, 14]
```

在函数式的世界裡，Haskell 称之为 [scan](http://zvon.org/other/haskell/Outputprelude/scanl_f.html), C++ 称之为 [partial_sum](http://www.cplusplus.com/reference/std/numeric/partial_sum/), Clojure 称之为 [reductions](http://clojuredocs.org/clojure_core/clojure.core/reductions)。

令人讶异的是，Ruby 居然没有这样的函数！让我们自己写一个。这个怎么样：

```Ruby
lengths = ["milu", "rantanplan"].partial_inject(0) do |dog_name|
  dog_name.length
end # [0, 4, 14]
```

Enumerable#partial_inject 可以这么实现：

```Ruby
module Enumerable
  def partial_inject(initial_value, &block)
    self.inject([initial_value, [initial_value]]) do |(accumulated, output), element|
      new_value = yield(accumulated, element)
      [new_value, output << new_value]
    end[1]
  end
end
```

实作的细节不重要，重要的是，当认出一个有趣的模式可以被抽象化时，我们将其写在另一个函式库，撰写文档，反覆测试。现在只要让实际的需求去完善你的扩充即可。

#### initial assign + conditional assign + conditional assign + ...

这样的程序我们常常看到：

```Ruby
name = obj1.name
name = obj2.name if !name
name = ask_name if !name
```

在此时你应该觉得这样的代码使你很不自在（一个变量一下是这个值，一下是这个；变量名 `name` 到处都是…等）。函数式的方式更简短，也更简洁：

```Ruby
name = obj1.name || obj2.name || ask_name
```

另一个有更复杂条件的例子：

```Ruby
def get_best_object(obj1, obj2, obj3)
  return obj1 if obj1.price < 20
  return obj2 if obj2.quality > 3
  obj3
end
```

可以写成像是这样的一个表达式：

```Ruby
def get_best_object(obj1, obj2, obj3)
  if obj1.price < 20
    obj1
  elsif obj2.quality > 3
    obj2
  else
    obj3
  end
end
```

确实有一点囉嗦，但逻辑比一堆行内 `if/unless` 来得清楚。经验法则告诉我们，仅在你确定会用到副作用时，使用行内条件式，而不是在变量赋值或返回的场合使用：

```Ruby
country = Country.find(1)
country.invade if country.has_oil?
# more code here
```

#### 如何从 enumerable 创造一个 hash

Vanilla Ruby 没有从 Enumerable 转到 Hash 的直接对应（本人认为是一个遗憾的缺陷）。这也是为什么新手持续写出下面这个糟糕的模式(而你怎么能责怪他们呢？唉！）：

```Ruby
hash = {}
input.each do |item|
  hash[item] = process(item)
end
hash
```

这真的非常可怕！阿～～～！但手边有没有更好的办法呢？过去 Hash 构造子需要一个有着连续键值对的 flatten 集合 （阿，用 flatten 数组来描述映射？Lisp 曾这么做，但还是很丑陋）。幸运的是，Ruby 的最新版本也接受键值对，这样更有意义（作为 `hash.to_a` 的逆操作），现在你可以这么写：

```Ruby
Hash[input.map do |item|
  [item, process(item)]
end]
```

不赖嘛，但这打破了平常的撰写顺序。在 Ruby 我们期望从左向右写，给对象调用方法。而“好的”函数式方式是使用 `inject`：

```Ruby
input.inject({}) do |hash, item|
  hash.merge(item => process(item))
end
```

我们都同意这还是很囉嗦，所以我们最好将它放在 Enumerable 模组，[Facets](http://rubyworks.github.com/facets/) 正是这么干的。它称之为 Enumerable#mash：

```Ruby
module Enumerable
  def mash(&block)
    self.inject({}) do |output, item|
      key, value = block_given? ? yield(item) : item
      output.merge(key => value)
    end
  end
end
```

```Ruby
["functional", "programming", "rules"].map { |s| [s, s.length] }.mash
# {"functional"=>10, "programming"=>11, "rules"=>5}
```

或使用 `mash` 及 选择性区块来一步完成：

```Ruby
["functional", "programming", "rules"].mash { |s| [s, s.length] }
# {"functional"=>10, "programming"=>11, "rules"=>5}
```

### 面向对象与函数式编程

[Joe Armstrong][JA] (Erlang 发明人) 在 “Coders At work” 谈论过面向对象编程的重用性：

“我认为缺少重用性是面向对象语言造成的，而不是函数式语言。面向对象语言的问题是，它们带着语言执行环境的所有隐含资讯四处乱窜。你想要的是香蕉，但看到的却是香蕉拿在大猩猩手裡，而大猩猩的后面是整个丛林”

公平点说，我的看法是这不是面向对象编程的本质问题。你可以写出函数式的面向对象程序，但确定的是：

* 典型的 OOP 倾向强调改变对象的状态。
* 典型的 OOP 倾向层与层之间紧密的耦合。
* 典型的 OOP 将同一性(identity)与状态的概念搞溷了。
* 数据与代码的混合物，导致了概念与实际的问题产生。

[Rich Hickey][RH]，Clojure 的发明人（一个给 JVM 用的函数式 Lisp 方言），在这场[出色的演讲](http://www.infoq.com/presentations/Value-Identity-State-Rich-Hickey)裡谈论了状态、数值以及同一性。

### 万物皆表达式

可以这么写：

```Ruby
if found_dog == our_dog
  name = found_dog.name
  message = "We found our dog #{name}!"
else
  message = "No luck"
end
```

然而，控制结构（`if`, `while`, `case` 等）也返回表达式，所以只要这样写就好：

```Ruby
message = if found_dog == my_dog
  name = found_dog.name
  "We found our dog #{name}!"
else
  "No luck"
end
```

这样子我们不用重复变量名 `message`，企图也更明显：当有段长的程序（用了一堆我们不在乎的变量），我们可以专注在程序在干什么（返回讯息）。再强调一次，我们在缩小程序的作用域。

另一个函数式程序的好处是，表达式可以用来构造数据：

```Ruby
{
  :name => "M.Cassatt",
  :paintings => paintings.select { |p| p.author == "M.Cassatt" },
  :birth => painters.detect { |p| p.name == "M.Cassatt" }.birth.year,
  ...
}
```

### 递归

纯函数式语言没有隐含的状态，大量利用了递归。为了避免栈溢出，函数式使用一种称为尾递归优化(TCO)的机制。Ruby 1.9 有实作这种机制，但缺省没有打开。要是你希望你的程序，在哪都可以动的话，就不要使用它。

但是某些情况下，递归仍然是很有用的，即便是每次递归时都创建新的栈。注意！某些递归的用途可以用 foldings 来实现(像 Enumerable#inject)。

在 MRI-1.9 启用 TCO：

```Ruby
RubyVM::InstructionSequence.compile_option = {
  :tailcall_optimization => true,
  :trace_instruction => false,
}
```

简单例子：

```Ruby
module Math
  def self.factorial_tco(n, acc=1)
    n < 1 ? acc : factorial_tco(n-1, n*acc)
  end
end
```

在递归深度不太可能很深的情况下，你仍可以使用：

```Ruby
class Node
  has_many :children, :class_name => "Node"

  def all_children
    self.children.flat_map do |child|
      [child] + child.all_children
    end
  end
end
```

### 惰性枚举器

惰性求值延迟了表达式的求值，在真正需要时才会求值。与 eager evaluation 相反，eager evaluation 当一个变量被赋值时、函数被调用时…甚至根本没用到变量等状况，都立马对表达式求值，惰性不是函数式编程的必需品，但这是个符合函数式范式的好策略（Haskell 大概是最佳的例子，瀰漫着懒惰的语言）。

Ruby 所採用的基本上是 eager evaluation（虽然许多其它的语言，在条件还没满足前不对表达式求值，以及短路布林运算 `&&`, `||` 等）。然而，与任何内置高阶函数的语言一样，延迟求值是隐性支援，因为程序员自己决定区块何时被调用。

Enumerators 同样 从 Ruby 1.9 开始支援(1.8 请用 backports)，它们提供了一个简单的介面来定义惰性 enumerables。经典的例子是构造一个枚举器，返回所有的自然数：

```Ruby
require 'backports' # 1.8 才需要
natural_numbers = Enumerator.new do |yielder|
  number = 1
  loop do
    yielder.yield number
    number += 1
  end
end
```

可以用更函数式的精神改写：

```Ruby
natural_numbers = Enumerator.new do |yielder|
  (1..1.0/0).each do |number|
    yielder.yield number
  end
end
```

```Ruby
natural_numbers.take(10)
# [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

现在，试试给 `natural_numbers` 做 `map`，发生什么事？它不会停止。标准的 enumerable 方法 (`map`, `select` 等）返回一个数组，所以在输入流是无穷大时，无法正常工作。让我们扩展 Enumerator 类别，比如加入这个惰性的 Enumerator#map：


```Ruby
class Enumerator
  def map(&block)
    Enumerator.new do |yielder|
      self.each do |value|
        yielder.yield(block.call(value))
      end
    end
  end
end
```

现在我们可以给所有自然数的流做 `map` 了：

```Ruby
natural_numbers.map { |x| 2*x }.take(10)
# [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
```

枚举器是用来构造惰性行为的区块的好东西，但你可以使用用懒惰风格，实作了所有 enumerable 方法的函式库：


https://github.com/yhara/enumerable-lazy

```Ruby
require 'enumerable/lazy'
(1..1.0/0).lazy.map { |x| 2*x }.take(10).to_a
# [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
```

#### 惰性求值的好处

1. 显而易见的好处: 无需在不必要的情况下，构造、储存完整的结构（也许，可以更有效率的使用 CPU 及内存）

2. 不太显而易见的好处: 惰性求值使写程序不需要了解超出你所需的范围。让我们看一个例子：你写了某种解题工具，可以提供无数种解法，但在某个时候，你只想要前十种解法。你可能会这么写：

```Ruby
solver(input, :max => 10)
```

当你与惰性结构一起工作时，不需要说什么时候该结束。调用者自己会决定他需要多少值。代码变得更简单，责任归属到对的地方，也就是调用者：

```Ruby
solver(input).take(10)
```

### 一个实际的例子

练习：“前十个平方可被五整除的自然数的和是多少？”

```Ruby
Integer::natural.select { |x| x**2 % 5 == 0 }.take(10).inject(:+) #=> 275
```

让我们跟等价的命令式版本来比较：

```Ruby
n, num_elements, sum = 1, 0, 0
while num_elements < 10
  if n**2 % 5 == 0
    sum += n
    num_elements += 1
  end
  n += 1
end
sum #=> 275
```

我希望这个例子展示了这个文档裡讨论的函数式编程的优点：

1. 更简洁: 你会撰写更少的代码。函数式程序处理的是表达式，而表达式可以连锁起来；命令式程序处理的是变量的改动（叙述式），而这不能连锁。

2. 更抽象: 你可以争论我们使用 `select`, `inject`…等等，来隐藏了一大堆代码，我很高兴你这么说，因为我们正是这么干的。将通用的、可重用的代码隐藏起来，这是所有编程的重点 –– 但函数式编程特别是关于如何撰写抽象。感到开心不是因为写了更少的代码，而是因为藉由认出可重用的模式，简化了代码的复杂性。

3. 更有声明式的味道: 看看命令式的版本，第一眼看起来是一沱无用的代码 –– 没有注解的话 –– 它会做什么你完全没有概念。你可能会说：“好吧，从这裡开始读，草草记下 `n` 与 `sum` 的值，进入某个迴圈，看看 `n` 与 `sum` 的值如何变化，看看最后一次迭代的情形” 等等。函数式版本另一方面是自我解释的，函数式版本描述、声明它在干的事，而不是如何干这件事。

“函数式编程就像是将你的问题叙述给数学家一样。命令式编程像是给白痴下指令” (arcus 在 Freenode #scheme 频道所说）

### 结论

更好的理解函数式编程的原理，帮助我们写出更清晰、重用性更高并更简洁的代码。Ruby 基本上是一个命令式语言，但它也有很大的函数式能力，明白什么时候用，及如何用（以及何时不该用）这些能力。将这句话当成你的座右铭吧 “状态是万恶的根源，尽可能避免它。”

### 简报

Workshop at [Conferencia Rails 2011](http://conferenciarails.org/): [Functional Programming with Ruby](http://public.arnau-sanchez.com/ruby-functional/) [(slideshare)](http://www.slideshare.net/tokland/functional-programming-with-ruby-9975242)

### 延伸阅读

http://en.wikipedia.org/wiki/Functional_programming

http://www.defmacro.org/ramblings/fp.html __[译文](http://t.cn/zYaCDw7)__

http://www.cse.chalmers.se/~rjmh/Papers/whyfp.html

http://www.khelll.com/blog/ruby/ruby-and-functional-programming/

http://www.bestechvideos.com/2008/11/30/rubyconf-2008-better-ruby-through-functional-programming

http://channel9.msdn.com/Blogs/pdc2008/TL11

http://www.infoq.com/presentations/Value-Identity-State-Rich-Hickey

## 授权

This document is licensed under the CC-By 3.0 License, which encourages you to share these documents. See http://creativecommons.org/licenses/by/3.0/ for more details.

<img alt="CC-By 3.0 License http://creativecommons.org/licenses/by/3.0/" style="border-width:0" src="http://i.creativecommons.org/l/by/3.0/88x31.png" />

[RH]: https://twitter.com/richhickey
[JA]: http://www.sics.se/~joe/
