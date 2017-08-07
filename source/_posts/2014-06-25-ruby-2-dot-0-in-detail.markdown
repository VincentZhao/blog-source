---
layout: post
title: "Ruby 2.0.0 更新详解"
date: 2014-06-25 20:32
comments: true
categories: translation
---
> 译者注：截至目前，Ruby 的最新版已经更新到了 2.1.2。为什么今时今日还要翻译 2.0.0 版本的变更点呢？众所周知，语言越小众，用户的升级越激进。而 Ruby 进入 1.9 以后，已经名副其实地跻身主流语言。当一门语言被广泛使用以后，版本升级的步伐也会相对放缓。目前我在工作上的项目使用的依然是 Ruby 1.9.3，我当时学习 Ruby 时候的版本的也是 1.9。但显然，将版本升级到 2.0 以上是大势所趋，为了全面地了解 1.9.3 到 2.0.0 之间发生了哪些变化，我选择了这篇非常全面的文章进行了翻译。希望对有同样需求的人有所帮助。

原文： [Ruby 2.0.0 in Detail](http://globaldev.co.uk/2013/03/ruby-2-0-0-in-detail/)

## 关键字参数（Keyword arguments）

```ruby
def wrap(string, before: "<", after: ">")
  "#{before}#{string}#{after}" # 无需从 hash 中获取 options
end

# 有无任意
wrap("foo")                                  #=> "<foo>"
# 数量任意
wrap("foo", before: "#<")                    #=> "#<foo>"
wrap("foo", after: "]")                      #=> "<foo]"
# 顺序任意
wrap("foo", after: "]", before: "[")         #=> "[foo]"
```

相比用 hash 来实现的伪关键字参数的方式，该特性的好处之一为，当出现单词拼写错误时会报错。

```ruby
begin
  wrap("foo", befoer: "#<")
rescue ArgumentError => e
  e.message                                  #=> "unknown keyword: befoer"
end
```

如同使用一个星号来捕获所有的常规参数一样，可以使用两个星号来捕获所有的关键字参数。两个星号还能用于将 hash 转换为关键字参数。

```ruby
# arguments
def capture(**opts)
  opts
end
capture(foo: "bar")                          #=> {:foo=>"bar"}

# key 必须为 symbol
opts = {:before => "(", :after => ")"}
wrap("foo", **opts)                          #=> "(foo)"
```

关键字参数仍然能够接收旧式的 hash 形式参数，因此你可以放心地更新方法定义，而无需修改每一处调用它的地方。

```ruby
wrap("foo", :before => "{", :after => "}")   #=> "{foo}"
```

如果你的库需要兼容 Ruby 2.0 和 1.9，那么仍然可以采用以前的 hash 形式的伪关键字参数，因为调用的时候和关键字参数的写法是一样的。

```ruby
def wrap(string, opts={})
  before = opts[:before] || "<"
  after = opts[:after] || ">"
  "#{before}#{string}#{after}"
end

wrap("foo", before: "[", after: "]")         #=> "[foo]"
```

<!-- more -->

## `%i` 和 `%I` 表示 symbol 数组的字面量

```ruby
%i{an array of symbols}   #=> [:an, :array, :of, :symbols]
```

也可以使用插值（interpolation）

```ruby
%I{#{1 + 1}x #{2 + 2}x}   #=> [:"2x", :"4x"]
```

## Refinements

Refinements 是个出色的想法，但目前的实现方式还不成熟，可能会出现诡异的情况或引发性能问题，所以 Ruby 2.0.0 的 Refinements 还处于实验阶段，实用性并不高。

可以对一个类创建一个 refinement，把它放到一个模块中

```ruby
module NumberQuery
  refine String do
    def number?
      match(/\A[1-9][0-9]*\z/) ? true : false
    end
  end
end
```

默认情况下，该 refinement 是不可见的

```ruby
"123".respond_to?(:number?)   #=> false
```

只有使用 `using` 来声明要使用这个模块中的 refinement 之后，才会变得可见。

```ruby
using NumberQuery
"123".number?                 #=> true
```

然而，`using` 只能够在顶层（top level）使用，并且适用范围仅为同一源文件并且位于声明之后的代码。如果你开启了警告，就会收到关于 refinements 还处于实验阶段的警告。

## `Module#prepend`

为了向 `#include` 致敬，module 添加了 `#prepend` 方法，用法与 `include` 相同，但是是将模块作为当前类的子类插入到继承链中。

```ruby
Object
superclass
included module
class
prepended module
```

这个特性可以取代 Rails 的 `#alias_method_chain`，或者在重新定义方法之前先给它设定一个别名方法再调用原方法的技巧。

```ruby
class Foo
  def do_stuff
    puts "doing stuff"
  end
end

module Wrapper
  def do_stuff
    puts "before stuff"
    super
    puts "after stuff"
  end
end

class Foo
  prepend Wrapper
end

Foo.new.do_stuff
```

输出为：

```ruby
before stuff
doing stuff
after stuff
```

正如 `::included` 和 `::append_features` 一样，这次也添加了 `::prepended` 和 `::prepend_features`。

## 模块中没有被绑定的方法可以被绑定到任意对象

这个特性听上去有些绕口，你或许以为是永远不会用到的某个小改进，事实上，这是一个很棒的新特性。

使用 `instance_method` 可以得到任意类或模块中的方法

```ruby
Object.instance_method(:to_s)   #=> #<UnboundMethod: Object(Kernel)#to_s>
```

但这个方法没有被绑定，没有 `self`，所以无法被调用。要调用它，你必须将其绑定到一个对象上，在 2.0 之前，我们只能把它绑定到相同类的一个对象上。

然而现在，我们可以从模块中取出任意一个方法并把它绑定到任意对象上了

```ruby
module Bar
  def bar
    "bar"
  end
  def baz
    "baz"
  end
end

Bar.instance_method(:bar).bind(Object.new).call   #=> "bar"
```

这意味着 `define_method` 也能接收模块中未绑定的方法了，由此我们可以实现选择性的 `include`

```ruby
module Kernel
  def from(mod, include: [])
    raise TypeError, "argument must be a module" unless Module === mod
    include.each do |name, original|
      define_method(name, mod.instance_method(original || name))
    end
  end
end

class Foo
  from Bar, include: {:qux => :bar}
end

f = Foo.new
f.qux                 #=> "bar
f.respond_to?(:baz)   #=> false
```

## `const_get` 能理解命名空间了

```ruby
class Foo
  module Bar
    Baz = 1
  end
end

Object.const_get("Foo::Bar::Baz")   #=> 1
```

## `#to_h` 成为了转换成 hash 的标准方法

`Hash`，以及 `ENV`、`nil`、`Struct` 和 `OpenStruct` 都有了返回 hash 类型的 `#to_h` 方法

```ruby
{:foo => "bar"}.to_h               #=> {:foo=>"bar"}
nil.to_h                           #=> {}
Struct.new(:foo).new("bar").to_h   #=> {:foo=>"bar"}
require "ostruct"
open = OpenStruct.new
open.foo = "bar"
open.to_h                          #=> {:foo=>"bar"}
```

和 `Array()` 一样，也有一个 `Hash()` 方法，内部委托给了 `#to_h`

```ruby
Hash({:foo => "bar"})              #=> {:foo=>"bar"}
Hash(nil)                          #=> {}
```

## `Array#bsearch` 和 `Range#bsearch`

Array 和 Range 添加了二分查找方法 `#bsearch`。它有两种运行模式：查找最小值和查找任意值。两种方式都接收一个 block，数组必须相对该 block 是有序的。

查找最小值模式会返回第一个大于或等于指定值的元素。使用该模式时，你只要让 block 在元素大于或等于指定值时返回 true，反之返回 false。

```ruby
array = [2, 4, 8, 16, 32]
array.bsearch {|x| x >= 4}       #=> 4
array.bsearch {|x| x >= 7}       #=> 8
array.bsearch {|x| x >= 9}       #=> 16
array.bsearch {|x| x >= 0}       #=> 2
array.bsearch {|x| x >= 33}      #=> nil
```

查找任意值模式中，提供的 block 在元素比指定值小时返回一个正数，比指定值大时返回一个负数，相等则返回 0。

```ruby
array.bsearch {|x| 4 <=> x}      #=> 4
array.bsearch {|x| 7 <=> x}      #=> nil
```

你的 block 可以让不止一个值返回 0，这种情况下，这些值中的任意一个都可能被选中。

```ruby
array = [0, 4, 7, 10, 12]
array.map {|x| 1 - x / 4 }       #=> [1, 0, 0, -1, -2]
array.bsearch {|x| 1 - x / 4 }   #=> 4 or 7
```

## `Enumerable#lazy`

任意 `Enumerable`（`Array`、`Hash`、`File`、`Range` 等）调用 `#lazy` 后会返回一个惰性枚举（lazy enumerator），它只有在被告之要进行计算时才会进行计算。另外在链式调用时，
枚举元素依次被传入调用链，而不是在每一步都评估整个枚举元素。这样的方式在某些场合下能减少计算量。

你可以这样来处理无穷集合

```ruby
[1,2,3].lazy.cycle.map {|x| x * 10}.take(5).to_a   #=> [10, 20, 30, 10, 20]
```

或者当你只需要少量数据时，可以避免庞大的资源消耗

```ruby
File.open(__FILE__).lazy.each.map(&:chomp).reject(&:empty?).take(3).force
```

上面这个示例中，当找到了3行非空行后就不会再继续读取文件。

`#lazy` 若使用不当时会引起性能问题，所以要确保用于正确的场合。

同样可以使用 `Enumerator::Lazy` 类来创建惰性枚举。上述示例也可以写成

```ruby
def populated_lines(file, &block)
  Enumerator::Lazy.new(file) do |yielder, line|
    string = line.chomp
    yielder << string unless string.empty?
  end.each(&block) # evals block, or returns enum if nil, like stdlib
end

populated_lines(File.open(__FILE__)).take(3).force
```

## 惰性的 `Enumerator#size` 和 `Range#size`

现在，`Enumerator#size` 无需循环评估整个枚举就能返回集合的容量，Range 也一样。

```ruby
array = [1,2,3,4]
array.cycle(4).size    #=> 16
array.cycle.size       #=> Infinity
# 如果无法计算容量则返回 nil
array.find.size        #=> nil
# Range too
(1..10).size           #=> 10
```

为了使自己代码返回的 `Enumerators` 也能享受到这个特性，现在，`Enumerator.new` 可以接收一个参数，该参数就用于计算 size。

参数可以是一个值

```ruby
enum = Enumerator.new(3) do |yielder|
  yielder << "a"
  yielder << "b"
  yielder << "c"
end

enum.size              #=> 3
```

也可以是一个能调用 `#call` 方法的对象

```ruby
def square_times(num, &block)
  Enumerator.new(-> {num ** 2}) do |yielder|
    (num ** 2).times {|i| yielder << i}
  end.each(&block)
end

square_times(6).size     #=> 36
```

现在，`#to_enum` （以及其别名方法 `#enum_for`）同样可以接收一个 block 来计算容量，上述代码也可以写成下面这样

```ruby
def square_times(num)
  return to_enum(:square_times) {num ** 2} unless block_given?
  (num ** 2).times do |i|
    yield i
  end
end

square_times(6).size     #=> 36
```

## Rubygems 支持 Gemfile

现在，Rubygems 支持使用 Gemfile （或 Isolate， 或 gems.deps.rb）来安装 gems 以及加载激活信息。

指定 `-- file` （或 `-g`）选项，就可以安装 Gemfile 中列出的 gems （以及依赖关系）了。该操作仅使用 Gemfile，而不是 Gemfile.lock，因此版本以 Gemfile 中指定的为准。

```ruby
gem install --file Gemfile
```

加载激活信息（即使用 gems 的哪个版本），需要指定 `RUBYGEMS_GEMDEPS` 环境变量，它的值应该是 Gemfile 的路径，不过你也可以使用 `-` 来让 Rubygems 自动检测。

和安装一样，这步操作也使用 Gemfile，而不是 Gemfile.lock，所以比起 Bundler 来说要更为宽松。并且，它只是将指定的版本激活，你仍然需要在代码中 require 这些 gems。

内置了这个特性确实带来了好处，我们不再需要使用 `bundle exec` 或 Bundlers binstub 这类东西了，而只需像往常一样启动应用程序。

```ruby
export RUBYGEMS_GEMDEPS=-
# 启动你的应用程序
```

该特性仅支持基本的 Gemfile 格式，比如它不支持声明 `gemspec`，然而它确实方便，且有望继续进化而变得更加强大，并且当 Bundler 不管用时，该特性就非常有用了。

`bundle install --path vendor/bundle` 可以粗略近似为

```ruby
gem install --file Gemfile --install-dir vendor/gem

export GEM_HOME=vendor/gem
export RUBYGEMS_GEMDEPS=-
# 启动你的应用程序
```

## RDoc 支持 Markdown 格式

RDoc 现在支持 markdown 了，可以使用 markup 选项来设置 rdoc 的格式。

```ruby
rdoc --markup markdown
```

该设置可以保存到你项目下的 `.doc_options` 文件中，这样就不用每次都重复上面的步骤了

```ruby
rdoc --markup markdown --write-options
```

## 像 puts 一样使用 warn

现在 warn 的用法和 puts 一样了，可以接收多个参数，或者一个数组，然后将其打印出来，只不过输出到 stderr 中，而不是 stdout。

```ruby
warn "foo", "bar"
warn ["foo", "bar"]
```

## Logger 兼容 syslog 接口

现在，将日志记录的场所在文件和 syslog 之间相互切换变得易如反掌，不再是二选一以后就不能改变了。你甚至可以在开发过程中将日志记录到 stdout 中，而在 production 环境下记录到 syslog，而无需修改所有调用日志操作的代码。

```ruby
if ENV["RACK_ENV"] == "production"
  require "syslog"
  logger = Syslog::Logger.new("my_app")
else
  require "logger"
  logger = Logger.new(STDOUT)
end

logger.debug("about to do stuff")
begin
  do_stuff
rescue => e
  logger.error(e.message)
end
logger.info("stuff done")
```

## TracePoint

`TracePoint` 是既有的 `Kernel#set_trace_func` 的面向对象版本。它能让你追踪 Ruby 代码的执行流程，这在调试逻辑混乱的代码的时候非常有用。

```ruby
# set up our tracer, but don't enable it yet
events = %i{call return b_call b_return raise}
trace = TracePoint.new(*events) do |tp|
  p [tp.event, tp.event == :raise ? tp.raised_exception.class : tp.method_id, tp.path, tp.lineno]
end

def twice
  result = []
  result << yield
  result << yield
  result
end

def check_idempotence(&block)
  a, b = twice(&block)
  raise "expected #{a} to equal #{b}" unless a == b
  true
end

trace.enable
a = 1
begin
  check_idempotence {a += 1}
rescue
end
trace.disable
```

输出为

```
[:call, :check_idempotence, "/Users/mat/Dropbox/ruby-2.0.0.rb", 841]
[:call, :twice, "/Users/mat/Dropbox/ruby-2.0.0.rb", 834]
[:b_call, nil, "/Users/mat/Dropbox/ruby-2.0.0.rb", 850]
[:b_return, nil, "/Users/mat/Dropbox/ruby-2.0.0.rb", 850]
[:b_call, nil, "/Users/mat/Dropbox/ruby-2.0.0.rb", 850]
[:b_return, nil, "/Users/mat/Dropbox/ruby-2.0.0.rb", 850]
[:return, :twice, "/Users/mat/Dropbox/ruby-2.0.0.rb", 839]
[:raise, #<RuntimeError: expected 2 to equal 3>, "/Users/mat/Dropbox/ruby-2.0.0.rb", 843]
[:return, :check_idempotence, "/Users/mat/Dropbox/ruby-2.0.0.rb", 843]
```

## 异步线程中断处理

Ruby 的线程可以被其他线程杀死或在其中抛出异常。这个特性并非绝对安全，因为行刑的线程不知道它要杀死的线程正在做什么，有可能一个线程正在进行一些重要的资源分配操作的途中就被杀死了。现今，我们有了更加安全的特性来处理此类情况。

举个例子，标准库中的 timeout 库的运行方式是新建一个线程，新线程在等待指定的时间后在原线程中抛一个异常。

假设我们有一个连接池库，它能够处理将连接放回失败的情况，但如果取出连接或放回连接的线程被中止则会失败。你写了一个方法来取得一个连接并用它发起一个请求然后将其返回，并且你猜测使用该方法的人可能会用 timeout 来对它进行包装。

```ruby
def request(details)
  result = nil
  # Block will defer given exceptions if they are raised in this thread by
  # another thread till the end of the block. Exceptions are not rescured or
  # ignored, but handled later.
  Thread.handle_interrupt(Timeout::ExitException => :never) do
    # no danger of timeout interrupting checkout
    connection = connection_pool.checkout
    # if checkout took too long, handle the interrupt immediately, effectively
    # raising the pending exception here
    if Thread.pending_interrupt?
      Thread.handle_interrupt(Timeout::ExitException => :immediate)
    end
    # allow interrupts during IO (or C extension call)
    Thread.handle_interrupt(Timeout::ExitException => :on_blocking) do
      result = connection.request(details)
    end
    # no danger of timeout interrupting checkin
    connection_pool.checkin(connection)
  end
end
```

上面这个方法能够安全地被 timeout 包装，连接始终能够被完全取出，如果完成了请求，连接始终能够被放回。如果在放回的时候发生了 timeout，放回不会被中断，但异常在方法的最后仍然会被抛出。

虽然这个例子不是很自然，但却很好的涵盖了这个优秀的新特性的要点。

## 垃圾回收的改进

Ruby 2.0 的垃圾回收机制有了一些改进，使得 Ruby 能够更好的进行写入时复制（Copy-on-Write）。这意味着多线程的应用程序，比如运行在 Unicorn 上的 Rails 应用，能够减少内存的使用量。

`GC::Profiler` 类新增了 `::raw_data` 方法，该方法以 hash 的数组形式返回原始数据，而不是字符串，这样能够更加便于 statsd 之类的工具来记录数据。

```ruby
GC::Profiler.enable # turn on the profiler
GC.start # force a GC run, so there will be some stats
GC::Profiler.raw_data
#=> [{:GC_TIME=>0.0012150000000000008, :GC_INVOKE_TIME=>0.036716,
#   :HEAP_USE_SIZE=>435920, :HEAP_TOTAL_SIZE=>700040,
#   :HEAP_TOTAL_OBJECTS=>17501, :GC_IS_MARKED=>0}]
```

## `ObjectSpace.reachable_objects_from`

该方法返回指定对象的所有可直接到达的对象。

```ruby
require "objspace"

Response = Struct.new(:code, :header, :body)
res = Response.new(200, {"Content-Length" => "12"}, "Hello world!")

ObjectSpace.reachable_objects_from(res)
#=> [Response, {"Content-Length"=>"12"}, "Hello world!"]
```

你可以结合 `ObjectSpace.memsize_of` 方法来取得一个对象和所有其引用的对象所占的内存容量，这在调试内存泄漏时非常有用

```ruby
def memsize_of_all_reachable_objects_from(obj)
  memsize = 0
  seen = {}.tap(&:compare_by_identity)
  to_do = [obj]
  while obj = to_do.shift
    ObjectSpace.reachable_objects_from(obj).each do |o|
      next if seen.key?(o) || Module === o
      seen[o] = true
      memsize += ObjectSpace.memsize_of(o)
      to_do << o
    end
  end
  memsize
end

memsize_of_all_reachable_objects_from(res)   #=> 192
```

## 更好的调用栈

现在，调用栈字符串不再随着每个异常的出现而生成，而是根据需要由轻量的对象集合立即生成。你可以使用 `caller_locations` 或 `Thread#backtrace_locations` 来得到这些对象。奇怪的是 `Exception` 无法调用这两个方法。

```ruby
def foo
  bar
end

def bar
  caller_locations
end

locations = foo
locations.map(&:label)   #=> ["foo", "<main>"]
locations.first.class    #=> Thread::Backtrace::Location
```

`caller` 现在可以接收取得的数量（limit）以及偏差（offset），或者是一个范围（range）

```ruby
def bar
  caller(2, 1)
end
foo   #=> ["/Users/mat/Dropbox/ruby-2.0.0.rb:361:in `<main>'"]

def bar
  caller(2..2)
end
foo   #=> ["/Users/mat/Dropbox/ruby-2.0.0.rb:368:in `<main>'"]
```

## 支持 Zlib 流

现在可以流式解压 Zilb 了，对 Zlib 的流式压缩的支持也得到了改进。

解压文件时通常会把压缩文件缓存至内存中，但未压缩的数据可能有数百 MB。`Zlib::Inflate#inflate` 现在可以接收一个 block，来将未压缩文件分块处理，这样就不需要把未压缩数据一次性全部缓存到内存中了。

```ruby
require "zlib"

inflater = Zlib::Inflate.new(Zlib::MAX_WBITS + 32)
File.open("app.log", "w") do |out|
  inflater.inflate(File.read("app.log.gz")) {|chunk| out << chunk}
end
inflater.close
```

类似的，`Zlib::Deflate#deflate` 同样能接收 block，当你不断地往 `#deflate` 中输送数据的过程中，只有当数据足够进行压缩时，处理才会被执行。

```ruby
deflater = Zlib::Deflate.new
File.open("app.log.gz", "w") do |out|
  File.foreach("app.log", 4 * 1024) do |chunk|
    deflater.deflate(chunk) {|part| out << part}
  end
  deflater.finish {|part| out << part}
end
```

## 多线程 Zlib 处理

Zlib 在处理过程中不再持有全局解释器锁（GIL），因此可以并发处理 gzip、zlib 以及 deflate 流。这也意味着应用程序可以在 Zlib 运行于后台的同时持续响应。

```ruby
require "zlib"

# processes 4 files in parallel, using 4 cores if required
threads = %W{a.txt.gz b.txt.gz c.txt.gz d.txt.gz}.map do |path|
  Thread.new do
    inflater = Zlib::Inflate.new(Zlib::MAX_WBITS + 32)
    File.open(path.chomp(File.extname(path)), "w") do |out|
      inflater.inflate(File.read(path)) {|chunk| out << chunk}
    end
    inflater.close
  end
end
# do other stuff here while Zlib works in other threads
threads.map(&:join)
```

## 编码默认为 UTF-8

现在不用在第一行写编码注释或者天书般的转义字符，也能够使用 US-ASCII 以外的字符了。

```ruby
currency = "€"   #=> "€"
```

## 取得二进制字符串的方法

`String#b` 可以方便地取得字符串的 ASCII-8BIT （也就是二进制）版

```ruby
s = "foo"
s.encoding     #=> #<Encoding:UTF-8>
s.b.encoding   #=> #<Encoding:ASCII-8BIT>
```

## `String#lines`、`#chars` 等方法返回数组

`lines`、`chars`、`codepoints`、`bytes` 方法不再返回 Enumerator ，而返回数组

```ruby
s = "foo\nbar"
s.lines        #=> ["foo\n", "bar"]
s.chars        #=> ["f", "o", "o", "\n", "b", "a", "r"]
s.codepoints   #=> [102, 111, 111, 10, 98, 97, 114]
s.bytes        #=> [102, 111, 111, 10, 98, 97, 114]
```

为了保持向下兼容性，这些方法仍然能够接收一个 block，但现在开始，这种情况你应该改用 `#each_line` 等方法。

在 `IO`、`ARGF`、`StirngIO` 和 `Zlib::GzipReader` 中的类似方法仍然返回 Enumerator，但这些方法已经被废弃，请使用 `each_*` 形式的版本。

## `__dir__`

类似 `__File__`，`__dir__` 返回文件的路径，但不包括文件名，在下列情况下很有用

```ruby
YAML.load_file(File.join(__dir__, "config.yml"))
```

## `__callee__` 返回调用它的方法名

`__callee__` 的返回值又回到了调用它的方法名，而不是定义别名方法了。这很有用。

```ruby
def do_request(method, path, headers={}, body=nil)
  "#{method.upcase} #{path}"
end

def get(path, headers={})
  do_request(__callee__, path, headers)
end
alias head get

get("/test")    #=> "GET /test"
head("/test")   #=> "HEAD /test"
```

## 正则表达式引擎换成了 Onigmo

1.9 的正则表达式引擎是 Oniguruma，Onigmo 是它的一个分支，新增了[一些特性](https://github.com/k-takata/Onigmo)。新特性似乎是受了 Perl 的影响，可以参考[这里](http://perldoc.perl.org/perlre.html)。

```
(?(cond)yes|no)
```

如果满足 cond 条件，则使用 yes 条件匹配，反之使用 no 条件匹配。cond 是 group number 或 name 的引用，或者是向前或向后匹配

下面的表达式匹配的是开头字母和结尾字母的大小写相同的字符串

```ruby
regexp = /^([A-Z])?[a-z]+(?(1)[A-Z]|[a-z])$/

regexp =~ "foo"   #=> 0
regexp =~ "foO"   #=> nil
regexp =~ "FoO"   #=> 0
```

## `Hash#default_proc=` 现在可接收 nil

这样就不再需要用 `hash.default = nil` 来清空 `hash.default_proc:` 了，现在新的代码更符合直觉。

```ruby
hash = {}
hash.default_proc = Proc.new {|h,k| h[k] = []}
hash[:foo] << "bar"
hash[:foo]                                       #=> ["bar"]
hash.default_proc = nil
hash[:baz]                                       #=> nil
```

## `Array#values_at` 对每个越界的值都返回一个 nil

以前当 `#values_at` 方法的参数是 range 时，对所有越界的 index 只返回一个 nil，现在改为对每个越界的 index 各返回一个 nil

```ruby
[2,4,6,8,10].values_at(3..7)   #=> [8, 10, nil, nil, nil]
```

## `File.fnmatch?` 可以通过选项来展开括号

如果出于一些原因你需要在 Ruby 中进行 shell 风格的文件名匹配，好消息是现在你可以使用 {foo, bar} 这样的模式了。

```ruby
# 3rd argument enables the brace expansion
File.fnmatch?("{foo,bar}", "foo", File::FNM_EXTGLOB)   #=> true
File.fnmatch?("{foo,bar}", "foo")                      #=> false
# or together multiple options old-school C style
casefold_extglob = File::FNM_CASEFOLD | File::FNM_EXTGLOB
File.fnmatch?("{foo,bar}", "BAR", casefold_extglob)    #=> true
```

## `Shellwords` 对参数调用 `#to_s`

`Shellwords#shellscape` 和 `#shelljoin` 现在会对参数调用 `#to_s`，和 `Pathname` 一起使用时会特别有用

```ruby
require "pathname"
require "shellwords"

path = Pathname.new("~/Library/Application Support/").expand_path
Shellwords.shellescape(path)
\#=> "/Users/mat/Library/Application\\ Support"

Shellwords.join(Pathname.glob("/Applications/A*"))
\#=> "/Applications/App\\ Store.app /Applications/Automator.app"
```

## `system` 和 `exec` 现在默认关闭非标准的文件描述符

使用 `exec` 时，除了 `STDIN`、`STDOUT` 和 `STDERR` 外所有打开的文件和套接字在新的进程中都会被关闭。以前可以通过指定选项（`cmd`, `close_others: true`）来做到，现在成了默认行为

## `#respond_to?` 对待 protected 方法同于 private 方法

和 `private` 方法一样，protected 方法不再被 `#respond_to?` 可见，除非第二个参数是 `true`

```ruby
class Foo
  protected
  def bar
    "baz"
  end
end

f = Foo.new
f.respond_to?(:bar)         #=> false
f.respond_to?(:bar, true)   #=> true
```

## `#inspect` 不再调用 `#to_s`

在 Ruby 1.9 以前，如果自定义了 `#to_s` 方法，`#inspect` 会被委托给这个 `#to_s` 方法，现在这个行为已经被去除了

```ruby
class Foo
  def to_s
    "foo"
  end
end

Foo.new.inspect   #=> "#<Foo:0x007fb4a2887328>"
```

## `LoadError#path`

`LoadError` 现在有了 `#path` 方法来获取无法加载文件的路径。虽然在错误消息里面也有，但现在能在代码中更方便地取得了

```ruby
begin
  require_relative "foo"
rescue LoadError => e
  e.message   #=> "cannot load such file -- /Users/mat/Dropbox/foo"
  e.path      #=> "/Users/mat/Dropbox/foo"
end
```

## `Process.getsid`

`getsid` 返回该进程的会话 ID。仅用于 unix/linux 操作系统

```ruby
Process.getsid   #=> 240
```

## `Signal.signame`

`signame` 用于获取指定号码的信号名

```ruby
Signal.signame(9)   #=> "KILL"
```

## 捕捉内部信号会报错

如果使用 `Signal.trap` 捕捉 `:SEGV`、`:BUS`、`:ILL`、`:FPE` 或 `:VTALRM` 时会抛出 `ArgumentError`，这些信号是 Ruby 内部使用的，因此你无法捕捉它们。

## 真正的线程局部变量

在 Ruby 1.9 中，`Thread#[]`、`#[]=`、`#keys` 和 `#key?` 用来对纤程的局部变量进行取值和设值，`Thread` 现在有了等价的 `#thread_variable_get`、`#thread_variable_set`、`#thread_variables` 和 `#thread_variable？` 方法，用来处理线程的局部变量

纤程局部变量

```ruby
b = nil

a = Fiber.new do
  Thread.current[:foo] = 1
  b.transfer
  Thread.current[:foo]
end

b = Fiber.new do
  Thread.current[:foo] = 2
  a.transfer
end

p a.resume   #=> 1
```

线程局部变量

```ruby
b = nil

a = Fiber.new do
  Thread.current.thread_variable_set(:foo, 1)
  b.transfer
  Thread.current.thread_variable_get(:foo)
end

b = Fiber.new do
  Thread.current.thread_variable_set(:foo, 2)
  a.transfer
end

p a.resume   #=> 2
```

## 改善加入当前线程/主线程时的错误消息

如果你试图对当前线程或主线程调用 `#join` 或 `#value` 方法，你会得到一个继承自 `StandardError` 的 `ThreadError`，而不是继承自 `Exception` 的 fatal。

```ruby
begin
  Thread.current.join
rescue => e
  e   #=> #<ThreadError: Target thread must not be current thread>
end
```

## 关于互斥锁的改动

我想不出一个有趣的实例来解释，但是你现在可以检查当前线程是否持有互斥锁了

```ruby
require "thread"

lock = Mutex.new
lock.lock
lock.owned?                      #=> true
Thread.new {lock.owned?}.value   #=> false
```

同样影响互斥锁的是，以下这些改变互斥锁状态的方法不再允许被用于信号处理： `#lock`、`#unlock`、`#try_lock`、`#synchronize` 和 `#sleep`。

另外，`#sleep` 可能会提前醒来，因此如果你对时间有精确的要求，你需要再次确认睡眠时间是否准确。

```ruby
sleep_time = 0.1
start = Time.now
lock.sleep(sleep_time)
elapsed = Time.now - start
lock.sleep(sleep_time - elapsed) if elapsed < sleep_time
```

## 自定义线程和纤程的栈大小

以下环境变量可以用来设置线程和纤程的栈容量。Ruby 只会在程序启动的时候检查它们。

- `RUBY_THREAD_VM_STACK_SIZE`： 用于创建线程的虚拟机栈容量。 默认值： 128KB (32位 CPU) 或 256KB (64位 CPU)。
- `RUBY_THREAD_MACHINE_STACK_SIZE`： 用于创建线程的机器栈容量。 默认值： 512KB 或 1024KB。
- `RUBY_FIBER_VM_STACK_SIZE`： 用于创建纤程的虚拟机栈容量。 默认值： 64KB 或 128KB。
- `RUBY_FIBER_MACHINE_STACK_SIZE`： 用于创建纤程的机器栈容量。 默认值： 256KB 或 256KB。

你可以这样来取得默认值：

```ruby
RubyVM::DEFAULT_PARAMS   #=> {:thread_vm_stack_size=>1048576,
                              :thread_machine_stack_size=>1048576,
                              :fiber_vm_stack_size=>131072,
                              :fiber_machine_stack_size=>524288}
```

## 更严格的 `Fiber#transfer`

转移出去的线程必须被转移回来，用 `resume` 来作弊的日子一去不复返了

```ruby
require "fiber"

f2 = nil

f1 = Fiber.new do
  puts "a"
  f2.transfer
  puts "c"
end

f2 = Fiber.new do
  puts "b"
  f1.transfer # under 1.9 this could have been a #resume
end

f1.resume
```

## `RubyVM::InstructionSequence`

`RubyVM::InstructionSequence` 不是新添加的，然而现在有了一些新的特性，使其变得更有用了，[具体文档在此](http://www.ruby-doc.org/core-2.0/RubyVM/InstructionSequence.html)。

你可以使用既有的方法取得指令序列

```ruby
class Foo
  def add(x, y)
    x + y
  end
end

instructions = RubyVM::InstructionSequence.of(Foo.instance_method(:add))
```

得到指令序列后，你还可以查看详细信息，比如它是在哪里被定义的

```ruby
instructions.path            #=> "/Users/mat/Dropbox/ruby-2.0.0.rb"
instructions.absolute_path   #=> "/Users/mat/Dropbox/ruby-2.0.0.rb"
instructions.label           #=> "add"
instructions.base_label      #=> "add"
instructions.first_lineno    #=> 654
```

## `ObjectSpace::WeakMap`

这个类是用于实现 `WeakRef` 的一部分，因此你最好使用 `WeakRef`（`require "weakref"`）。该类对存储的对象持有弱关联，意味着它们或许会成为垃圾回收的对象。

```ruby
map = ObjectSpace::WeakMap.new
# keys can't be immediate values (numbers, symbols), and you must use the
# exact same object, not just one that is equal.
key = Object.new

map[key] = "foo"
map[key]                #=> "foo"
# force a garbage collection run
sleep(0.1) and GC.start 
map[key]                #=> nil
```

## 顶层的 `define_method`

`define_method` 可以在顶层使用了，而不一定非要在类或模块中

```ruby
Dir["config/*.yml"].each do |path|
  %r{config/(?<name>.*)\.yml\z} =~ path
  define_method(:"#{name}_config") {YAML.load_file(path)}
end
```

## 以下划线开头的变量在没被使用时不会发生警告

以下方法会发生警告，因为 `family`、`port` 和 `host` 变量没被用到。

```ruby
def get_ip(sock)
  family, port, host, address = sock.peeraddr
  address
end
```

在以前版本中，虽说改用下划线可以抑制警告，但这样就降低了代码的可读性

```ruby
def get_ip(sock)
  _, _, _, address = sock.peeraddr
  address
end
```

到了 Ruby 2.0 ，我们可以做到两全其美，只要在变量名前加上 `_`

```ruby
def get_ip(sock)
  _family, _port, _host, address = sock.peeraddr
  address
end
```

## `Proc#==` 和 `Proc#eql?` 方法被删除

在 Ruby 1.9.3 以前，有内容相同以及绑定相同的 `Proc` 被视为是相等的，但实际上你只可能通过 `clone` 来得到相等的 `Proc`。该比较操作现在被删除了，因为实在没什么用处。

```ruby
proc = Proc.new {puts "foo"}

proc == proc.clone   #=> false
```

## `ARGF#each_codepoint`

`ARGF` 现在有了类似 `IO` 的 `#each_codepoint` 方法

```ruby
count = 0
ARGF.each_codepoint {|c| count += 1 if c > 127}
puts "there are #{count} non-ascii chacters in the given files"
```

## `Time#to_s`

`Time#to_s` 返回的字符串的编码由 ASCII-8BIT（二进制） 变为了 US-ASCII

```ruby
Time.now.to_s.encoding   #=> #<Encoding:US-ASCII>
```

## `Array#shuffle!` 和 `Array#sample` 的随机参数调用时使用 `max` 参数

这是一个小改动，或许对你完全没有影响，现在给 `Array` 的 `#shuffle!`、`#shuffle` 和 `#sample` 方法提供随机参数时，需要提供一个 `max` 参数，返回 0 到 max 之间一个整数，而不是 0 到 1 之间的浮点数。

```ruby
array = [1, 3, 5, 7, 9]
randgen = Object.new
def randgen.rand(max)
  max                           #=> 4
  1
end
array.sample(random: randgen)   #=> 3
```

## CGI HTML5 标签构建器

标准库中的 `CGI` 的标签构建器实例新增了 HTML5 模式

```ruby
require "cgi"
cgi = CGI.new("html5")
html = cgi.html do
  cgi.head do
    cgi.title {"test"}
  end +
  cgi.body do
    cgi.header {cgi.h1 {"example"}} +
    cgi.p {"lorem ipsum"}
  end
end
puts html
```

旧的 `#header` 方法（发送 HTTP 头）改名为 `#http_header`，其实它是 `#header` 方法的别名，因为考虑到非 HTML5 模式情况下的向下兼容性。

## `CSV::dump` 和 `CSV::load` 被删除

`CSV::dump` 和 `CSV::load` 被删除了。以前它们用来将 Ruby 的对象数组输出到 CSV 文件中，以实现序列化和反序列化。由于安全问题，这两个方法被删除了。

## `Iconv` 被删除

`Iconv` 被删除了，请使用 `String#encode`。

以前你可能写了如下形式的代码

```ruby
require "iconv"
Iconv.conv("ISO-8859-1", "UTF8", "Résumé")   #=> "R\xE9sum\xE9"
```

现在要写成

```ruby
"Résumé".encode(Encoding::ISO_8859_1)        #=> "R\xE9sum\xE9"
```

## `Syck` 被删除

YAML 解析器 Syck 被删除了，因为有了更好的 Psych（由 libyaml 绑定），并且 Ruby 现在捆绑了 libyaml。Ruby 中 YAML 的接口都维持原样，所以对代码应该没有影响。

## `io/console`

`io/console` 不是新添加的，更新的只是[文档](http://www.ruby-doc.org/stdlib-2.0/libdoc/io/console/rdoc/IO.html)，因此现在你可以知道怎么来使用它了。Ruby 2.0.0 的变更履历说新添加了 `IO#cooked` 和 `IO#cooked!`，但貌似 1.9.3 就有了。

```ruby
require "io/console"
IO.console.raw!
# console in now in raw mode, disabling line editing and echoing
IO.console.cooked!
# back in cooked mode, line editing works like normal
```

`#raw!` 和 `#raw` 新增了两个参数，`min` 和 `time`

```ruby
IO.console.raw!(min: 5) # reading from console buffers for 5 chars
IO.console.raw!(min: 5, time: 1) # read after 1 second if buffer not full
```

## `io/wait`

`io/wait` 新增了 `#wait_writable` 方法，用来阻塞直到 IO 能够被写。`#wait` 改名为 `#wait_readable`，为了向下兼容性，还保留了 `#wait` 的别名方法

```ruby
require "io/wait"
timeout = 1
STDOUT.wait_writable(timeout)   #=> #<IO:<STDOUT>>
```

## `Net::HTTP` 性能改进

`Net::HTTP` 现在默认自动请求以及解压 gzip 并且使用 DEFLATE 压缩。这和全新的无全局解释器锁的 Zlib 相得益彰。

SSL 会话现在被重用，减少了用于协商链接的时间。

## `Net::HTTP` 能指定连接元的主机和端口

如果出于某种原因，你需要指定指定本地连接元的主机和端口，和连接目标的主机和端口，可以这样

```ruby
http = Net::HTTP.new(remote_host, remote_port)
http.local_host = local_host
http.local_port = local_port
http.start do
  # ...
end
```

## `OpenStruct` 可以像 hash 一样

`OpenStruct` 有了 `#[]`、`#[]=` 和 `#each_pair` 方法，所以可以如 hash 一样使用

```ruby
require "ostruct"

o = OpenStruct.new
o.foo = "test"
o[:foo]               #=> "test"
o[:bar] = "example"
o.bar                 #=> example
```

同时它也新增了 `#hash` 和 `#eql?` 方法，这两个方法在 `Hash` 内部被用来检验相等性。这也使得 `OpenStruct` 能够更好地扮演 hash 键的作用，相等的对象能作为相同的键。

## `Resolv` 支持超时

`Resolv` 现在支持自定义超时

```ruby
require "resolv"

resolver = Resolv::DNS.new
resolver.timeouts = 1 # 1 second
resolver.getaddress("globaldev.co.uk").to_s   #=> "204.232.175.78"
```

同样也能接收数组，依次设置超时时间。可以用下面的代码来实现指数回归

```ruby
resolver.timeouts = [1].tap {|a| 5.times {a.push(a.last * 2)}}
```
