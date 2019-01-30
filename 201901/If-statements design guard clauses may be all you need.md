# If-statements design: guard clauses may be all you need（If 语句设计: 卫语句可能就是你所需要的）

There is a lot of debate on how the if-statements should be used for better code clarity and readability. Most of them boil down to an opinion that its completely subjective and aesthetic. And usually that’s true, but sometimes we still can find a common ground. This post aims to derive a small set of advices for a few such cases. Hopefully this will reduce the amount of thinking and debating when writing something as simple as an if-statement.

对于如何使用 if 语句来提高代码的清晰度和可读性，有很多争论。其中大部分观点归根结底都是从个人主观性的和代码美观触发的。虽然事实这样，但我们仍然可以从其中找到一些共同点。这篇文章的目的是就是提供一些建议。希望能减少写使用简单语句如 if 语句时的思考和争论。

## Guard Clauses（卫语句）

Probably the most documented case is the guard clause, also known as assert or precondition. The idea is that when you have something to assert in the beginning of a method — do this using a fast return.

大多数文档中给出建议使用卫语句的例子为断言或者前提条件。建议是这样的，当你在方法的开头进行断言时使用 return 快速返回值。

```
public void run( Timer timer ) {
  if( !timer.isEnabled() )
    return;
  
  if( !timer.valid() )
    throw new InvalidTimerException();
  timer.run();
}
```

Despite looking obvious, it is often left behind due to the fact that our brains function differently: we tend to think “run the timer if it’s enabled”, not “run the timer, but if it’s not enabled do nothing”. However, in the code, the latter leads to a better separation of corner cases from the main method logic, if we strictly follow that only something asserted in the beginning can be considered a guard clause.

尽管看起来很明显，但由于我们的大脑运行方式不同导致这一事实常常被抛在脑后：我们倾向于认为`如果启用了计时器就运行计时器`，而不是`运行计时器，但如果没有启用则不执行任何操作`。 然而，在代码编写过程中，后者可使边缘案例与主方法逻辑的更好分离，如果我们严格遵循只要在开头断言就考虑使用卫语句。

As appears, the guard clause is not nearly a new idea. Yet I see it as one of the most straight-forward and simple ones, which can be accepted without a lot of debate. But the biggest revelation for me was that most of the cases when if-statement is used can be transformed to use a guard clause instead, and this is what I’m going to elaborate on.

看起来卫语句并不是一个新概念。我认为这是最直截了当的方法之一，不需要很多争论就可以被大多数人接受。这对我来说最大的启示是，大多数使用 if 语句的情况都可以改为卫语句，这也是我后续要阐述的内容。

## Large if-blocks（大 if 代码块）

It is pretty common to find something like the following in a codebase:

在代码库中常常能查找到类似以下内容：

```
if( something.isEnabled() ) {
  // pretty
  // long 
  // logic
  // of
  // running
  // something
}
```

Usually it isn’t designed this way from the beginning. Instead, such blocks appear when a new requirement appears which says “we should run that something only in case of X”. The most straight-forward solution, of course, would be to wrap the “run that something” part into an if-statement. Easy.

通常从一开始不是这样设计的。但当一个新的需求出现时，就会出现这样的块，比如 “我们应该只在 x 的情况下运行它”。最直接的解决方案是将 `run that something` 部分包装成 if 语句。如此简单。

The problem is that actually it’s only easy to write, but not to read. See, to understand the code one needs to derive the whole picture from it, a concept to further work with — because humans are much better in operating on concepts, not on algorithms. Introducing this kind of condition in the beginning creates an additional context, which has to be kept in mind when working with the main method logic. Obviously enough, we should strive to reduce the amount of mental context required at any given moment.

问题是，实际上它只容易编写，但不易读。为了理解代码，我们需要从整个思考这段代码进而将其转换为编码思路，因为人类在处理思路上要比在算法上好很多。开始就引入这种条件会创建一个额外的上下文，尤其是使用主方法逻辑时特意注意这一点。我们应该努力减少任何时刻使用这种思维方式。

Another big problem arises when the logic of such code becomes long enough to not fit the screen vertically. At some point you may even forget that it’s there, which would completely break your picture.

当这种代码的逻辑变得足够长而不能完整显示在屏幕时，就会出现另一个大问题。在某个时刻，你甚至可能忘记它在那里，这将会打断你的思路。

To avoid all of this, you just need to invert the if-statement so that the condition would become a guard clause:

为了避免这些问题，您只需要讲 if 语句翻转，这样条件就变成了一个卫语句：

```
if( !something.isEnabled() )
  return;
// same
// long 
// logic
// of
// running
// something
```

Again, there’s a separation of concerns: first you get rid of the preconditions at the beginning (throwing them away from your mind, too — which is important to better focus on the main logic), and then just do what the method is required to do.

