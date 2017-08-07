---
layout: post
title: "Ruby 风格元素之 Length vs Size vs Count"
date: 2014-03-31 21:58
comments: true
categories: translation
---
原文：[The Elements of Style in Ruby #13: Length vs Size vs Count](http://batsov.com/articles/2014/02/17/the-elements-of-style-in-ruby-number-13-length-vs-size-vs-count/)

Ruby 初学者会头疼的问题之一就是，相同的操作往往会有多种实现方式。例如，要获取 `Enumerable` 对象（即使用 `Enumerable` mixin 的类的实例，通常为 `Array`、`Hash`、`Set` 等集合）中元素的数量，就可以使用 `Enumerable#count`，或者 `length` 方法或其同义词 `size`。

```ruby
arr = [1, 2, 3]

arr.length # => 3
arr.size # => 3
arr.count # => 3

h = { a: 1, b: 2 }

h.length # => 2
h.size # => 2
h.count # => 2

str = 'name'
str.length # => 4
str.size # => 4
# str.count won't work as String does not include Enumerable
```

你该用哪一个呢？让我来帮你作出选择吧。

`length` 不是 `Enumerable` 的方法，而是具体类（如 `String` 或 `Array`）中的方法，且通常其时间复杂度为 `O(1)`（常数），速度飞快，也就是说使用该方法会很不错。

该用 `length` 还是 `size` 取决于个人喜好。以我为例，我对集合（散列表和数组等）使用 `size`，而对字符串使用 `length`，因为我觉得对于散列表或堆栈等对象来说没有长度的概念，而应称其为大小（取决于其包含的元素）。相反，认为一段文本存在长度则相当普遍。总之，你调用的是同一个方法，所以语义上的区别并没有那么重要。

而说到 `Enumerable#count`，就是一头完全不同的野兽了。通常调用该方法时会带上 block，或一个参数来返回 `Enumerable` 中符合条件的元素数量。

```ruby
arr = [1, 1, 2, 3, 5, 6, 8]

arr.count(&:even?) # => 3
arr.count(1) # => 2
```

然而，你的确可以不带任何参数地调用它，它会返回枚举中所有元素数量。

```ruby
arr.count # => 7
```

但是这可能会导致性能上的问题，因为 `count` 方法计算枚举的容量时会进行遍历，因此执行速度相对较慢（特别是处理一些巨大的集合）。一些类（如 `Array`）实现了优化版的 `count` 方法，也就是 `length`，但多数类没有提供。

一句话概括就是，当你能够使用 `length` 或 `size` 解决问题的时候，不要使用 `count`。

如果你是 Rails 程序员，需要注意 `ActiveRecord::Relation` 的 `length`、`size` 和 `count` 方法各自都不相同，但这不在本文的讨论范围。（具体可参见 Sean Griffin 的留言）

*以下译自 Sean Griffin 的留言：*

`ActiveRecord::Relation` 的 `length`、`size` 和 `count` 方法也各不相同。

`#count` 用于执行 SQL 的 count 查询。  
`#length` 会加载所有的记录，并对结果数组调用 `#length` 方法。  
`#size` 也具有相同功能但更为智能，如果已经加载过数据就不会再加载。

总结： 优先使用 `#size`，除非你在调用前确实需要加载一次最新的数据，这时使用 `#length`。不要使用 `#count`，因为 `#size` 会尝试跳过查询操作。
