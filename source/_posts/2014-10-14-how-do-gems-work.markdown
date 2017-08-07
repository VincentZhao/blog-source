---
layout: post
title: "Ruby Gems 是如何运作的？"
date: 2014-10-14 20:01
comments: true
categories: translation
---
原文: [How Do Gems Work?](http://www.justinweiss.com/blog/2014/09/29/how-do-gems-work/)

Ruby gems 在通常情况下能够运转自如，但 Ruby 的魔法背后始终隐藏着一个不小的问题：发生错误后很难找出原因所在。

Gems 一般不会出现问题，然而一旦发生了，搜索引擎却常常会显得无能为力。笼统的错误消息根本无法帮助我们定位问题的出处。如果你不明白 Ruby 中的 gems 是如何运作的，独自调试这些问题对你来说无疑会是一段痛苦的经历。

Gems 看上去很魔幻，但只要经过一小番探索，你会发现其实并非那么难以理解。

## gem install 做了什么？

一个 Ruby gem 只是一些被打包的代码，再加上一些额外的数据。通过 `gem unpack` 命令，你可以看到 gem 内部的代码：

```
~/Source/playground jweiss$ gem unpack resque_unit
Fetching: resque_unit-0.4.8.gem (100%)
Unpacked gem: '/Users/jweiss/Source/playground/resque_unit-0.4.8'
~/Source/playground jweiss$ cd resque_unit-0.4.8
~/Source/playground/resque_unit-0.4.8 jweiss$ find .
.
./lib
./lib/resque_unit
./lib/resque_unit/assertions.rb
./lib/resque_unit/errors.rb
./lib/resque_unit/helpers.rb
./lib/resque_unit/plugin.rb
./lib/resque_unit/resque.rb
./lib/resque_unit/scheduler.rb
./lib/resque_unit/scheduler_assertions.rb
./lib/resque_unit.rb
./lib/resque_unit_scheduler.rb
./README.md
./test
./test/resque_test.rb
./test/resque_unit_scheduler_test.rb
./test/resque_unit_test.rb
./test/sample_jobs.rb
./test/test_helper.rb
~/Source/playground/resque_unit-0.4.8 jweiss$
```

`gem install` 命令最简单的行为如下，它获取 gem 后将其文件存储到你系统上一个特别的目录下。你可以运行 `gem environment` 命令来查看 `gem install` 命令把 gems 安装到哪里（找到 `INSTALLATION DIRECTORY:` 行）：

```
~ jweiss$ gem environment
RubyGems Environment:
  - RUBYGEMS VERSION: 2.2.2
  - RUBY VERSION: 2.1.2 (2014-05-08 patchlevel 95) [x86_64-darwin14.0]
  - INSTALLATION DIRECTORY: /usr/local/Cellar/ruby/2.1.2/lib/ruby/gems/2.1.0
  ...
```

```
~ jweiss$ ls /usr/local/Cellar/ruby/2.1.2/lib/ruby/gems/2.1.0
bin           bundler     doc         gems
build_info    cache       extensions  specifications
```

所有已安装的 gem 代码都在那里，在 `gems` 目录下面。

上述路径随着操作系统的不同而不同，同样也取决于 Ruby 的安装方式（rvm 与 Homebrew 不同，与 rbenv 也不同，等等）。所以如果你想知道 gems 的代码在哪里，`gem environment` 命令会是一个好帮手。

## gem 代码怎样被加载？

为了让 gems 中的代码被使用到，RubyGems 覆写了 Ruby 的 `require` 方法。具体的代码在 [core_ext/kernel_require.rb](https://github.com/rubygems/rubygems/blob/169b12eb5815784d7ee721186456ad20a004ce71/lib/rubygems/core_ext/kernel_require.rb#L38)。注释写的很清楚：

```ruby
##
# When RubyGems is required, Kernel#require is replaced with our own which
# is capable of loading gems on demand.
#
# When you call <tt>require 'x'</tt>, this is what happens:
# * If the file can be loaded from the existing Ruby loadpath, it
#   is.
# * Otherwise, installed gems are searched for a file that matches.
#   If it's found in gem 'y', that gem is activated (added to the
#   loadpath).
#
```

比方说你要加载 `active_support`，RubyGems 会尝试使用 Ruby 的 `require` 方法来加载，这时会得到如下错误：

```
LoadError: cannot load such file -- active_support
  from (irb):17:in `require'
  from (irb):17
  from /usr/local/bin/irb:11:in `<main>'
```

RubyGems [看到了这个错误消息](https://github.com/rubygems/rubygems/blob/169b12eb5815784d7ee721186456ad20a004ce71/lib/rubygems/core_ext/kernel_require.rb#L125)，知道了需要去寻找某个 gem 中的 `active_support.rb`。然后它[扫描所有 gems 的 metadata](https://github.com/rubygems/rubygems/blob/169b12eb5815784d7ee721186456ad20a004ce71/lib/rubygems/specification.rb#L927)，寻找其中包含 `active_support.rb` 的 gem：

```
irb(main):001:0> spec = Gem::Specification.find_by_path('active_support')
=> #<Gem::Specification:0x3fe366874324 activesupport-4.2.0.beta1>
```

找到以后，RubyGems 将其激活，并把 gem 中的代码添加到了 Ruby 的加载路径（load path，也就是你可以 `require` 其他文件的地方）中：

```
irb(main):002:0> $LOAD_PATH
=> ["/usr/local/Cellar/ruby/2.1.2/lib/ruby/site_ruby/2.1.0", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/site_ruby/2.1.0/x86_64-darwin14.0", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/site_ruby", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/vendor_ruby/2.1.0", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/vendor_ruby/2.1.0/x86_64-darwin14.0", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/vendor_ruby", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/2.1.0", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/2.1.0/x86_64-darwin14.0"]
irb(main):003:0> spec.activate
=> true
irb(main):004:0> $LOAD_PATH
=> ["/usr/local/Cellar/ruby/2.1.2/lib/ruby/gems/2.1.0/gems/i18n-0.7.0.beta1/lib", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/gems/2.1.0/gems/thread_safe-0.3.4/lib", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/gems/2.1.0/gems/activesupport-4.2.0.beta1/lib", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/site_ruby/2.1.0", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/site_ruby/2.1.0/x86_64-darwin14.0", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/site_ruby", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/vendor_ruby/2.1.0", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/vendor_ruby/2.1.0/x86_64-darwin14.0", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/vendor_ruby", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/2.1.0", "/usr/local/Cellar/ruby/2.1.2/lib/ruby/2.1.0/x86_64-darwin14.0"]
```

既然 `active_support` 已经被加到了加载路径中，你就可以像往常一样 `require` gem 中的文件了。你甚至可以使用原始的 `require` 方法，也就是被 RubyGems 所覆写的哪个方法：

```
irb(main):005:0> gem_original_require 'active_support'
=> true
```

酷！

## 小知识，大用处

RubyGems 看上去很复杂，但它最基础的用途只是替你管理 Ruby 的加载路径。但也不是说 RubyGems 都很简单，这里没有涉及 RubyGems 如何管理 gems 的版本冲突、gem 二进制文件（比如 `rails` 和 `rake`）、C 语言扩展等其他内容。

然而，即使只了解 RubyGems 的基础内容也会对你产生很大的帮助。经过一番简单的源码阅读和 `irb` 中的实践后，你就知道了怎样去深入探索 gems 代码。你知道了你的 gems 都在哪里，所以能确保 RubyGems 也能找到它们。一旦你明白了 gems 的加载过程，你也就有能力去解决一些看上去很棘手的加载方面的问题了。
