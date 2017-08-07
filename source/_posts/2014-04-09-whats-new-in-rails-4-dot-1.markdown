---
layout: post
title: "Rails 4.1 的新特性"
date: 2014-04-09 20:21
comments: true
categories: translation
---
原文： [What's new in Rails 4.1](http://brewhouse.io/blog/2013/12/17/whats-new-in-rails-4-1.html)

如果你还不知道的话，Rails 4.1 已经[于今天正式发布了](http://weblog.rubyonrails.org/)！虽然只是版本的小更新，但还是新引入了诸多激动人心的新特性。本文列出了其中几个最让我欣喜的新特性，并介绍了为什么我觉得它们很有用。

## Action Mailer 预览

测试 Rails 的邮件模板一直以来都很不方便。目前我的测试流程为：

1. 改动邮件模板
2. 通过 rails console 发送邮件
3. 在浏览器中查看输出内容
4. 重复上述步骤

虽然 [Letter Opener](https://github.com/ryanb/letter_opener) gem 提供了一些便利，但还远远不够理想。幸运的是，[@pixeltrix](https://github.com/pixeltrix) 通过一番努力将 [MailView](https://github.com/37signals/mail_view) gem 集成进了 Rails 4.1。现在你可以轻易地为 mailer 创建 preview 并在浏览器中进行预览了 `http://localhost:3000/rails/mailers`：

```ruby
# In /test/mailers/previews/notifier_preview.rb
class NotifierPreview < ActionMailer::Preview
  def welcome
    # Mock up some data for the preview
    user = FactoryGirl.build(:user)

    # Return a Mail::Message here (but don't deliver it!)
    Notifier.welcome(user)
  end
end
```

需要注意的是，虽然默认情况下 preview 文件位于 test 目录下（可以通过 `config.action_mailer.preview_path` 修改），但其实它运行于 development 环境。因此，如果你要用 `FactoryGirl` 等 gem 来生成数据，需要确保 Gemfile 中这些 gem 也在 development 的 group 下面。

如果你的 app 没有 `test` 目录（比如 `rspec` 用户），你可能会将默认的 `config.action_mailer.preview_path` 修改成类似 `/app/mailer/previews` 的路径。但请注意，`/app` 目录在 production 环境下是即时加载（eager-loaded）的，所以或许这并不是 preview 文件的最佳存放之处。

该特性的延伸阅读：

- [Release notes](http://edgeguides.rubyonrails.org/4_1_release_notes.html#action-mailer-previews)
- [Documentation](http://edgeapi.rubyonrails.org/classes/ActionMailer/Base.html)
- Pull request: [#13332](https://github.com/rails/rails/pull/13332)

## Active Record Enums

你曾经在 model 中使用多个 `boolean` 字段来表示一个复杂的状态吗？我绝对这么干过，并且代码很快就变得无法控制了。

救星 Enums 来了！

```ruby
class Bug < ActiveRecord::Base
  # Relevant schema change looks like this:
  #
  # create_table :bugs do |t|
  #   t.column :status, :integer, default: 0 # defaults to the first value (i.e. :new)
  # end

  enum status: [ :new, :assigned, :in_progress, :resolved, :rejected, :reopened ]

  belongs_to :assignee, class_name: 'Developer'

  def assignee=(developer)
    if developer && self.new?
      self.status = :assigned
    else
      self.status = :new
    end

    super
  end
end

Bug.resolved           # => a scope to find all resolved bugs

bug.resolved?          # => check if bug has the status :resolved

bug.resolved!          # => update! the bug with status set to :resolved

bug.status             # => a symbol describing the bug's status

bug.status = :resolved # => set the bug's status to :resolved
```

在内部，与这些状态在数据库中相对应的是整数值，以节省空间。同样值得一提的是，`enum` 宏所添加的方法是通过 mix-in 一个 module 来实现的。这意味着你可以方便地在 model 中重写它们并使用 `super` 来调用原来的实现。

使用该特性时还有如下一些注意事项：

**I.** 不要被其名字所迷惑，在一些数据库中并不使用 `ENUM` 类型来实现该特性。状态和其对应的整数值的匹配是通过 model 文件来维护的。所以一旦定义完 `enum` 的 symbol 后，你就不应该再去改动其顺序了。要移除不用的状态，可以使用显式的匹配：

```ruby
class Bug < ActiveRecord::Base
  enum status: {
    new: 0,
    in_progress: 2,
    resolved: 3,
    rejected: 4,
    reopened: 5
  }
end
```

**II.** 避免在一个类的不同 enum 中使用相同的名字！这样会使 Active Record 很迷茫！

```ruby
class Bug < ActiveRecord::Base
  enum status: [ :new, ... ]
  enum code_review_status: [ :new, ... ] # WARNING: Don't do this!
end
```

**III.** 如果你要使用自定义的 scope 来查询 enum 字段，需要传入对应的整数而不是 symbol。你可以使用宏所添加的常量来访问 enum-integer 的匹配：

```ruby
class Bug < ActiveRecord::Base
  scope :open, -> {
    where('status <> ? OR status <> ?', STATUS[:resolved], STATUS[:rejected])
  }
end
```

~~**IV.** 目前，dirty tracking 方法（比如 `status_was?`）还无法用于 enum（目前返回的是整数值而不是 symbol 值）。此问题应该在正式版发布前会修复（进度参见 [#13267](https://github.com/rails/rails/pull/13267)）~~

该特性的延伸阅读：

- [Release notes](http://edgeguides.rubyonrails.org/4_1_release_notes.html#active-record-enums)
- [Documentation](http://edgeapi.rubyonrails.org/classes/ActiveRecord/Enum.html)
- Original commit: [db41eb8a](https://github.com/rails/rails/commit/db41eb8a6ea88b854bf5cd11070ea4245e1639c5)

## Action Pack Variants

作为 Web 开发人员，应该已经意识到我们已经全面过渡到了[后PC时代](http://zh.wikipedia.org/wiki/%E5%BE%8C%E5%80%8B%E4%BA%BA%E9%9B%BB%E8%85%A6%E6%99%82%E4%BB%A3)。虽然我钟爱于响应式设计，但它并不是解决跨设备网页显示问题的银弹。多数情况下，更合适的方法是为特定的设备种类定制 view 来显示最恰当的内容。

有了 **Action Pack Variants** 以后，在 Rails 4.1 中实现起来就容易多了：

```ruby
class ApplicationController < ActionController::Base
  before_action :detect_device_variant

  private

    def detect_device_variant
      case request.user_agent
      when /iPad/i
        request.variant = :tablet
      when /iPhone/i
        request.variant = :phone
      end
    end
end

class PostController < ApplicationController
  def show
    @post = Post.find(params[:id])

    respond_to do |format|
      format.json
      format.html               # /app/views/posts/show.html.erb
      format.html.phone         # /app/views/posts/show.html+phone.erb
      format.html.tablet do
        @show_edit_link = false
      end
    end
  end
end
```

上面的示例用了 `before_action` 使 HTTP 的 `User-Agent` 消息头匹配给定的关键字，并相应地赋值给 `request.variant`。在 `respond_to` 的 block 中通过指定所支持的 variant，Rails 会根据特定的 format 和 variant 的组合来 render 相应的模板。你还可以传一个 block 来指定某个 variant 的情况下要执行的代码。

实际上你甚至可以将声明省略——只要在 `views` 目录下有对应的模板文件，Rails 就会自动匹配上。而如果某个 variant 没有专门的模板，Rails 会退一步去加载该 format 默认的模板（比如 `show.html.erb`）。因此你可以在两个 variant 之间共享模板。在上面的示例中，如果不存在 `/app/views/posts/show.html+tablet.erb`， `tablet` variant 就会重用默认模板。

虽然在介绍该特性时，大多数示例都会使用 `User-Agent` 消息头，但值得注意的是 Rails 中的实现完全是不可预知的。在渲染模板之前，`request.variant` 可能是基于任何条件被赋值的，例如基于请求的域名（或子域名）、HTTP 消息头、session 数据、甚至是抛硬币的结果。

这就使得该特性具有相当的灵活性，可以用于很多用途，例如 API 版本控制、A/B 测试、甚至可以用来做特性演示！

该特性的延伸阅读：

- [Release notes](http://edgeguides.rubyonrails.org/4_1_release_notes.html#variants)
- [Documentation](http://edgeapi.rubyonrails.org/classes/ActionController/MimeResponds.html#method-i-respond_to)
- Pull request: [#12977](https://github.com/rails/rails/pull/12977)、[#13290](https://github.com/rails/rails/pull/13290)

## Application Message Verifier

Rails 4.1 还引入了一个内置的 helper 来生成基于 [HMAC](http://en.wikipedia.org/wiki/Hash-based_message_authentication_code) 的加密消息。消息验证以前被用于加密 Cookies 等高端操作，然而现在已经可以方便地将其用于其他一些用途了。

例如，你可以实现一个无状态的「重置密码」功能而无须在数据库中存储 token：

```ruby
class User < ActiveRecord::Base
  class << self
    def verifier_for(purpose)
      @verifiers ||= {}
      @verifiers.fetch(purpose) do |p|
        @verifiers[p] = Rails.application.message_verifier("#{self.name}-#{p.to_s}")
      end
    end
  end

  def reset_password_token
    verifier = self.class.verifier_for('reset-password') # Unique for each type of messages
    verifier.generate([id, Time.now])
  end

  def reset_password!(token, new_password, new_password_confirmation)
    # This raises an exception if the message is modified
    user_id, timestamp = self.class.verifier_for('reset-password').verify(token)

    if timestamp > 1.day.ago
      self.password = new_password
      self.password_confirmation = new_password_confirmation
      save!
    else
      # Token 过期
      # ...
    end
  end
end

class Notifier < ActionMailer::Base
  def reset_password(user)
    @user = user
    @reset_password_url = password_reset_url(token: @user.reset_password_token)
    mail(to: user.email, subject: "Your have requested to reset your password")
  end
end
```

这样一来，重置密码所需的所有信息都包含在链接中了，没必要在数据库中存储信息。这也能用于 `OAuth` 等（`state` 参数）。

使用该特性时，需要重点防范[重放攻击](http://en.wikipedia.org/wiki/Replay_attack)。以上面的代码为例，如果我们没有使用 timestamp 来检验是否过期，万一邮件落到了居心不良的人手中，这个 URL 就可以随时用来重置用户的密码！

另外，用来加密消息的 key 基于应用程序的 `secret_key_base` 以及你加入的「盐值（salt）」（本例中为 `User-reset-password`）。改变其中一个，就会使得之前的加密消息全部无效。

## Spring

根据你使用的 gems，启动一个 Rails app 平均需要5秒钟时间。意味着每次运行测试都会浪费5秒钟，哪怕你只是运行一个单独的测试用例！如果你遵循 TDD，你每天大约会运行50次测试。这么算的话，过去5年中就浪费了[整整5天时间](http://xkcd.com/1205/)！

我们终于时来运转，由 Rails 4.1 生成的新应用在内部集成了预加载器 [Spring](https://github.com/jonleighton/spring)。

Spring 使你的应用程序运行于后台，所以你就不必每次运行测试、rake task 或 migration 时都要启动应用程序了。如果你熟悉 [Zeus](https://github.com/burke/zeus) 或 [Spork](https://github.com/sporkrb/spork) 等 gem，就不会对此感到陌生。然而 spring 使用 binstub 包装了常用的 Rails 命令（默认为 `rake` 和 `rails`），因此如果 `./bin` 在你的 `PATH` 中，那么什么都不用做就能提速不少了！

我在自己的项目中试了一下，使用了 spring 后每次运行测试都能节省近5秒钟。我能因此得到5天假期吗？:)

你可以阅读 [Spring README](https://github.com/jonleighton/spring#readme) 来了解它的工作原理，以及如何在既有的项目中引入它。

该特性的延伸阅读：

- [Release notes](http://edgeguides.rubyonrails.org/4_1_release_notes.html#spring-application-preloader)
- [Upgrading guide](http://edgeguides.rubyonrails.org/upgrading_ruby_on_rails.html#spring)
- [Documentation](https://github.com/jonleighton/spring#readme)
- Pull request: [#12958](https://github.com/rails/rails/pull/12958)

## 其他新特性

这些都只是本次更新内容的一部分。还有很多其他可能对你有用的新特性，比如 [使用 secrets.yml 来记录敏感数据](http://edgeguides.rubyonrails.org/4_1_release_notes.html#config-secrets-yml)、[在测试中控制时间](https://github.com/rails/rails/pull/12824)、[更好的 JSON 处理](https://github.com/rails/rails/pull/12183)、[Module#concerning](http://edgeguides.rubyonrails.org/4_1_release_notes.html#module-concerning)、[to_param 宏](https://github.com/rails/rails/pull/12891)等等。我建议你通过[更新日志](http://edgeguides.rubyonrails.org/4_1_release_notes.html)来查看所有的变更点！
