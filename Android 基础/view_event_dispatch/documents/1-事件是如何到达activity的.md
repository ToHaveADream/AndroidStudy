# 事件是如何到达 activity 的

> 事件分发，真的一定从 Activity 开始吗？

事件分发的流程：Activity -> window -> view。这个我们大致都知道，那事件是怎么到达 Activity 的呢？如果了解过 Window 机制的话会知道，事件分发也是 Window 的一部分，而 Activity 不属于 Window 机制内，那么触摸事件应该是从 Window 开始才对，怎么是从 Activity 开始呢？抱着这些疑问，重新学习下事件分发。

接下来需要探究的是，一个触摸信息从系统底层产生之后，一步步到达 Activity 进行分发的整体流程。