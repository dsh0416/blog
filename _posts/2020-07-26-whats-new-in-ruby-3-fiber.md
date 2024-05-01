---
title: Ruby 3 Fiber 变化前瞻
date: 2020-07-26 18:16:43 +0900
tags: [Fiber, Ruby]
---

随着 [GitHub #3032](https://github.com/ruby/ruby/pull/3032) 的合并，从 [Feature #13618](https://bugs.ruby-lang.org/issues/13618) 开始的，关于 Ruby Fiber 调度器的讨论取得了实质性的进展。但相关的变化还没有结束。目前正在被讨论与还没有合并的 Issue 还包括 [Feature #16786](https://bugs.ruby-lang.org/issues/16786)、[Feature #16792](https://bugs.ruby-lang.org/issues/16792)。这些 Issue 正在围绕 Ruby Fiber 调度器剩余的一些实现进行讨论，这些围绕着 Fiber 技术展开的对并发的实现，将作为 Ruby 3 并发提升的重要来源之一。

Ruby 3 Fiber 调度器会给我们带来什么？如何理解 Ruby 3 Fiber 调度器的引入？如何面对 Ruby 3 Fiber 的新变化？本文就此些问题进行一些讨论。

## 为什么要有 Fiber？

现代操作系统一个基本的特性就是允许多任务的执行。这个「多任务」可能是多线程或者多进程系统。对于一个 CPU，一个典型的情况是拥有 8 个左右的核心数，所以理论上只能同时执行 8 个任务。但操作系统同时执行的进程数往往有数千个，并不能「真正」同时运行。而操作系统需要在不同进程中快速切换从而实现多任务的同时运行。

现代操作系统使用的调度系统称为抢占式调度系统。简单理解，就是任务运行过程中，如果其它任务急需运行，操作系统会强制停止当前任务来执行其它任务。更传统的操作系统会使用协作式多任务（cooperative multitasking）系统来实现。也就是一个正在执行的任务必须主动宣布自己可以暂停运行，系统才会把执行权交给其它任务。Windows 3.1x、Mac OS 9 就是使用该方法进行的任务调度。

协作式多任务有着显著的优点和缺点。优点是切换的频率减少，执行效率提高了。而缺点是如果有程序发生了死循环或者长时间占用，系统就会陷入卡死，用户体验极差。

然而不同于操作系统，对于单一程序内，协作式多任务有时会带来更大的好处。由于线程是由操作系统实现和管理的，调度必须依赖操作系统，而一次操作系统的切换会带来很大的耗时。相比操作系统无法确定程序会不会发生[死循环](https://zh.wikipedia.org/wiki/%E5%81%9C%E6%9C%BA%E9%97%AE%E9%A2%98)，自己的程序内部代码完全是自己控制的，如果发生死循环那必然是自己的代码问题。在自己的程序内实现一个简单的协作式多任务系统来提高并发显然是个好办法。

而 Ruby 标准库就实现了一个简单的协作式多任务系统，其中的最小的执行单元称为 `Fiber` 纤程。提供了 `resume` `yield` 和 `transfer` 方法，实现了纤程之间的切换。

Fiber 的实现很简单，早年 Ruby Fiber 是基于 `caller`、 `callee` 来实现的。熟悉 Lisp 语言的，对这两个函数可能是再熟悉不过了。但是对于现在的 Ruby Fiber 实现，主要可以参考 [Feature #14739](https://bugs.ruby-lang.org/issues/14739) 的实现。由于这个代码是多个机器平台的汇编实现（出于性能上的考虑），我们这里以 `amd64` 平台为例。

```asm
##
##  This file is part of the "Coroutine" project and released under the MIT License.
##
##  Created by Samuel Williams on 10/5/2018.
##  Copyright, 2018, by Samuel Williams. All rights reserved.
##

.text

# For older linkers
.globl _coroutine_transfer
_coroutine_transfer:

.globl coroutine_transfer
coroutine_transfer:
	# Save caller state
	pushq %rbp
	pushq %rbx
	pushq %r12
	pushq %r13
	pushq %r14
	pushq %r15

	# Save caller stack pointer
	movq %rsp, (%rdi)

	# Restore callee stack pointer
	movq (%rsi), %rsp

	# Restore callee stack
	popq %r15
	popq %r14
	popq %r13
	popq %r12
	popq %rbx
	popq %rbp

	# Put the first argument into the return value
	movq %rdi, %rax

	# We pop the return address and jump to it
	ret
```

简简单单，非常好理解。`amd64` 的 callee-saved register 是 `%rbx` (base pointer), `%rbp` (frame prointer) 以及 `%r12` `%r13` `%r14` `%r15`。把这 6 个指针塞入栈，然后把栈指针 `%rsp` 返回。而还原一个上下文则是把这个栈顶指针找出来，然后依次取出这 6 个指针，就还原了上下文。 

## Fiber 与 I/O

但是要想让 Fiber 来提升 Web 系统的并发问题，还需要解决一个问题，那就是基于 I/O 的调度。我们清楚地知道，如果我们收到一个连接，在 Web 请求传输完之前，我们的 Ruby 程序什么都不能做，只能干等。而当我们处理完返回结果后，我们还是要干等到数据传输完后才能关闭连接。虽然现代的 [rack](https://github.com/rack/rack) 服务器例如 [puma](https://github.com/puma/puma) 能够异步解决这一问题。但是一旦涉及到 Redis、数据库和文件读写，我们依然逃不开这个问题。这是包括 Rails 在内的 Ruby 几乎所有 Web 框架性能问题的主要原因。

如果我们能围绕 I/O 设计一个 Fiber 调度器，那么我们就能极大提高 Ruby Web 框架的性能，但是这个问题并不是没有人做过。从早年的 [EventMachine](https://github.com/eventmachine/eventmachine) 到基于 [nio4r](https://github.com/socketry/nio4r) 的 [async](https://github.com/socketry/async)，包括我自己写的 [midori](https://github.com/midori-rb/midori.rb) 内单独实现的调度器 [murasaki](https://github.com/midori-rb/murasaki)，都是相同的原理。虽然这些框架的细节、性能和功能略有不同。

Ruby 今天 Fiber 自动异步调度仍然没有称为主流的核心原因是社区的分裂。

这几个开源的调度器都有一些小问题，然而大家的解决方法就是「一言不合，再写一个」。这使得像是 ActiveRecord 之类的常用框架都很难跟进这些快速迭代的调度器。根本方法就是大家合力来维护同一个调度器，让这个调度器进入标准库。这就是 [GitHub #3032](https://github.com/ruby/ruby/pull/3032) 的核心思路。

Scheduler 主要实现了三个核心的调度形式 `scheduler.wait_writable` `scheduler.wait_readable` 和 `scheduler.wait_sleep`。也就是当 Fiber 需要等待 I/O 完成写入、读取或者需要休眠时，就会主动将工作权让渡出来，交给其它 Fiber。从而实现基于多个 Fiber 的单线程内的并发性能提升。

## 目前 Fiber 调度器剩余的问题

目前 Scheduler 使用 `poll` 和 `select` 方法实现 I/O 的多路复用，而未来显然会支持 Linux 上的 `epoll` 、BSD 上的 `kqueue`、Windows 上的 `iocp` 来实现更好的多路复用性能，而无需调整 API。因为以目前 Ruby Scheduler 的 API 定义，是可以兼容这些多路复用方法的。而至于会不会去支持 macOS 的 `kqueue` 可能就要打个问号了，毕竟 macOS 的多路复用实现太 buggy 了。

另一个 Ruby 3 Fiber 亟待解决的问题是目前的 `Mutex` 锁是基于线程的。而对于同一个 Thread 下多个 Fiber 出现的锁竞争，`Mutex` 会遇到不小的问题。而目前各个已有的框架都是通过元编程在业务上解决的，比如我 `midori-contrib` 中对 MySQL 的[封装](https://github.com/midori-rb/midori-contrib/blob/master/lib/midori-contrib/sequel/mysql2.rb#L74)就使用了一系列奇技淫巧来避免问题。不过好在 [Feature #16792](https://bugs.ruby-lang.org/issues/16792) 正在针对这一问题提出方案，希望在 Ruby 3 之前能够有比较好的解决。

## 如何迎接 Ruby 3 Fiber 的新变化？

如何你是单纯的 Ruby 高级框架的使用者，那么你几乎什么都不用做。你只需要等着你常用的框架例如 Rails、Sinatra、ActiveRecord、Sequel 更新来支持这一特性，你的 Web 性能就理应会得到质的飞跃。根据我个人的实测，Ruby 的 Web 服务受到 I/O 调度问题而损失的性能高达 80% 到 90%，这意味着随着你使用的库全面支持 Fiber 的自动调度后，性能有望提升 5-10 倍。

如果你是 Ruby 框架的维护者和贡献者，那么你要做的事情就相对比较多。本来我想在这篇文章中进一步讨论 Fiber 调度器的使用，不过由于 API 还有很大的变化的可能，并且你需要使用 `ruby-head` 版本才能进行体验，我决定把该内容放在之后的文章里讲。核心的就是要尽快让你的 gem 中涉及底层 I/O 调用、锁实现和计时实现兼容新 Fiber。因为对于一个任务的 I/O 阻塞来说，一处阻塞处处阻塞，良好的性能必须要由完全不阻塞的 I/O 实现才能做到，否则都会受到显著的影响。

如果你是 Ruby 的贡献者，并且于 Ruby 不需要额外引入类似 `async` `await` 的原语而实现 I/O 无痛的性能提升很感兴趣的话，Ruby 3 Fiber 调度器需要做的事情还很多。比如对 `epoll` `kqueue` `iocp` 的支持；比如对 Ruby 2.x 的 backports。请不要害羞，请尽情贡献你的代码吧。

## 结

Ruby 今天 27 岁了，慢慢步入中年。但是我们依然能看到这门步入中年的语言里闪烁着令人激动的新特性的光辉。也许中年危机不单单是中年危机，更是中年转机。而这份转机靠的是我们每一个 Ruby 的使用者、贡献者和宣传者，让更多的程序员开心起来。
