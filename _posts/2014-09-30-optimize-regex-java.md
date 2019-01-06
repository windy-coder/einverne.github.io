---
layout: post
title: "优化 Java 中正则表达式"
tagline: ""
description: ""
category: 经验总结
tags: [java, regex, ]
last_updated:
---

Java 中和正则相关的工具类都在 `java.util.regex` 包下，Java 使用了 Nondeterministic Finite Automaton （NFA)，之所以称为非确定性是因为当正则匹配给定字符串时，每一个字符都可能和正则匹配多次。这个匹配引擎被广泛的使用了，在 .NET, PHP, perl, Python, Ruby 中都是。

在引擎内部，NFA 使用回溯（backtracking) ，通常情况下针对给定的字符串，正则表达式可能有不止一种匹配方式，那么规则匹配引擎则会尽可能匹配所有可能，直到失败。

为了更好的理解 NFA 和回溯，举一个简单的例子，比如有一个正则表达式 `sc(ored|ared|oring)x` ，而输入的文本是 `scared`。

在开始的时候，引擎会查找到 `sc`，立即找到匹配的前两个字符，接着会从第三个字符开始匹配 `ored`，发现不匹配则会尝试下一个可能 `ared`，这个匹配成功，继续匹配后面的 `x`，此时发现不成功，那么引擎会回溯回去匹配第三个可能 `oring`，同样也没有匹配。那么引擎会回溯到第二个字符开始继续匹配 `sc`，直到匹配到字符结尾，然后抛出匹配失败。

### 优化回溯
在上面的例子中就能看到，NFA 使用回溯来进行规则匹配，上面的例子也非常容易发现回溯的一个问题，就是即使是很简单的例子，也可能导致回溯非常多次。由此可以想象，当回溯失控时对应用程序的性能影响。因此优化正则表达式的一个很重要的部分就是尽量减少回溯的次数。

因为回溯的存在，在遇到现实复杂情况时，正则匹配可能需要花费大量时间来寻找完整的匹配。更糟糕的是，引擎花费大量的时间来匹配从而宣告一次失败，这个时间远比一次成功的匹配来得多。因此需要记住的是，当测试正则表达式的速度时，测试正则匹配不成功的时候，而不是测试成功的匹配。而当测试匹配时，则尽量使用刚刚好匹配的字符串，因为这种字符串花费最长的时间。

### 避免重新编译正则
当程序中需要不止一次使用相同的正则表达式时，预先编译好后使用。先使用 `Pattern.compile()` 定义好，而不是直接使用 `Pattern.matches()`。`matches()` 方法内部每一次都会重新编译表达式。

并且记住对于不同的输入字符串，我们可以复用 `Matcher` 对象，不过要记住调用 `reset()` 方法。

### 合理使用括号
如果不需要引用括号内容，使用非捕获型括号 `(?:PATTERN)`，节省捕获时间，减少回溯使用状态数量。只在必要时使用括号。

### 不要滥用字符组

比如在 `[Ff]` 应当使用不区分大小写的匹配而不是字符组

### 使用起始终止锚点

除非是及其罕见的情况，否则以 .* 开头的正则表达式都应该在最前面添加 `^` 或者 `\A` 如果这个正则表达式在某个字符串的开头不能匹配，那么显然在其他位置它也不能匹配。添加锚点无论是手工添加还是通过优化自动添加都能够配合开头字符 / 字符串 / 字串识别优化，节省大量不必要的工作

在表达式前面独立出 ^ 和 \G

`^(?:abc|123)` 和 `(^abc|^123)` 在逻辑上是等价的，但是许多正则引擎指挥对第一个表达式使用 开头字符 / 字符串 / 字串识别优化。所以第一种办法的效率高得多

在表达式结尾独立出 `$`

确定起止锚点能够提高匹配速度，如果使用了 `$` 标记结尾，正则引擎会从符合条件的长度开始匹配，而略过目标字符串中其他不关心的字符。

### 暴露尽可能多的匹配部分
用 `xx*` 替代 `x+` 能够暴露必须匹配的 x 同样，用`-----{0,2}`代替`-{5,7}`。

用 `th(?:is|at)` 替代 `(?:this|that)` ，就能暴露出必须的 `th`。如果不同的多选分支的结尾部分相同，我们也可以从右面"提取"。例如 `(?:optim|standard)ization`。

对于传统 NFA，一旦匹配到结果就会停止。


### 用字符组代替分支条件
比如对于想要匹配的 a 或 b 或 c 或 d，使用 `[a-d]` 而不要使用 `(?:a|b|c|d)`。字符组比使用分支条件速度快。

## 谨慎使用点号
谨慎使用点号，星号，加号这样的元字符，只要能确定范围，就不要用点号。只要能够预测重复范围，就不要使用量词。

## reference

- <https://www.javaworld.com/article/2077757/core-java/optimizing-regular-expressions-in-java.html>
- <https://stackoverflow.com/q/1252194/1820217>