---
layout: post
title: "RSpec - 谨慎使用 before(:all)"
date: 2014-03-07 19:50
comments: true
categories: weblog
---
今天遇到了类似如下结构的 RSpec 测试代码，运行后出现了「插入重复数据」的数据库错误。

```ruby
require 'spec_helper'

describe User do

  describe "#1" do
    before(:all) do
      create(:user, id: 1)
    end

    # Test
    # :
  end

  describe "#2" do
    before(:all) do
      create(:user, id: 1)
    end

    # Test
    # :
  end

end
```

虽然使用了 database_cleaner 这个 gem 来进行清空数据库的操作，但显然在上面的代码中，当一个 describe 执行结束后，由 `before(:all)` 创建的 `user` 数据并没有被清除。

在进行一番简单的调查后，在 [RSpec 的官方文档](https://www.relishapp.com/rspec/rspec-rails/docs/transactions)中找到了 「在 `before(:all)` 中创建的数据不会被 Rollback」这个解释。

原来如此。可是我没有使用 RSpec 提供的事务回滚，而是使用 database_cleaner 来清除数据的啊，而且在 `spec_helper.rb` 中也已经做了如下配置。

```ruby
RSpec.configure do |config|
  # :
  # :

  config.after(:all) do
    DatabaseCleaner.clean
  end
end
```

又是一番调查。在[这篇文章](http://toctan.com/articles/be-careful-with-before\(:all\)-in-rspec/)的最后给出了一个方法，即在每个 `before(:all)` 的地方都显式地使用 `after(:all)` 来清除数据。试了一下，果然测试可以通过了。

那么上面那段写在 RSpec 配置中的 `config.after(:all)` 究竟在什么时候被执行呢？一开始我想当然地认为它会对应每一个 `before(:all)`，只要有 `before(:all)` 的地方在执行结束后就会调用这个 block。实际测试后才发现，只有在最外层的 Example Group 执行完毕时（比如上文的 `describe User`）才会调用这个定义在 `RSpec.configure` 中的 `config.after(:all)`。

鉴于 `before(:all)` 的陷阱很多，也有人曾经[提出过将其废除](https://github.com/rspec/rspec-core/issues/573)。虽然没有被采纳，但是不到迫不得已，还是不建议使用 `before(:all)`。
