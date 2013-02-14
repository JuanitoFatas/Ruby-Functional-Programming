# Functional Programming with Ruby by Arnau Sanchez

This document is compiled from [RubyFunctionalProgramming](http://code.google.com/p/tokland/wiki/RubyFunctionalProgramming) by Arnau Sanchez (tokland)

[Japanese version](http://www.h6.dion.ne.jp/~machan/misc/FPwithRuby.html) is also available.

## Table of Contents

* [Introduction](#introduction)
* [The theory](#the-theory)
* [Functional programming in Ruby](#functional-programming-in-ruby)
	* [Don't update variables](#dont-update-variables)
	* [Blocks as higher order functions](#blocks-as-higher-order-functions)
	* [OOP and funcional programming](#oop-and-funcional-programming)
	* [Everything is an expression](#everything-is-an-expression)
	* [Recursion](#recursion)
	* [Lazy enumerators](#lazy-enumerators)
	* [A practical example](#a-practical-example)
* [Conclusion](#conclusion)
* [Presentations](#presentations)
* [Further reading](#further-reading)

## Introduction

> Is the imperative programming stronger?
> No, no, no. Only quicker, easier, more seductive

	x = x + 1

In the good old days in primary school we would have been puzzled by this line. Which magical x is this that can be added one and yet remain unchanged. Somehow, we started programming and we didn't mind anymore. 'Well', we thought, 'that's not a serious issue, programming is about getting real business done and there's no need to quibble over mathematic purity' (let the loony bearded guys in universities deal with it). But it turns out we were wrong, we were paying a high price, only that we didn't know it.

## The theory

From the [Wikipedia](http://en.wikipedia.org/wiki/Functional_programming): "Functional programming is a programming paradigm that treats computation as the evaluation of mathematical functions and avoids state and mutable data". In other words, functional programming promotes code with no side effects, no change of value in variables. It oposes to imperative programming, which enfatizes change of state.

Surprisingly, that's all there is to it. So what are the advantages?

* Cleaner code: "variables" are not modified once defined, so we don't have to follow the change of state to comprehend what a function, a, method, a class, a whole project works.

* Referential transparency: Expressions can be replaced by its values. If we call a function with the same parameters, we know for sure the output will be the same (there is no state anywhere that would change it). There is a reason for which Einstein defined insanity as "doing the same thing over and over again and expecting different results".

Referential transparency opens the gate to some nice things:

* Parallelization: If calls to functions are independent, they may be executed in different processes or even machines with no race-condition issues. All the nasty details of "normal" concurrency code (locks, semaphores, ...) just vanish on the functional paradigm.

* Memoization. Since a function call is equivalent to its return value, we may be cache them.

* Modularization: We have no state that pervades the whole code, so we build our project with small, black boxes that we tie together, so it promotes bottom-up programming.

* Ease of debugging: Functions are isolated, they only depend on their input and their output, so they are very easy to debug.

## Functional programming in Ruby

This is all great, but how can we apply it to our daily-programming in Ruby (which is, indeed, not a functional language)? FP is, in its general sense, a style, it may be used in any language. Of course it will be the more natural way on languages specially designed for the paradigm, but to some extend, it can be applied to any language.

Let's be clear about this: this guide does not pretend to promote bizarre style just to adhere to theoretical functional purity. On the contrary, the point that I am trying to make is that we should **use FP whenever it increases the quality of code**, otherwise it's just a bad solution.

### Don't update variables

Don't update them, just create new ones.

#### Don't append to arrays or strings

No:

```ruby
indexes = [1, 2, 3]
indexes << 4
indexes # [1, 2, 3, 4]
```

Yes:

```ruby
indexes = [1, 2, 3]
all_indexes = indexes + [4] # [1, 2, 3, 4]
```

#### Don't update hashes

No:

```ruby
hash = {:a => 1, :b => 2}
hash[:c] = 3
hash
```

Yes:

```ruby
hash = {:a => 1, :b => 2}
new_hash = hash.merge(:c => 3) 
```

#### Don't use bang methods which modify in-place

No:

```ruby
string = "hello"
string.gsub!(/l/, 'z')
string # "hezzo"
```

Yes:

```ruby
string = "hello"
new_string =  string.gsub(/l/, 'z') # "hezzo"
```

#### How to accumulate values

No:

```ruby
output = []
output << 1
output << 2 if i_have_to_add_two
output << 3
```

Yes:

```ruby
output = [1, (2 if i_have_to_add_two), 3].compact
```

### Blocks as higher order functions

If a language is to be used functionally we need higher-order functions. That's it, functions can take other functions as parameters, and can also return other functions.

Ruby (along with Smalltalk and some others) is special in this regard, the facility is built-in in the language: **blocks**. A block is an anonymous piece of code you can pass around and execute at will. Let's see a typical usage of blocks to build functional constructions.

#### init-empty + each + push = map

No:

```ruby
dogs = []
["milu", "rantanplan"].each do |name|
  dogs << name.upcase
end
dogs # => ["MILU", "RANTANPLAN"]
```

Yes:

```ruby
dogs = ["milu", "rantanplan"].map do |name|
  name.upcase
end # => ["MILU", "RANTANPLAN"]
```
#### init-empty + each + conditional push -> select/reject


No:

```ruby
dogs = []
["milu", "rantanplan"].each do |name|
  if name.size == 4
    dogs << name
  end
end
dogs # => ["milu"]
```

Yes:

```ruby
dogs = ["milu", "rantanplan"].select do |name|
  name.size == 4
end # => ["milu"]
```

#### initialize + each + accumulate -> inject

No:

```ruby
length = 0
["milu", "rantanplan"].each do |dog_name|
  length += dog_name.length
end
length # => 15
```

Yes:

```ruby
length = ["milu", "rantanplan"].inject(0) do |accumulator, dog_name|
  accumulator + dog_name.length
end # => 15
```

In this particular case, when there is a simple operation between accumulator and element, we don't need to write the block, just pass the operation symbol:

```ruby
length = ["milu", "rantanplan"].map(&:length).inject(0, :+) # 15
```

#### empty + each + accumulate + push -> scan

Imagine you don't want only the final result of a fold (the inject we saw before) but also the partial values. In imperative code you'd write:

```ruby
lengths = []
total_length = 0
["milu", "rantanplan"].each do |dog_name|
  lengths << total_length
  total_length += dog_name.length
end
lengths # [0, 4, 15]
```

In the functional world, Haskell calls it [scan](http://zvon.org/other/haskell/Outputprelude/scanl_f.html), C++ calls it [partial_sum](http://www.cplusplus.com/reference/std/numeric/partial_sum/), Clojure calls it [reductions](http://clojuredocs.org/clojure_core/clojure.core/reductions). Ruby, surprisingly, has no such function, let's write our own. How about that:

```ruby
lengths = ["milu", "rantanplan"].partial_inject(0) do |dog_name|
  dog_name.length
end # [0, 4, 15]
```

Enumerable#partial_inject can be written:

```ruby
module Enumerable
  def partial_inject(initial_value, &block)
    self.inject([initial_value, [initial_value]]) do |(accumulated, output), element|
      new_value = yield(accumulated, element)
      [new_value, output << new_value]
    end[1]
  end
end
```

The details of the implementation are not important, what matters is that when we identify a interesting pattern to be abstracted, we wrote it in a separate library, we documented it, we tested it. And now simply let the real needs refine your extensions.

#### initial assign + conditional assign + conditional assign + ...

We see code like this all the time:

```ruby
name = obj1.name
name = obj2.name if !name
name = ask_name if !name
```

At this point you should feel uneasy before a code like this (a variable has now this value, now this other one; the variable name being repeated everywhere, ...). A functional approach is shorter and clearer:

```ruby
name = obj1.name || obj2.name || ask_name
```

Another example with more complex conditions:

```ruby
def get_best_object(obj1, obj2, obj3)
  return obj1 if obj1.price < 20
  return obj2 if obj2.quality > 3
  obj3
end
```

Can be written as a real expression like this:

```ruby
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

Indeed, a bit more verbose, but the logic is much more clear than a bunch of inline if/unless. As a rule of thumb, use inline conditionals only and only if you are doing a real side-effect, not variable assignation nor returns:

```ruby
country = Country.find(1)
country.invade if country.has_oil?
# more code here
```

#### How to create a hash from an enumerable

Vanilla Ruby has no direct translation from Enumerable to Hash (a sad flaw, in my opinion). That's why novices keep writing this terrible pattern (and how can you blame them?):

```ruby
hash = {}
input.each do |item|
  hash[item] = process(item)
end
hash
```

This is hideous. Period. But is there anything better at hand? on the past the Hash constructor required a flatten collection of consecutive key/value (ugh, a flatten array to describe a mapping? Lisp used to do this, but it's still ugly). Fortunately, latest versions of Ruby also take key/value pairs, which makes much more sense (as the reverse operation of hash.to_a), and now you can write:

```ruby
Hash[input.map do |item|
  [item, process(item)]
end]
```

Not bad, but it kind of breaks the natural writing directionality. In Ruby we expect to write from left to right, calling methods for objects. Whie the "good" functional way is to use inject:

```ruby
input.inject({}) do |hash, item|
  hash.merge(item => process(item))
end
```

We all agree this is still too verbose, so we better move it as a method in the module Enumerable, which is exactly what Facets does. They call it Enumerable#mash:

```ruby
module Enumerable
  def mash(&block)
    self.inject({}) do |output, item|
      key, value = block_given? ? yield(item) : item
      output.merge(key => value)
    end
  end
end
```

```ruby
["functional", "programming", "rules"].map { |s| [s, s.length] }.mash
# {"rules"=>5, "programming"=>11, "functional"=>10}
```

Or in a single step using mash using the optional block:

```ruby
[["functional", "programming", "rules"].mash { |s| [s, s.length] }]
# {"rules"=>5, "programming"=>11, "functional"=>10}
```

### OOP and funcional programming

Joe Armstrong (the creator of Erlang) discussed in "Coders At work" about the reusability of Object-Oriented Programming:

"I think the lack of reusability comes in object-oriented languages, not in functional languages. Because the problem with object-oriented languages is they’ve got all this implicit environment that they carry around with them. You wanted a banana but what you got was a gorilla holding the banana and the entire jungle."

To be fair, in my opinion it's not an intrinsic problems of OOP. You can write OOP code which is also functional, but certainly:

* Typical OOP tends to emphasize change of state in objects.
* Typical OOP tends to impose tight coupling between layers (which hinders modularization).
* Typical OOP mixes the concepts of identity and state.
* Mixture of data and code raises both conceptual and practical problems.

Rich Hickey, the creator of Clojure (a functional Lisp-dialect for the JVM), discusses state, values and identity in this [excellent talk](http://www.infoq.com/presentations/Value-Identity-State-Rich-Hickey).

### Everything is an expression

You may write this:

```ruby
if found_dog == our_dog 
  name = found_dog.name
  message = "We found our dog #{name}!"
else
  message = "No luck"
end
```

However, control-structures (if, while, case and so on) also return an expression, so let's just write:

```ruby
message = if found_dog == my_dog
  name = found_dog.name
  "We found our dog #{name}!"
else
  "No luck"
end
```

It's not only that we don't repeat the variable name message, also the intent is much more clear: while there is a bunch of code which may be large (and using a lot of other variables we don't really care about), we can concentrate on what it does (return a message). Again, we are narrowing down the scope of our code.

Another advantage, FP code, being expressions, can be used to build data:

```ruby
{
  :name => "M.Cassatt",
  :paintings => paintings.select { |p| p.author == "M.Cassatt" },
  :birth => painters.detect { |p| p.name == "M.Cassatt" }.birth.year,
  ...
}
```

### Recursion

Pure functional languages, having no implicit state, use recursion a lot. To avoid infinite stacks, functional languages have a mechanism called tail-recursion optimization (TCO). Ruby 1.9 has this mechanism coded but it's disabled by default, so you don't use it if you expect your code to work everywhere.

However, in certain circumstances recursion is still valid and usable, even if a new stack is created on each recursion. Note that some usages of recursion may be achieved with foldings (like Enumerable#inject).

To enable TCO in MRI-1.9:

```ruby
RubyVM::InstructionSequence.compile_option = {
  :tailcall_optimization => true,
  :trace_instruction => false,
}
```

Simple example:

```ruby
module Math
  def self.factorial_tco(n, acc=1)
    n < 1 ? acc : factorial_tco(n-1, n*acc)
  end
end
```

You still can use it when the recursion-depth is very unlikely to be large:

```ruby
class Node
  has_many :children, :class_name => "Node"

  def all_children
    self.children.flat_map do |child|
      [child] + child.all_children
    end
  end
end
```

### Lazy enumerators

Lazy evaluation delays the evaluation of the expression until it's needed, as opposed to eager evaluation, where expressions are calculated when a variable is assigned, a function called, etc, even if it's not really used anywhere. Laziness is not a requisite for FP, but it's a strategy that fits nicely on the paradigm (Haskell is probably the best example, laziness pervades the language).

Ruby uses, basically, eager evaluation (though as many other languages, it does not evaluate expressions on conditionals if not reached, it short-circuits boolean operators &&, ||, etc). However, as any language with high-order function, delayed evaluation is supported implicitly because the programmer decides when blocks are going to be called.

Also, enumerators are available from Ruby 1.9 (use backports for 1.8) and they provide a clean interface for defining lazy enumerables. The classical example is to build a enumerator that returns all the natural numbers:

```ruby
require 'backports' # needed only for 1.8
natural_numbers = Enumerator.new do |yielder|
  number = 1
  loop do
    yielder.yield number
    number += 1
  end
end
```

Which could be re-written in a more functional spirit:

```ruby
natural_numbers = Enumerator.new do |yielder|
  (1..1.0/0).each do |number|
    yielder.yield number
  end
end
```

```ruby
natural_numbers.take(10)
# [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

Now, try to do a map on natural_numbers, what happens? it never ends. Standard enumerable methods (map, select, etc) return an array so they won't work if the input is an infinite stream. Let's extend the Enumerator class, for example with this lazy Enumerator#map:

```ruby
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

And now we can do a map on our stream of all natural numbers:

```ruby
natural_numbers.map { |x| 2*x }.take(10)
# [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
```

Enumerators are great as building blocks of lazy behaviors, but you can use libraries that implement all the enumerable methods in a lazy fashion:

https://github.com/yhara/enumerable-lazy

```ruby
require 'enumerable/lazy'
(1..1.0/0).lazy.map { |x| 2*x }.take(10).to_a
# [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
```

#### Advantages of lazy evaluation

1. The obvious: you don't build and store complete structures if you don't need to (which may be efficient in CPU, memory, or both).

2. Not so obvious: Lazy evaluation makes possible to write code that does not know (not want to know) more that it needs to. Let's see an example: you wrote a solver of some kind that yields infinite number of solutions, but at some point you only want to get the first 10. You'd write something like:

```ruby
solver(input, :max => 10) 
```

When you are working with lazy structures there is no need to say when to stop. The caller decides how many values it wants. The code becomes simplier and the responsibility goes where it should be, to the caller:

```ruby
solver(input).take(10)
```

### A practical example

Exercise: "What's the sum of the first 10 natural number whose square value is divisible by 5?".

```ruby
Integer::natural.select { |x| x**2 % 5 == 0 }.take(10).inject(:+) #=> 275
```

Let's compare it with the equivalent imperative version:

```ruby
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

I hope this example shows some of the advantages we discussed in this document:

1. Compactness: You'll write less code. Functional code deals with expressions, and expressions are chainable; imperative code deals with variable modifications (statements), which are not chainable.

2. Abstraction: You can argue that we are hiding a lot of code when using select, inject, ...), and so on, and I am glad you brought it up because that's exactly what we are doing. Hiding generic, reusable code, that's what all programming -but specially functional programming- is about, about writing abstractions. We are not happy because we write less lines of code, we are happy because we reduced the complexity of our code by identifying reusable patterns.

3. More declarative: Look at the imperative version, it's an amorph bunch of code that at first glance -not being commented- you have absolute no idea what it may be doing. You may say: 'well, let's start here, jot down the values for n and sum, do some loops, see how they evolve, look at the last iteration' and so on. The functional version on the other hand is self-explanatory, it describes, it declares what it's doing, not how it's doing it.

"Functional programming is like describing your problem to a mathematician. Imperative programming is like giving instructions to an idiot" (arcus, #scheme on Freenode).

### Conclusion

A better understanding of the principles of Functional Programming will help us to write more clear, reusable and compact code. Ruby is basically an imperative language, but it also has great functional capabilities, know when and how to use them (and when not to). Be your motto "state is the root of all evil" and avoid it whenever possible.

### Presentations

Workshop at [Conferencia Rails 2011](http://conferenciarails.org/): [Functional Programming with Ruby](http://public.arnau-sanchez.com/ruby-functional/) [(slideshare)](http://www.slideshare.net/tokland/functional-programming-with-ruby-9975242)

### Further reading

http://en.wikipedia.org/wiki/Functional_programming

http://www.defmacro.org/ramblings/fp.html

http://www.cse.chalmers.se/~rjmh/Papers/whyfp.html

http://www.khelll.com/blog/ruby/ruby-and-functional-programming/

http://www.bestechvideos.com/2008/11/30/rubyconf-2008-better-ruby-through-functional-programming

http://channel9.msdn.com/Blogs/pdc2008/TL11

http://www.infoq.com/presentations/Value-Identity-State-Rich-Hickey

### LICENSE

This document is licensed under the CC-By 3.0 License, which encourages you to share these documents. See http://creativecommons.org/licenses/by/3.0/ for more details.

<img alt="CC-By 3.0 License http://creativecommons.org/licenses/by/3.0/" style="border-width:0" src="http://i.creativecommons.org/l/by/3.0/88x31.png" />
