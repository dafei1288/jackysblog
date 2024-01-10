---
title: 一种基于LLM（大语言模型）的栈机实现
date: 2024-01-05 14:21:00
tags: [大预言模型,llm,jimlang,java,编译原理]
---

最近在给jimlang升级，从ast解析型变成编译成中间码（IR）模拟java的字节码形式，交给栈机去执行，在实现栈机的过程中，突发奇想，是不是可以通过LLM的agnet来完成栈机的任务呢？虽然执行效率上就别提了，但是这个思路还是挺有意思的，于是说写就写，一个晚上的时候，完成了极简原型。

整体架构如下：
![](../img/blog/1005/arch.png)

<!-- more -->

受到LLM agent的启发：
![](../img/blog/1005/agent.jpg)


DEMO如下：
![](../img/blog/1005/demo.png)

jimlang的栈机实现部分

![](../img/blog/1005/jvmopcode.png)

![](../img/blog/1005/opcode.jpg)

![](../img/blog/1005/stackm.jpg)



