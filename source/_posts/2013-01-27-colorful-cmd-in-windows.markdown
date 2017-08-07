---
layout: post
title: "在 Windows 的命令提示符 (cmd) 中执行 RSpec 时显示颜色"
date: 2013-01-19 10:58
comments: true
categories: weblog
---

TDD (测试驱动开发)的常规流程是「红 -> 绿 -> 重构」。[RSpec](http://rspec.info) 是 Rails 项目中使用的很普遍的一个测试的工具。在 Linux 和 Mac 的 Bash 终端中，Rspec 能够很好的用颜色来显示测试的执行结果。通过的 case 显示为绿色，没通过的用红色来显示。但是 Rspec 在 Windows 的命令行里却看不到 RSpec 五彩缤纷的执行结果，取而代之的是一群丑陋并且有点可笑的符号。

其实在 Windows 的命令提示符 （cmd.exe） 中也是可以显示 RSpec 执行结果的颜色的。我们需要使用「[ANSICON](https://github.com/adoxa/ansicon)」这个工具。

<!-- more -->

执行步骤也相当简单。

1. 下载「ANSICON」。[下载链接](http://adoxa.3eeweb.com/ansicon/dl.php?f=ansicon)
2. 解压
3. 打开命令提示符，32位操作系统的话移动路径至「x86」文件夹，64位系统使用「x64」文件夹。
4. 输入以下命令，回车即可。

```
C:\x86>ansicon -i
```

现在，你应该就可以看到 RSpec 绚丽多彩的执行结果了。不光如此，很多其他能够显示颜色的场合，比如使用 rails 的 generate 语句的时候，执行结果也会有颜色的显示了。
