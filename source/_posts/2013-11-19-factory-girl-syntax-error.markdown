---
layout: post
title: "解决使用FactoryGirl时找不到方法的错误"
date: 2013-11-19 22:13
comments: true
categories: weblog
---

今天在新建的工程里跑第一个测试。代码的简略版如下。

```ruby
require 'spec_helper'

describe User do
  describe "#can_login?" do
    context "admin" do
      user = build(:admin_user)
      it { expect(user.can_login?).to be_true }
    end
  end
end
```

但是执行后总是报下列错误，说找不到 `build` 方法。

> undefined method `build' for #<Class:0x35e0470> (NoMethodError)

<!-- more -->

`build` 是 FactoryGirl 的方法，而将 `build` 改成 `FactoryGirl.build` 后测试能通过。

于是很自然的想到了可能是 FactoryGirl 的配置有问题。然而在 `spec_helper.rb` 中已经做了如下配置，意思是 include 了 FactoryGirl 的 syntax，这样就不用每次都写 `FactoryGirl.xxx` 了。

    config.include FactoryGirl::Syntax::Methods

于是接下来花了大约一个小时排查各种原因，这个工程在 `bundle install` 时安装了 rspec 的最新版，所以去 github 上的各个相关项目的 issue 中查看了有没有相关信息。没什么收获后又排查了 JRuby、MRI、Spork 等方面的原因，都无功而返。

最后终于解决了。原因在于 rspec 的代码写的不符合规范。FactoryGirl 的syntax 是不能在 `context` 的 block 中使用的，将它移到 `it` 中就好了。而如果要在 `context` 下使用的话，必须要在 `before` block 中。

```ruby
require 'spec_helper'

describe User do
  describe "#can_login?" do
    context "admin" do
      it "can login" do
        user = build(:admin_user)
        expect(user.can_login?).to be_true
      end
    end
  end
end
```
