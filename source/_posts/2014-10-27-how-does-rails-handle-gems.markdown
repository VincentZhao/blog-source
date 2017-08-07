---
layout: post
title: "Rails 是如何处理 Gems 的？"
date: 2014-10-27 20:01
comments: true
categories: translation
---
原文: [How Does Rails Handle Gems?](http://www.justinweiss.com/blog/2014/10/13/how-does-rails-handle-gems/)

之前我写过[一篇文章来介绍 RubyGems 如何管理 Ruby 的加载路径](http://zhaowen.me/blog/2014/10/14/how-do-gems-work/)。然而 Rails 并没有直接使用 RubyGems，而是使用 [Bundler](http://bundler.io/) 来管理 gems。

如果不明白 Bundler 的运行机制，那么 Rails 自动将 gems 加载到应用中的举动似乎会显得过于神奇。只需要在 `Gemfile` 中添加一行代码，竟然就能够在应用中使用 gem 中的代码了？Bundler、Rails 和 RubyGems 是怎样通力合作，将处理 gems 的依赖关系的过程变得轻松的呢？

## 为什么是 Bundler？

我把 Bundler 想象成一位严格的 gem 管理员。也就是说，Bundler 帮你安装你所需要的正确版本的 gems，并强制你的应用只使用你指定的版本。

这太有用了，要知道为什么，我们必须要回溯到没有 Bundler 的日子里是什么样的光景。

在 Bundler 出现以前，要安装指定版本的 gems 还是相当容易的，可以使用一些配置脚本：

```
gem install rails -v 4.1.0
gem install rake -v 10.3.2
...
```

（但是要确保以上脚本中 Rails 4.1 的依赖关系不与 Rake 10.3.2 冲突！）

可是如果你正在开发多个 Rails 应用，并且每个应用都使用不同版本的 gems，会发生什么呢？除非你特别细心，否则你肯定会遭遇到令人闻风丧胆的激活 gem 异常：

```
Gem::Exception: can't activate hpricot (= 0.6.161, runtime),
already activated hpricot-0.8.3
```

呃，这条错误消息映入眼帘时，脑海中还是会浮现从前那些不堪回首的日子。这个错误的出现通常意味着你将花上一整天的时间来安装和卸载 gems，来确保机器上 gems 都安装了正确的版本。一个不经意的 `gem install rake` 就可能会完全搞乱你精心准备的计划。

[rvm gemsets](https://rvm.io/gemsets/basics) 的出现暂时减缓了这个问题。然而 gemsets 配置起来很耗时，而且一旦你不小心安装到了错误的 gemset 中以后，还是会遇到和以前相同的问题。而有了 Bundler 之后，你几乎不用去考虑 gems 之间的依赖关系，你的应用通常情况下就会正常工作。而且 Bundler 比起 gemsets 要容易配置的多。

总的来说，Bundler 帮你做了两件重要的事。它安装所有你需要的 gems 并且锁定了 RubyGems，使得你在 Rails 应用内部只能加载那些指定的 gems。

## Rails 如何使用 Bundler

Bundle 最关键的作用是安装并隔离你的 gems，但却不止于此。`Gemfile` 中的 gems 的代码是如何被加载到 Rails 应用里面的呢？

如果你看过 `bin/rails` 文件：

```ruby
#!/usr/bin/env ruby
begin
  load File.expand_path("../spring", __FILE__)
rescue LoadError
end
APP_PATH = File.expand_path('../../config/application',  __FILE__)
require_relative '../config/boot'
require 'rails/commands'
```

能看到它通过 require `../config/boot` 来启动 Rails。来看看这个文件吧：

```ruby
ENV['BUNDLE_GEMFILE'] ||= File.expand_path('../../Gemfile', __FILE__)

require 'bundler/setup' # Set up gems listed in the Gemfile.
```

嘿，看到 Bundler 了！（这里我还学到了一招，你可以通过设定环境变量 `BUNDLE_GEMFILE` 来指定不同的 `Gemfile`，这很酷。）

`bundler/setup` 做了几件事：

- 它移除了 `$LOAD_PATH` 中所有 gems 的路径（相当于将 RubyGems 所做的工作都撤销了）
- 然后，它仅将 `Gemfile.lock` 中出现的 gems 加到 `$LOAD_PATH` 中

这样一来，你能 `require` 的 gems 就仅限于 `Gemfile` 中的那些 gems 了。

到了这里，所有你需要的 gems 都在加载路径中了。但是当你使用 RubyGems 的时候，按理说你仍然需要 `require` 你所需要的文件。使用 Rails 的时候我们为什么不用 `require` 就能使用呢？

快速来看一下 `config/application.rb`，这个文件在 Rails 启动后就会被执行：

```ruby
# Require the gems listed in Gemfile, including any gems
# you've limited to :test, :development, or :production.
Bundler.require(*Rails.groups)
```

又看到 Bundler 了！`Bundler.require` 会加载传递给它的 group 中所有的 gems。（group 是指[你在 Gemfile 中指定的 group](http://bundler.io/v1.7/groups.html)）

那么 `Rails.groups` 包含了哪些 group 呢？

```ruby
# Returns all rails groups for loading based on:
#
# * The Rails environment;
# * The environment variable RAILS_GROUPS;
# * The optional envs given as argument and the hash with group dependencies;
#
#   groups assets: [:development, :test]
#
#   # Returns
#   # => [:default, :development, :assets] for Rails.env == "development"
#   # => [:default, :production]           for Rails.env == "production"
def groups(*groups)
 hash = groups.extract_options!
 env = Rails.env
 groups.unshift(:default, env)
 groups.concat ENV["RAILS_GROUPS"].to_s.split(",")
 groups.concat hash.map { |k, v| k if v.map(&:to_s).include?(env) }
 groups.compact!
 groups.uniq!
 groups
end
```

上述代码回答了这个问题。以 development 模式启动 Rails 时，`Rails.groups` 的值为 `[:default, :development]`，而以 production 模式启动 Rails 时，`Rails.groups` 的值为 `[:default, :production]`，等等。

所以，Bundler 会去 `Gemfile` 中查找属于指定 group 的 gems，并且对每个找到的 gem 执行 `require`。如果你写了 `nokogiri` 这个 gem，它就会替你执行 `require "nokogiri"`。这就解释了为什么你无需写任何多余的代码，就能使你的 gems 在 Rails 中正常工作。

## 了解手中的工具

当你能够很好的了解你手头的工具以后，你就会更加好的使用它们。因此，如果你长时间的使用某样东西，最好花一些时间来对它进行一番深入的研究。

如果你使用 Ruby 和 Rails，那你每天都会和 gems 打交道。抽些时间来好好学习它们吧！
