---
title: Imperative vs Declarative
---
## Conception
Imperative Programming 和 Declarative Programming是两种不同的编程>理念。首先，先总体地说一下二者的定义或者说是关键的区分点：

- Imperative Programming: telling the "machine" how to do something, and, as a result, what you want to happen will happen. 
- Declarative Programming: telling the "machine" what you would like to happen, and let the computer figure out how to do it.

以上提供给我们的最直接的概念就是imperative是程序员在程序中明确的指明了应该如何去做一件事，而declarative是程序员仅仅在程序中制定自己要做的事情，至于该如何去做，则交由程序或者系统自己去决定。 

## Examples
### 例1
如果我们要实现一个数组中的元素的累加，imperative programming 会倾向于有程序员自己定义一个for loop来实现累加，而declarative则可能仅仅使用一个sum()方法，对于累加这件事，imperative是由编程者指定了每一步的操作，而declarative则是编程者仅仅指明了累加这件事。**可以这样说，imperative programming指定了任务的具体实现，而declarative programming指定了任务的内容。**

### 例2
SQL是一个declarative programming的例子，假设有这样一串query：
```
    SELECT * from dogs
    INNER JOIN owners
    WHERE dogs.owner_id = owners.id
```
这里我们并没有告诉SQL应该怎样去一步一步查找我们想要的结果，而是直接告诉SQL我们需要什么样的结果，至于如何查找，那是数据库自己去解决的问题。

## Thinking 
最早的时候，计算机的资源有限，因此必须精打细算每一步执行，这是imperative programming的起源，尽管到了今天，imperative programming依然流行，C，C++，Jave，Ruby等等都是imperative programming的风格，而declarative programming是一种更加接近人类思维的方式，它更加抽象，更加high-level，它总结出的用来描述逻辑的语言能够使我们的程序更加简洁，直观。这里我不会去说两者哪个性能更好，毫无疑问，一个经验一般的程序员实现的imperative不会比隐藏在declarative后面的由专业人员优化的版本更快，但同样的，有时候一个通盘考虑的imperative实现打破了在declarative中任务之间的壁垒，也许会取得比declarative更好的效果也说不定。

从语言发展的角度来看，它总是朝着更高级，更抽象的角度发展，如果某项任务有着一种内在的逻辑可以抽象出来，使用declarative的方式提供一种一劳永逸的方法应该是一种更好的选择。尽管在某些场景下确实需要我们指定how，可能是出于对性能的不满意，也可能是没有一种现成的抽象表达来适配我们想要做的事情，这些时候我们需要去实现一些imperative code。但是换一种角度，当我们发现某件事情没有一个让我们满意的抽象库的实现，我们同样可以考虑是否可以由自己去实现一个。虽然工作量可能大于imperative，但从长远的角度来看，同样也好处多多。

imperative和declarative的界限其实也并非那么泾渭分明，今天的imperative也可能是昨天的declarative，而今天的declarative到了明天未尝不会变成更高级的declarative背后的一个imperative步骤。

## Reference
[*Imperative vs Declarative*](http://latentflip.com/imperative-vs-declarative)
[*Imperative programming or declarative programming*](https://www.puritys.me/zh_cn/docs-blog/article-320-Imperative-programming-or-declarative-programming.html)

