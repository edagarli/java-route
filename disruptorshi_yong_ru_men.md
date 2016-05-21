# Disruptor使用入门

在最近的项目中看到同事使用到了Disruptor，以前在ifeve上看到过关于Disruptor的文章，但是没有深入研究，现在项目中用到了，就借这个机会对这个并发编程框架进行深入学习。项目中使用到的是disruptor-2.10.4，所以下面分析到的Disruptor的代码是这个版本的。
[并发编程网](http://ifeve.com/disruptor/)介绍Disruptor的文章是disruptor1.0版本，所以有一些术语在2.0版本上已经没有了或者被替代了。


### Disruptor术语

github上Disruptor的wiki对Disruptor中的术语进行了解释，在看Disruptor的过程中，对于几个其他的类，觉得有必要与这些术语放到一起，就加进来了。

