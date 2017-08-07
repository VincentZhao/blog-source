---
layout: post
title: "Ruby 实现的简易推荐系统"
date: 2014-03-25 21:05
comments: true
categories: translation
---
原文：[Simple recommendation system written in Ruby](http://otobrglez.opalab.com/ruby/2014/03/23/simple-ruby-recommendation-system.html)

我在求职，于是昨天我回顾了一下以前做过的 Rails 项目，试图能给我的简历添上几笔。我找到了一个有意思的老项目，我在其中实现了推荐系统。但也没有很出彩，只是基于博客文章的标签做了推荐。我决定拿出其中的一些代码来写一篇文章。

推荐系统的算法基于 [Jaccard 系数](http://en.wikipedia.org/wiki/Jaccard_index)，也被称为 Jaccard 相似度系数。Jaccard 系数是由植物学家 [Paul Jaccard](http://en.wikipedia.org/wiki/Paul_Jaccard) 提出的，是用来体现样本相似性和差异性的数值。

## 运作原理

取出当前的 item（比如博客文章）和能够很好地描述它的属性（标签、分类或单词）。然后对其它每个 item 计算出它们交集和并集的商。数学过程可以表述为以下公式。

![](http://upload.wikimedia.org/math/1/8/6/186c7f4e83da32e889d606140fae25a0.png)

该方程的计算结果在 0 到 1 之间。你也能够方便地用以下公式来计算项目间的差异性。

![](http://upload.wikimedia.org/math/0/2/9/02906c47e0a08707ad6e35a6c34a43b4.png)

## Ruby 的实现示例

下面我将使用书名中的单词来推荐书籍。使用简单的 `Book` 类即可。

```ruby
class Book < Struct.new(:title)

  # 长度大于2的不重复的单词的数组
  # 也可以是「标签」或「分类」的数组
  def words
    @words ||= self.title.gsub(/[a-zA-Z]{3,}/).map(&:downcase).uniq.sort
  end

end
```

`BookRecommender` 类使用当前的书籍和书籍数组进行初始化。`recommendations` 方法会循环数组并给每一个元素设定 `jaccard_index` 值，最后将书籍进行排序。

```ruby
class BookRecommender

  def initialize book, books
    @book, @books = book, books
  end

  def recommendations

    # 计算每个元素的 jaccard_index 值并排序
    @books.map! do |this_book|

      # 运行中定义 jaccard_index 的取值 singleton 方法
      this_book.define_singleton_method(:jaccard_index) do
        @jaccard_index
      end

      # 还有赋值方法
      this_book.define_singleton_method("jaccard_index=") do |index|
        @jaccard_index = index || 0.0
      end

      # 计算样本的交集
      intersection = (@book.words & this_book.words).size
      # ... 和并集
      union = (@book.words | this_book.words).size

      # 将除法运算的结果赋值，如无法计算则捕捉异常并赋值为0
      this_book.jaccard_index = (intersection.to_f / union.to_f) rescue 0.0

      this_book

      # 排序
    end.sort_by { |book| 1 - book.jaccard_index }

  end

end
```

演示推荐过程：

```ruby
# ...
# 读取数据并定义书籍数组
BOOKS = DATA.read.split("\n").map { |l| Book.new(l) }

# 定义当前书籍
current_book = Book.new("Ruby programming language")

# 进行推荐...
books = BookRecommender.new(current_book, BOOKS).recommendations

books.each do |book|
  puts "#{book.title} (#{'%.2f' % book.jaccard_index})"
end

__END__
Finding the best language for the job
Could Ruby save the day
Python will rock your world
Is Ruby better than Python
Programming in Ruby is fun
Python to the moon
Programming languages of the future
```

下面是名为「Ruby programming language」的书籍的输出结果，右边的数值就是 Jaccard 指数。

```plain
Programming in Ruby is fun (0.50)
Programming languages of the future (0.17)
Is Ruby better than Python (0.17)
Could Ruby save the day (0.14)
Finding the best language for the job (0.12)
Python to the moon (0.00)
Python will rock your world (0.00)
```

## 总结

以上就是纯粹基于 Ruby 的解决方案，[源代码在我的 gist 上](https://gist.github.com/otobrglez/9738998)。我还写了使用标签的 [PostgresSQL 版本](https://gist.github.com/otobrglez/1078953)。但要注意的是，当样本变得很大以后代码的执行速度也会变慢，因此不要每次操作 item 时都运行推荐，更好的方法是定义后台服务在后台进行推荐的计算。

希望你能够在自己的项目中用到这个方法，也请给我反馈。谢谢！：）
