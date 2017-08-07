---
layout: post
title: "反面模式：用迭代循环来构建集合"
date: 2014-03-22 15:26
comments: true
categories: translation
---
原文：[Anti-Pattern: Iteratively Building a Collection](http://robots.thoughtbot.com/iteration-as-an-anti-pattern)

Ruby 自带了很多出色的 Enumerable 方法，然而其中最有用的两个方法是传承自 Smalltalk 和 LISP 的 `#map` 和 `#inject`。一些冗长的方法定义通过使用这两个方法进行改写以后，可以变得既简洁又更加清晰。

## 构建一个数组

需求：

> 每个用户都有一个 PGP 公钥，我想要得到所有用户的公钥以便能够快速地将它们导入密钥服务器。

第一版代码有些冗长且过于死板。

```ruby
def signer_key_ids
  result = []

  signers.each do |signer|
    result << signer.key_id
  end

  result
end
```

但只需简单地使用 `#map` ，就能更清晰地表明该方法的用途。

```ruby
def signer_key_ids
  rsigners.map { |signer| signer.key_id }
end
```

## 由多个数组构建一个数组

另一个需求：

> 每个用户都有一个 PGP 公钥，我想要一份包括了所有用户的所有 UID 的清单，以便于我能看到他们的名字和地点。

我们可以用 `#each` 和 `#flatten` 来结构化地实现。

```ruby
def signer_uids
  result = []

  signers.each do |signer|
    result << signer.uids
  end

  result.flatten
end
```

然而 `#map` 更加清晰。注意这里用到了 `Symbol#to_proc`。

```ruby
def signer_uids
  signers.map(&:uids).flatten
end
```

`#inject` 配合 `Array#+` 就可以不用在末尾调用 `#flatten`：

```ruby
def signer_uids
  signers.inject([]) do |result, signer|
    result + signer.uids
  end
end
```

其实本例中，使用 `#inject` 还不是最直接的方法，我们可以直接使用 `Enumerable#flat_map`：

```ruby
def signer_uids
  signers.flat_map(&:uids)
end
```

## 由一个数组构建一个 hash

又收到了一个需求：

> 每个用户都有一个 PGP 公钥，我想要得到一个散列表来通过每个用户的公钥 id 匹配他的全部 UID，以便于我来构建自己的密钥服务器。

我们需要构建一个 hash，而且我们需要使用数组中的每个元素。至少，下面的代码诠释了这一点：

```ruby
def signer_keys_and_uids
  result = {}

  signers.each do |signer|
    result[signer.key_id] = signer.uids
  end

  result
end
```

其实还有另一种诠释的方法：先给定一个空的 hash，把用户数组中的每个元素的公钥 id 到 UID 的配对注入（`#inject`）到 hash 中：

```ruby
def signer_keys_and_uids
  signers.inject({}) do |result, signer|
    result.merge(signer.key_id => signer.uids)
  end
end
```

## 由一个数组构建一个 Boolean

他们发誓这个最后一个需求：

> 每个用户都有一个 PGP 公钥，我想要确认我的所有用户都是通过我来认证的，以便于我时常能够互相感觉良好。

上面 hash 的例子中我们处理的是另一个 Enumerable。而这里是一个 Boolean，我们先来看一个绕远路的方法：

```ruby
def mutually_signed?
  result = true

  signers.each do |signer|
    result = result && signer.signed_by?(self)
  end

  result
end
```

然后是下面这种方法，有些眼熟：

```ruby
def mutually_signed?
  signers.inject(true) do |result, signer|
    result && signer.signed_by?(self)
  end
end
```

然而这个方法太不犀利了，我们可以把它当成一个所有元素必须全部为 true 的 Boolean 的数组：

```ruby
def mutually_signed?
  signers.map(&:signed_by?).inject(:&)
end
```

身为 Rubyists，我们还应该知道 `Enumerable` 有很多其他出色的抽象方法：

```ruby
def mutually_signed?
  signers.all?(&:signed_by?)
end
```

## 下一步是？

想要更好地培养对 `#map`、`#inject` 和其他 `Enumerble` 方法的感觉，我建议暂时离开 Ruby 一会儿。看看下面这些关于函数式编程的优秀书籍：

- [Learn You a Haskell for Great Good](http://learnyouahaskell.com/)
- [The Little Schemer](http://www.ccs.neu.edu/home/matthias/BTLS/)
- [Smalltalk Best Practice Patterns](http://www.amazon.com/Smalltalk-Best-Practice-Patterns-Kent-ebook/dp/B00BBDLIME/ref=tmm_kin_title_0)

如果你想要阅读更多关于 Ruby 的 `#inject` 的文章，可以参阅：

- [Come Correct with Inject and Collect](http://robots.thoughtbot.com/come-correct-with-inject-and-collect)
- [Derive #inject for a Better Understanding](http://robots.thoughtbot.com/derive-inject-for-a-better-understanding)
