---
layout: post
title: "如何用少于30行代码实现一个模板引擎"
date: 2015-04-04 19:27
comments: true
categories: translation
---
原文： [How to write a template engine in less than 30 lines of code](http://bits.citrusbyte.com/how-to-write-a-template-library/)

声明：本文基于模板引擎库 [mote](https://github.com/soveran/mote)。其简洁的代码赋予了我灵感，如果你之前没有探索过模板引擎的内部实现，相信这会是一个绝佳的学习范例。

## 前言： 什么是模板？

模板引擎是一种使用模板来生成文本（字符串）的工具，它还有助于将呈现逻辑与应用逻辑分离。

除非你正在某些有悠久历史的软件代码中挣扎（或者你开发的是没有用户界面的软件），否则你极有可能已经接触过模板引擎了。

但是你有没有思考过模板引擎的工作原理呢？你尝试过自己来构建一个吗？如果我们来瞥一眼主流的模板引擎的代码库，你会看到有[几百行(erb)](https://github.com/ruby/ruby/blob/trunk/lib/erb.rb)的代码，甚至[几千行(erubis)](https://github.com/genki/erubis/)。即使是名叫 [slim](https://github.com/slim-template/slim) 的家伙也并没有那么苗条。

所以你或许会觉得处理模板是一个相当复杂的问题，然而下面我会一步步地向你展示其实你可以只用少量代码就构建出一个模板引擎。

那让我们开始吧。

## 定义特性

本文我们将要实现的模板引擎只有两条规则：

1. 以 % 开头的行会被解释为 Ruby 代码。
2. 可以把 Ruby 代码插入到任何行中 {% raw %}{{{% endraw %} ... {% raw %}}}{% endraw %} 符号的中间。比如我们可以使用 `{% raw %}{{{% endraw %} article.title {% raw %}}}{% endraw %}`。

完了？就两条规则？没错，记住，第一条规则让我们可以访问 Ruby 的一切，也就是说所有常见的模板特性（比如循环、调用上层函数、嵌入 partial 等）都可以实现。如此简单的特性甚至还带来了一个额外的好处：你不需要去学一个新的模板语言或者 DSL，因为你已经会 Ruby 了。

你可以这样来调用其他模板：

```
% render("path/to/template.template", {})
```

也可以写注释：

```
% # this is a comment
```

或者执行 blocks ：

```
% 3.times do |i|
{% raw %}{{{% endraw %}i{% raw %}}}{% endraw %}
% end
```

根据上面的特性，我们可以来编写一个示例模板：

```
<html>
<body>
% if access == 0
  <div> no access :( </div>
% else
  <ul>
  % data.each do |i|
    <li>{% raw %}{{{% endraw %}i{% raw %}}}{% endraw %}</li>
  % end
  </ul>
% end
% # comments are just normal ruby comments
</body>
</html>
```

下面我们称这个模板为 `index.template`。

下面我们要做的就是怎样编写一个方法来解析这个模板并输出正确的字符串。在此之前，我们先来看一个中间步骤：如何使用纯粹的 Ruby 代码来输出 HTML。

## 与 index.template 行为相同的 render 函数

在模板引擎出现以前，你同样可以使用纯 Ruby 来达到与 `index.template` 一样的效果，就像下面的代码：

```ruby
def render_index(access, data)
  output = "" # 一个新的字符串，用来存放输出的内容
  output << "<html>"
  output << "<body>"

  if access == 0
    output << "<div> no access :( </div>"
  else
    output << "<ul>"
    data.each do |i|
      output << "<li>#{ i }</li>"
    end
    output << "</ul>"
  end

  output << "</body>"
  output << "</html>"

  return output
end
```

你可以把它粘贴到 IRB 中试试：

```
>> render_index(0,["foo", "bar"])
=> "<html><body><div> no access :( </div></body></html>"
>> render_index(1,["foo", "bar"])
=> "<html><body><ul><li>foo</li><li>bar</li></ul></body></html>"
```

假如你的应用涉及的范围非常有限，或许你就根本用不着模板引擎，这样就大功告成了！你可以根据需要自己编写 `render_index()`、`render_header()`、`render_footer()` 方法。PHP 本身其实就是一个模板引擎，这也解释了为什么 PHP 社区的人们会经常这么做。

然而我们来看 `render_index()` 的目的是为了探寻如何将 `index.template` 转换成 `render_index()` ，并将转换的方法共通化，这样我们的模板引擎就出来了。我们并不想真的去编写 `render_index()`、`render_header()`、`render_footer()` 方法，也不想用代码生成器来实现。我们想要的是能够动态生成一个行为类似 `render_index()` 的方法，而不是亲手来写 `render_index()` 方法的代码。

在看具体的实现方式之前， 我们再来看一个中间步骤：

```ruby
def define_render_index()

  func = "" # 空字符串，用来存储构建方法的字符串
  func << "def render_index(access, data) \n"
  func << "output = \"\" \n"
  func << "output << \"<html>\" \n"
  func << "output << \"<body>\" \n"
  func << "if access == 0\n"
  func << "  output << \"<div> no access :( </div>\" \n"
  func << "else\n"
  func << "   output << \"<ul>\" \n"
  func << "      data.each do |i|\n"
  func << "        output << \"<li> \#{ i } </li>\" \n"
  func << "      end \n"
  func << "   output << \"</ul>\" \n"
  func << "end\n"
  func << "  output << \"</body>\" \n"
  func << "  output << \"</html>\" \n"
  func << "  return output \n"
  func << "end\n"

  eval(func)
end
```

你可以把它粘贴到 IRB 中并调用：

```
>> define_render_index() 
=> nil
>> render_index(1, ["foo", "bar"])
=> "<html><body><ul><li>foo</li><li>bar</li></ul></body></html>"
```

现在我们就得到一个更为完整的蓝图了：我们可以将原始的模板逐行转换为一系列字符串，然后修改这些字符串使其能够被 Ruby 解释。生成了这个方法以后，在调用它时就会执行我们所期望的模板的行为。

上面这个逐行转换的过程就是我们的 `parse` 方法的根基，下面就让我们具体来看。

## Parse 方法

### 1. 使用 Proc

我们不想让 `func` 字符串像 `define_render_index()` 一样以一个具名函数开头，所以我们用 Proc 并将其保存至一个变量，有需要时就 `.call` 它。

### 2. 设定 Proc 的变量

`define_render_index()` 还写死了其中的变量： access 和 data。但我们需要将变量名传给 `parse` 函数，这样才能构建定义 Proc 的字符串。这里我们把变量名直接以字符串形式传递给 parse 方法，就像这样：

```ruby
parse(template, "access, data")
```

### 3. 将模板逐行转译成函数字符串

上面的 `define_render_index()` 已经告诉我们逐行处理模板来创建一个新的 Ruby 方法时所需遵循的规则了。这些规则就是：

- 所有的双引号必须被转义，每一行的内容用双引号围起来，并在每一行的最后添加换行符 "\n"。

- 如果行的第一个字符（不包括空格 ）是 `%`，就把 `%` 删除。

```
变换前:
  % if data.empty? 
变换后:
  "if data.empty?\n"
```

- 其他的所有行都在前面加上 `"output <<"`

```
变换前:
  <html>
变换后:
  "output << \"<html>\" \n"
```

### 4. {% raw %}{{{% endraw %} ... {% raw %}}}{% endraw %} 会被变换为 #{ ... }

我们会使用正则表达式来实现这一点

```
变换前:
  <li>{% raw %}{{{% endraw %} i {% raw %}}}{% endraw %}</li> 
变换后:
  "output << \"<li> \#{ i } </li>\" \n"
```

要运用上述规则，我们首先要用 `.split("\n")` 将原始模板文件分割成数组，这样数组中的每个元素就是模板文件的每一行了。然后循环这个数组来构建 `func` 字符串，并最终将其 `eval`。

最后，我们得到的 parse 函数如下：

```ruby
def parse(template,vars = "")
  lines = File.read(template).split("\n")

  func = "Proc.new do |#{vars}| \n output = \"\" \n "

  lines.each do |line|
    if line =~ /^\s*(%)(.*?)$/
     func << " #{line.gsub(/^\s*%(.*?)$/, '\1') } \n" 
    else
     func << " output << \" #{line.gsub(/\{\{([^\r\n\{]*)\}\}/, '#{\1}') }\" \n "
    end
  end

  func << " output; end \n "

  eval(func)
end
```

你可以在 IRB 中亲自尝试一番：

```
>> index = parse("index.template", "access, data")
=> nil
>> index.call(1,["Foo"]) 
=> " <html>\n <body>\n<ul>\n<li>foo</li> \n</ul>\n </body>\n </html>\n"
```

大功告成了！就这么几行代码，你就实现了一个颇为强劲的模板引擎了。由于没有涉及太多多余的功能，我们将复杂度降到了最低，也提高了模板语言的清晰程度。复杂度的降低能够让应用程序条例更清晰，更不容易出错，还能加快开发特性的速度。

## 它能扩展（Scale）吗？

这里的代码不行！但是 [mote](https://github.com/soveran/mote)，也就是本文的灵感来源，当然可以。mote 附带了一些 helper 方法和缓存功能，我们已经成功将其运用在了各种大型的 Web 应用中。更不用提 [mote 的速度相当快](https://github.com/soveran/mote/blob/master/benchmarks/result.txt)。

同时我还想就其简洁这个显著的特点说两句——虽然 mote 的体积极其小，但对于其要解决的问题给出了一个专注而又完整的解决之道。

我希望这篇文章能够会对从未探寻过模板引擎的人或者想要做一个模板引擎的人有所帮助。欢迎留下你的评论或者反馈。
