# Ruby 函數式程式設計 by Arnau Sanchez

本文件翻譯自 Arnau Sanchez (tokland)所編譯的這份文件 [RubyFunctionalProgramming](http://code.google.com/p/tokland/wiki/RubyFunctionalProgramming)。

同時也有[日文版本](http://www.h6.dion.ne.jp/~machan/misc/FPwithRuby.html)。

## 目錄

* [簡介](#-1)
* [理論部分](#-2)
* [Ruby的函數式程式設計](#ruby)
	* [不要更新變數](#-3)
	* [用 Blocks 作為高階函數](#-blocks-)
	* [物件導向與函數式程式設計](#-6)
	* [萬物皆表達式](#-7)
	* [遞迴](#-8)
	* [惰性枚舉器](#-9)
	* [一個實際的範例](#-11)
* [結論](#-12)
* [簡報](#-13)
* [延伸閱讀](#-14)

## 簡介

> 命令式程式設計比較威嗎？
> 不！不！不！只是比較快，比較簡單，比較誘人而已。

	x = x + 1

在以前上小學的美好回憶裡，我們可能都曾對上面這個式子感到困惑。這個 `x` 到底是什麼呢？為什麼加了一之後，`x` 仍然還是 `x`。

不知道為什麼，我們就開始寫程式了，也就不在乎這是為什麼了。心想：“嗯”，“這不是什麼大問題，程式設計就是事情做完最重要，沒有必要去挑剔數學的純粹性 （讓大學裡的大鬍子教獸們去煩惱就好）” 。但我們錯了，也因此付出極高的代價，只因我們不了解它。

## 理論部分

[維基百科](http://en.wikipedia.org/wiki/Functional_programming)的解釋：“函數式程式設計是一種寫程式的範式，將計算視為對數學函數的求值，並避免使用狀態及可變的資料” 換句話說，函數式程式設計提倡沒有副作用的程式，不改變變數的值。這與命令式程式設計相反，命令式程式設計強調改變狀態。

令人驚訝的是，函數式程式設計就這樣而已。那…有什麼好處呢？

* 更簡潔的程式碼：“變數”一旦定義之後就不再改動，所以我們不需要追蹤變數的狀態，就可以理解一個函數、方法、類別、甚至是整個專案是怎麼工作的。

* 參照透明：表達式可以用本身的值換掉。如果我們用同樣的參數呼叫一個函數，我們確信輸出會是一樣的結果（沒有其它的狀態可改變它的值）。這也是為什麼愛因斯坦說：“重複做一樣的事卻期望不同的結果”是瘋狂的理由。

參照透明打開了前往某些美妙事物的大門

* 平行化：如果呼叫函數是各自獨立的，則他們可以在不同的進程甚至是機器裡執行，而不會有競態條件的問題。“平常” 寫 Concurrency 程式討厭的細節（鎖、semaphore…等）在函數式程式設計裡面通通消失不見了。

* 記憶化：由於函數呼叫的結果等於它的回傳值，我們可以把這些值快取起來。

* 模組化：程式碼裡不存有狀態，所以我們可以將專案用小的黑箱連結起來，函數式程式設計提倡自底向上的程式設計風格。

* 容易除錯：函數彼此互相隔離，只依賴輸入與輸出，所以很容易除錯。

## Ruby 的函數式程式設計

一切都是這麼美好，但怎樣才能將函數式程式設計，應用到每天寫 Ruby（Ruby 不是個函數式語言）的程式開發裡呢？函數式程式設計廣義來說，是一種風格，可以用在任何語言。當然啦，用在特別為這種範式打造的語言裡顯得更自然，但某種程度上來說，可以應用到任何語言。

讓我們先釐清這一點：本文沒有要提倡古怪的風格，比如僅僅為了要延續理論函數式程式設計的純粹性所帶來的古怪風格。反之，我想說的重點是，我們應該 **當可以提昇程式碼品質時，才使用函數式程式設計** ，不然這只不過是個糟糕的解決辦法。

### 不要更新變數

別更新它們，創造新的變數。

#### 不要對陣列或字串做 `append`

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

#### 牽扯到記憶體位置的地方，不要使用破壞性方法。

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

#### 如何累積值

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

### 用 Blocks 作為高階函數

如果一個語言要搞函數式，會需要高階函數。高階函數是什麼？函數可以接受別的函數作為參數，並可以回傳函數，就這麼簡單。

Ruby (與 Smalltalk 還有其它語言）在這個方面上非常特別，語言本身就內建這個功能： **blocks** 區塊。區塊是一段匿名的程式碼，你可以隨意的傳來傳去或是執行它。讓我們看區塊的典型用途，來建構函數式程式設計的建構子。

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

在這個特殊情況下，當累積器與元素之間有操作進行時，我們不需要區塊，只要將操作傳給符號即可。

```Ruby
length = ["milu", "rantanplan"].map(&:length).inject(0, :+) # 14
```

#### empty + each + accumulate + push -> scan

想像一下，你不僅想要摺疊(fold)的結果，也想要過程中產生的部分數值。用命令式程式設計風格，你可能會這麼寫：

```Ruby
lengths = []
total_length = 0
["milu", "rantanplan"].each do |dog_name|
  lengths << total_length
  total_length += dog_name.length
end
lengths # [0, 4]
```

在函數式的世界裡，Haskell 稱之為 [scan](http://zvon.org/other/haskell/Outputprelude/scanl_f.html), C++ 稱之為 [partial_sum](http://www.cplusplus.com/reference/std/numeric/partial_sum/), Clojure 稱之為 [reductions](http://clojuredocs.org/clojure_core/clojure.core/reductions)。

令人訝異的是，Ruby 居然沒有這樣的函數！讓我們自己寫一個。這個怎麼樣：

```Ruby
lengths = ["milu", "rantanplan"].partial_inject(0) do |dog_name|
  dog_name.length
end # [0, 4, 14]
```

`Enumerable#partial_inject` 可以這麼實現：

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

實作的細節不重要，重要的是，當認出一個有趣的模式可以被抽象化時，我們將其寫在另一個函式庫，撰寫文件，反覆測試。現在只要讓實際的需求去完善你的擴充即可。

#### initial assign + conditional assign + conditional assign + ...

這樣的程式我們常常看到：

```Ruby
name = obj1.name
name = obj2.name if !name
name = ask_name if !name
```

在此時你應該覺得這樣的程式碼使你很不自在（一個變數一下是這個值，一下是這個；變數名 `name` 到處都是…等）。函數式的方式更簡短，也更簡潔：

```Ruby
name = obj1.name || obj2.name || ask_name
```

另一個有更複雜條件的例子：

```Ruby
def get_best_object(obj1, obj2, obj3)
  return obj1 if obj1.price < 20
  return obj2 if obj2.quality > 3
  obj3
end
```

可以寫成像是這樣的一個表達式：

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

確實有一點囉嗦，但邏輯比一堆行內 `if/unless` 來得清楚。經驗法則告訴我們，僅在你確定會用到副作用時，使用行內條件式，而不是在變數賦值或回傳的場合使用：

```Ruby
country = Country.find(1)
country.invade if country.has_oil?
# more code here
```

#### 如何從 enumerable 創造一個 hash

Vanilla Ruby 沒有從 Enumerable 轉到 Hash 的直接對應（本人認為是一個遺憾的缺陷）。這也是為什麼新手持續寫出下面這個糟糕的模式(而你又怎麼能責怪他們呢？唉！）：

```Ruby
hash = {}
input.each do |item|
  hash[item] = process(item)
end
hash
```

這真的非常可怕！阿～～～！但手邊有沒有更好的辦法呢？過去 Hash 建構子需要一個有著連續鍵值對的 flatten 集合 （呃，用 flatten 陣列來描述映射？Lisp 曾這麼做，但還是很醜陋）。幸運的是，最新版本的 Ruby 也接受鍵值對，這樣更有意義（作為 `hash.to_a` 的逆操作），現在你可以這麼寫：

```Ruby
Hash[input.map do |item|
  [item, process(item)]
end]
```

不賴嘛，但這打破了平常的撰寫順序。在 Ruby 我們期望從左向右寫，給物件呼叫方法。而“好的”函數式方式是使用 `inject`：

```Ruby
input.inject({}) do |hash, item|
  hash.merge(item => process(item))
end
```

我們都同意這還是很囉嗦，所以我們最好將它放在 Enumerable 模組，[Facets](http://rubyworks.github.com/facets/) 正是這麼幹的。它稱之為 `Enumerable#mash`：

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

或使用 `mash` 及選擇性區塊來一步完成：

```Ruby
["functional", "programming", "rules"].mash { |s| [s, s.length] }
# {"functional"=>10, "programming"=>11, "rules"=>5}
```

### 物件導向與函數式程式設計

[Joe Armstrong][JA] (Erlang 發明人) 在 “Coders At work” 談論過物件導向程式設計的重用性：

“我認為缺少重用性是物件導向語言造成的，而不是函數式語言。物件導向語言的問題是，它們帶著語言執行環境的所有隱含資訊四處亂竄。你想要的是香蕉，但看到的卻是香蕉拿在大猩猩手裡，而大猩猩的後面是整個叢林”

公平點說，我的看法是這不是物件導向程式設計的本質問題。你可以寫出函數式的物件導向程式，但確定的是：

* 典型的 OOP 傾向強調改變物件的狀態。
* 典型的 OOP 傾向層與層之間緊密的耦合。
* 典型的 OOP 將同一性(identity)與狀態的概念搞混了。
* 資料與程式碼的混合物，導致了概念與實際的問題產生。

[Rich Hickey][RH]，Clojure 的發明人（一個給 JVM 用的函數式 Lisp 方言），在這場[出色的演講](http://www.infoq.com/presentations/Value-Identity-State-Rich-Hickey)裡談論了狀態、數值以及同一性。

### 萬物皆表達式

可以這麼寫：

```Ruby
if found_dog == our_dog
  name = found_dog.name
  message = "We found our dog #{name}!"
else
  message = "No luck"
end
```

然而，控制結構（`if`, `while`, `case` 等）也回傳表達式，所以只要這樣寫就好：

```Ruby
message = if found_dog == my_dog
  name = found_dog.name
  "We found our dog #{name}!"
else
  "No luck"
end
```

這樣子我們不用重複變數名 `message`，企圖也更明顯：當有段長的程式（用了一堆我們不在乎的變數），我們可以專注在程式在幹什麼（回傳訊息）。再強調一次，我們在縮小程式的作用域。

另一個函數式程式的好處是，表達式可以用來建構資料：

```Ruby
{
  :name => "M.Cassatt",
  :paintings => paintings.select { |p| p.author == "M.Cassatt" },
  :birth => painters.detect { |p| p.name == "M.Cassatt" }.birth.year,
  ...
}
```

### 遞迴

純函數式語言沒有隱含的狀態，大量利用了遞迴。為了避免 stack overflow，函數式使用一種稱為尾遞迴優化（TCO）的機制。Ruby 1.9 有實作這種機制，但預設沒有打開。要是你希望你的程式，在哪都可以動的話，就不要使用它。

但是某些情況下，遞迴仍然是很有用的，即便是每次遞迴時都創建新的堆疊。注意！某些遞迴的用途可以用 foldings 來實現（像 `Enumerable#inject`）。

在 MRI-1.9 啟用 TCO：

```Ruby
RubyVM::InstructionSequence.compile_option = {
  :tailcall_optimization => true,
  :trace_instruction => false,
}
```

簡單範例：

```Ruby
module Math
  def self.factorial_tco(n, acc=1)
    n < 1 ? acc : factorial_tco(n-1, n*acc)
  end
end
```

在遞迴深度不太可能很深的情況下，你仍可以使用：

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

### 惰性枚舉器

惰性求值延遲了表達式的求值，在真正需要時才會求值。與 eager evaluation 相反，eager evaluation 當一個變數被賦值時、函數被呼叫時…甚至根本沒用到變數等狀況，都馬上對表達式求值，惰性不是函數式程式設計的必需品，但這是個符合函數式範式的好策略（Haskell 大概是最佳的例子，瀰漫著懶惰的語言）。

Ruby 所採用的基本上是 eager evaluation（雖然許多其它的語言，在條件還沒滿足前不對表達式求值，以及短路布林運算 `&&`, `||` 等）。然而，與任何內建高階函數的語言一樣，延遲求值是隱性支援，因為程式設計師自己決定區塊何時被呼叫。

Enumerators 同樣 從 Ruby 1.9 開始支援(1.8 請用 backports)，它們提供了一個簡單的介面來定義惰性 enumerables。經典的例子是建構一個枚舉器，回傳所有的自然數：

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

可以用更函數式的精神改寫：

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

現在，試試給 `natural_numbers` 做 `map`，發生什麼事？它不會停止。標準的 enumerable 方法 (`map`, `select` 等）回傳一個陣列，所以在輸入流是無窮大時，無法正常工作。讓我們擴展 Enumerator 類別，比如加入這個惰性的 `Enumerator#map`：


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

現在我們可以給所有自然數的流做 `map` 了：

```Ruby
natural_numbers.map { |x| 2*x }.take(10)
# [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
```

枚舉器是用來建構惰性行為的區塊的好東西，但你可以使用用懶惰風格，實作了所有 Enumerable 方法的函式庫：


https://github.com/yhara/enumerable-lazy

```Ruby
require 'enumerable/lazy'
(1..1.0/0).lazy.map { |x| 2*x }.take(10).to_a
# [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
```

#### 惰性求值的好處

1. 顯而易見的好處: 無需在不必要的情況下，建構、儲存完整的結構（也許，可以更有效率的使用 CPU 及記憶體）

2. 不太顯而易見的好處: 惰性求值使寫程式不需要了解超出你所需的範圍。讓我們看一個例子：你寫了某種解題工具，可以提供無數種解法，但在某個時候，你只想要前十種解法。你可能會這麼寫：

```Ruby
solver(input, :max => 10)
```

當你與惰性結構一起工作時，不需要說什麼時候該結束。呼叫者自己會決定他需要多少值。程式碼變得更簡單，責任歸屬到對的地方，也就是呼叫者：

```Ruby
solver(input).take(10)
```

### 一個實際的範例

練習：“前十個平方可被五整除的自然數的和是多少？”

```Ruby
Integer::natural.select { |x| x**2 % 5 == 0 }.take(10).inject(:+) #=> 275
```

讓我們跟等價的命令式版本來比較：

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

我希望這個例子展示了這個文件裡討論的函數式程式設計的優點：

1. 更簡潔: 你會撰寫更少的程式碼。函數式程式處理的是表達式，而表達式可以連鎖起來；命令式程式處理的是變量的改動（敘述式），而這不能連鎖。

2. 更抽象: 你可以爭論我們使用 `select`, `inject`…等等，來隱藏了一大堆程式碼，我很高興你這麼說，因為我們正是這麼幹的。將通用的、可重用的程式碼隱藏起來，這是所有程式設計的重點 ── 但函數式程式設計特別是關於如何撰寫抽象。感到開心不是因為寫了更少的代碼，而是因為藉由認出可重用的模式，簡化了程式碼的複雜性。

3. 更有聲明式的味道: 看看命令式的版本，第一眼看起來是一沱無用的程式碼 ── 沒有註解的話 ── 它會做什麼你完全沒有概念。你可能會說：“好吧，從這裡開始讀，草草記下 `n` 與 `sum` 的值，進入某個迴圈，看看 `n` 與 `sum` 的值如何變化，看看最後一次迭代的情形” 等等。函數式版本另一方面是自我解釋的，函數式版本描述、聲明它在幹的事，而不是如何幹這件事。

“函數式程式設計就像是將你的問題敘述給數學家一樣。命令式程式設計像是給白痴下指令” (arcus 在 Freenode #scheme 頻道所說）

### 結論

更好的理解函數式程式設計的原理，幫助我們寫出更清晰、重用性更高並更簡潔的程式碼。Ruby 基本上是一個命令式語言，但它也有很大的函數式能力，明白什麼時候用，及如何用（以及何時不該用）這些能力。將這句話當成你的座右銘吧 “狀態是萬惡的根源，盡可能避免它。”

### 簡報

Workshop at [Conferencia Rails 2011](http://conferenciarails.org/): [Functional Programming with Ruby](http://public.arnau-sanchez.com/ruby-functional/) [(slideshare)](http://www.slideshare.net/tokland/functional-programming-with-ruby-9975242)

### 延伸閱讀

http://en.wikipedia.org/wiki/Functional_programming

http://www.defmacro.org/ramblings/fp.html __[譯文](http://t.cn/zYaCDw7)__

http://www.cse.chalmers.se/~rjmh/Papers/whyfp.html

http://www.khelll.com/blog/ruby/ruby-and-functional-programming/

http://www.bestechvideos.com/2008/11/30/rubyconf-2008-better-ruby-through-functional-programming

http://channel9.msdn.com/Blogs/pdc2008/TL11

http://www.infoq.com/presentations/Value-Identity-State-Rich-Hickey

## 授權

This document is licensed under the CC-By 3.0 License, which encourages you to share these documents. See http://creativecommons.org/licenses/by/3.0/ for more details.

<img alt="CC-By 3.0 License http://creativecommons.org/licenses/by/3.0/" style="border-width:0" src="http://i.creativecommons.org/l/by/3.0/88x31.png" />

[RH]: https://twitter.com/richhickey
[JA]: http://www.sics.se/~joe/