这样会出现思路的分离：首先在开始时去掉了前提条件（把它们从你的思路中抛弃，这能更好的关注主要逻辑），然后按照方法的要求去做要做的。

## Returning in the middle（中间返回值）

Multiple returns are considered a bad idea for reasons, and this is probably one of the biggest. Contrary to a return in a guard clause, a return or throw in the middle of the method is not so easily detected at the first sight. Like any other conditional logic, it complicates the process of building a mental picture of the method. Moreover, it requires you to consider that under some circumstances, the main logic of this method will not be executed completely, and every time you’ll have to decide: should you put the new code before or after this return?

多个 `return` 被认为是一个坏主意的原因，这可能是最大的一个。对比卫语句，在函数中间返回值或抛出异常不那么容易第一眼就意识到。与其他条件逻辑一样，他会使方法的思维图谱变的复杂化。此外，它还要求您考虑在某些情况下，此方法的主要逻辑不会完全执行，每次您都必须决定：您应该在返回值之前还是之后书写新代码？

Here’s an example of some real code:

下面是一些实际代码的示例：

```
public void executeTimer( String timerId ) {
  logger.debug( "Executing timer with ID {}", timerId );
  TimerEntity timerEntity = timerRepository.find( timerId );
  logger.debug( "Found TimerEntity {} for timer ID {}", timerEntity, timerId );
  if( timerEntity == null )
    return;
  Timer timer = Timer.fromEntity( timerEntity );
  timersInvoker.execute( timer );
}
```

Looks like a precondition, doesn’t it? Let’s put it to where a precondition has to be:

看起来像一个前提条件，难道不是吗？让我们讲起变为一个前提条件的样子：

```
public void executeTimer( String timerId ) {
  logger.debug( "Executing timer with ID {}", timerId );
  TimerEntity timerEntity = timerRepository.find( timerId );
  logger.debug( "Found TimerEntity {} for timer ID {}", timerEntity, timerId );
  executeTimer( timerEntity );
}

private void executeTimer( TimerEntity timerEntity ) {
  if( timerEntity == null )
    return;
  Timer timer = Timer.fromEntity( timerEntity );
  timersInvoker.execute( timer );
}
```

This refactored code is much easier to read, since there are no conditions at all in the main logic, there’s just a mere sequence of simple actions. What I wanted to show is that it’s almost always possible to split a method which has a return in the middle so that the preconditions are nicely separated from the main logic.

这种重构的代码更容易阅读，因为在主逻辑中根本没有条件，只包含一些简单的操作。我想表达的是，中间返回值可以通过方法进行切割，这样前置条件就可以很好地与主逻辑分离。

## Small conditional actions（短小的条件语句）

It is also a common practice to have lots of small conditional statements like this:

使用短小的条件语句也是一种常见的做法：

```
if( timer.getMode() != TimerMode.DRAFT )
  timer.validate();
```

This is totally legit in general case, but often is just a hidden precondition which should better be placed in a method itself. You’ll probably notice this further, when each time you invoke the method you need to add this if-statement because the business logic says so. Considering that, the proper solution would be:

这在一般情况下是完全有效的，但隐藏的前提条件是最好将其放在一个方法中。您可能会注意到这样一点，每次调用该方法时都需要添加此 if 语句，因为业务逻辑就是这样。考虑到这一点，更好的解决方案是：

```
public void validate() {
  if( mode == TimerMode.DRAFT )
    return;
  
  // validation logic
}
```

And the usage is now as simple as:

现在的用法很简单：

```
timer.validate();
```

## Conclusion（结论）

The use of guard clauses is a good practice to avoid unnecessary branching, and thus make your code more lean and readable. I believe if these small advices are considered systematically, it will greatly decrease the maintenance cost of your piece of software.

使用卫语句是避免不必要分支的好时间，会使您的代码更加精简和可读。我相信更加系统地考虑这些小建议，它将大大降低您软件的维护成本。

And at last, generally, there’s nothing bad in having a conditional statement here or there in your code. Though my experience shows that if you never care about trying to avoid that, then at some point you’ll end up wasting hours building a mental picture of a complex highly branched piece of code while trying to put that next thing a business wants into it.

最后，通常在代码中有一个条件语句并没有什么不好。虽然我的经验表明，如果你从不关心试图避免这种情况，那么你尝试讲下一段业务逻辑插入其中的时候讲会话费大量的时间构建一个负责且多分支的思路图然后再编码。

## 原文

[If-statements design: guard clauses may be all you need](https://medium.com/@scadge/if-statements-design-guard-clauses-might-be-all-you-need-67219a1a981a)

****
**THE END [ 2019-01-30 ]**