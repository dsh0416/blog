---
title: 尝试使用 Ruby 3 调度器
date: 2020-08-18 21:46:47 +0900
tags: [Ruby, 编程]
---

## 一次失败的提案

在准备 RubyConf China 2020 的时候，我仔细检查了 [Fiber 调度器](https://github.com/ruby/ruby/pull/1870) 提出的补丁。当我看调度器的样例代码的时候，我发现其调用的是 Ruby 中的 `IO.select` API。`IO.select` API 在 Ruby 内部有多种实现，它可能调用 `poll`、大尺寸 `select`、POSIX 兼容的 `select` 取决于不同的操作系统。于是我想用一些更快的 syscall 来实现，比如 `epoll` `kqueue` 和 `IOCP`。

我做了一个相关的[提案](https://bugs.ruby-lang.org/issues/17059)但是被拒绝了。主要问题是 Ruby 的 `IO.select` API 是无状态的。如果没有含状态的注册，这些新 API 的性能甚至会不如 `poll`。在 [Koichi Sasada](https://bugs.ruby-lang.org/issues/17059#note-14) 跑了 banchmark 证明了这一点后，提案被正式拒绝。在和 Samuel Williams 在 Twitter 上讨论后，它建议我从 `Scheduler` 的实现上来进行注入，因为 `Scheduler` 本身是有状态的。于是我开始写一个 [gem](https://github.com/dsh0416/evt) 作为 Ruby 3 调度器接口的概念证明。

## 实现调度器

本文中的 Ruby 版本是：

```
ruby 2.8.0dev (2020-08-18T10:10:09Z master 172d44e809) [x86_64-linux]
```

基本的 Scheduler 例子来自于 Ruby 的[单元测试](https://github.com/ruby/ruby/blob/b2976a4fcab70bf9323180fd5ba6c29a5bca0747/test/fiber/test_scheduler.rb)。这是 Ruby 3 调度器的测试，而不是真正用于生产的，因此是使用 `IO.select` 进行 I/O 多路复用。因此我们可以基于此，开发一个性能更好的 Ruby 调度器。

我们需要做一些 C 开发来支持其它 syscall，因此第一件事是兼容原始的实现。

### Fallback 到 Ruby `IO.select`

对于 select/poll API, 不需要预先创建文件描述符，也不需要在运行时注册文件描述符。所以唯一要做的就是处理调度器触发时的行为。

```c
VALUE method_scheduler_wait(VALUE self) {
    // return IO.select(@readable.keys, @writable.keys, [], next_timeout)
    VALUE readable, writable, readable_keys, writable_keys, next_timeout;
    ID id_select = rb_intern("select");
    ID id_keys = rb_intern("keys");
    ID id_next_timeout = rb_intern("next_timeout");

    readable = rb_iv_get(self, "@readable");
    writable = rb_iv_get(self, "@writable");

    readable_keys = rb_funcall(readable, id_keys, 0);
    writable_keys = rb_funcall(writable, id_keys, 0);
    next_timeout = rb_funcall(self, id_next_timeout, 0);

    return rb_funcall(rb_cIO, id_select, 4, readable_keys, writable_keys, rb_ary_new(), next_timeout);
}
```

我们花了 10 行 C 干了原来 1 行 Ruby 就干好了的事。主要是这允许我们用 C 的宏定义来控制，从而使用其它 I/O 多路复用方法，例如 `epoll` and `kqueue`。我们需要实现 4 个 C 方法：

```ruby
Scheduler.backend
scheduler = Scheduler.new

scheduler.register(io, interest)
scheduler.deregister(io)
scheduler.wait
```

```c
#include <ruby.h>

VALUE Evt = Qnil;
VALUE Scheduler = Qnil;

void Init_evt_ext();
VALUE method_scheduler_init(VALUE self);
VALUE method_scheduler_register(VALUE self, VALUE io, VALUE interest);
VALUE method_scheduler_deregister(VALUE self, VALUE io);
VALUE method_scheduler_wait(VALUE self);
VALUE method_scheduler_backend();

void Init_evt_ext()
{
    Evt = rb_define_module("Evt");
    Scheduler = rb_define_class_under(Evt, "Scheduler", rb_cObject);
    rb_define_singleton_method(Scheduler, "backend", method_scheduler_backend, 0);
    rb_define_method(Scheduler, "init_selector", method_scheduler_init, 0);
    rb_define_method(Scheduler, "register", method_scheduler_register, 2);
    rb_define_method(Scheduler, "deregister", method_scheduler_deregister, 1);
    rb_define_method(Scheduler, "wait", method_scheduler_wait, 0);
}
```

`Scheduler.backend` 是专门给调试用的，剩下 4 个 API 会注入到调度器的 `Scheduelr#run`, `Scheduelr#wait_readable`, `Scheduelr#wait_writable`, `Scheduelr#wait_any` 中。

### 使用 `epoll` 和 `kqueue`

epoll 的三个核心 API 是 `epoll_create` `epoll_ctl` `epoll_wait`。很好理解，我们只要在调度器初始化的时候初始化 `epoll` fd，然后在注册 I/O 事件的时候调用 `epoll_ctl`，最后用 `epoll_wait` 替换掉 `IO.select`。

```c
#if defined(__linux__) // TODO: Do more checks for using epoll
#include <sys/epoll.h>
#define EPOLL_MAX_EVENTS 64

VALUE method_scheduler_init(VALUE self) {
    rb_iv_set(self, "@epfd", INT2NUM(epoll_create(1))); // Size of epoll is ignored after Linux 2.6.8.
    return Qnil;
}

VALUE method_scheduler_register(VALUE self, VALUE io, VALUE interest) {
    struct epoll_event event;
    ID id_fileno = rb_intern("fileno");
    int epfd = NUM2INT(rb_iv_get(self, "@epfd"));
    int fd = NUM2INT(rb_funcall(io, id_fileno, 0));
    int ruby_interest = NUM2INT(interest);
    int readable = NUM2INT(rb_const_get(rb_cIO, rb_intern("WAIT_READABLE")));
    int writable = NUM2INT(rb_const_get(rb_cIO, rb_intern("WAIT_WRITABLE")));
  
    if (ruby_interest & readable) {
        event.events |= EPOLLIN;
    } else if (ruby_interest & writable) {
        event.events |= EPOLLOUT;
    }
    event.data.ptr = (void*) io;

    epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event);
    return Qnil;
}

VALUE method_scheduler_deregister(VALUE self, VALUE io) {
    ID id_fileno = rb_intern("fileno");
    int epfd = NUM2INT(rb_iv_get(self, "@epfd"));
    int fd = NUM2INT(rb_funcall(io, id_fileno, 0));
    epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL); // Require Linux 2.6.9 for NULL event.
    return Qnil;
}

VALUE method_scheduler_wait(VALUE self) {
    int n, epfd, i, event_flag, timeout;
    VALUE next_timeout, obj_io, readables, writables, result;
    ID id_next_timeout = rb_intern("next_timeout");
    ID id_push = rb_intern("push");
    
    epfd = NUM2INT(rb_iv_get(self, "@epfd"));
    next_timeout = rb_funcall(self, id_next_timeout, 0);
    readables = rb_ary_new();
    writables = rb_ary_new();

    if (next_timeout == Qnil) {
        timeout = -1;
    } else {
        timeout = NUM2INT(next_timeout);
    }

    struct epoll_event* events = (struct epoll_event*) xmalloc(sizeof(struct epoll_event) * EPOLL_MAX_EVENTS);
    
    n = epoll_wait(epfd, events, EPOLL_MAX_EVENTS, timeout);
    // TODO: Check if n >= 0

    for (i = 0; i < n; i++) {
        event_flag = events[i].events;
        if (event_flag & EPOLLIN) {
            obj_io = (VALUE) events[i].data.ptr;
            rb_funcall(readables, id_push, 1, obj_io);
        } else if (event_flag & EPOLLOUT) {
            obj_io = (VALUE) events[i].data.ptr;
            rb_funcall(writables, id_push, 1, obj_io);
        }
    }

    result = rb_ary_new2(2);
    rb_ary_store(result, 0, readables);
    rb_ary_store(result, 1, writables);

    xfree(events);
    return result;
}

VALUE method_scheduler_backend() {
    return rb_str_new_cstr("epoll");
}
#endif
```

`kqueue` 是类似的。唯一不同的是，BSD 的注册和等待用的是同一个 API，只是参数不同，所以有点难懂。

```c
#if defined(__FreeBSD__) || defined(__NetBSD__) || defined(__APPLE__)
#include <sys/event.h>
#define KQUEUE_MAX_EVENTS 64

VALUE method_scheduler_init(VALUE self) {
    rb_iv_set(self, "@kq", INT2NUM(kqueue()));
    return Qnil;
}

VALUE method_scheduler_register(VALUE self, VALUE io, VALUE interest) {
    struct kevent event;
    u_short event_flags = 0;
    ID id_fileno = rb_intern("fileno");
    int kq = NUM2INT(rb_iv_get(self, "@kq"));
    int fd = NUM2INT(rb_funcall(io, id_fileno, 0));
    int ruby_interest = NUM2INT(interest);
    int readable = NUM2INT(rb_const_get(rb_cIO, rb_intern("WAIT_READABLE")));
    int writable = NUM2INT(rb_const_get(rb_cIO, rb_intern("WAIT_WRITABLE")));
    
    if (ruby_interest & readable) {
        event_flags |= EVFILT_READ;
    } else if (ruby_interest & writable) {
        event_flags |= EVFILT_WRITE;
    }

    EV_SET(&event, fd, event_flags, EV_ADD|EV_ENABLE, 0, 0, (void*) io);
    kevent(kq, &event, 1, NULL, 0, NULL); // TODO: Check the return value
    return Qnil;
}

VALUE method_scheduler_deregister(VALUE self, VALUE io) {
    struct kevent event;
    ID id_fileno = rb_intern("fileno");
    int kq = NUM2INT(rb_iv_get(self, "@kq"));
    int fd = NUM2INT(rb_funcall(io, id_fileno, 0));
    EV_SET(&event, fd, 0, EV_DELETE, 0, 0, NULL);
    kevent(kq, &event, 1, NULL, 0, NULL); // TODO: Check the return value
    return Qnil;
}

VALUE method_scheduler_wait(VALUE self) {
    int n, kq, i;
    u_short event_flags = 0;

    struct kevent* events; // Event Triggered
    struct timespec timeout;
    VALUE next_timeout, obj_io, readables, writables, result;
    ID id_next_timeout = rb_intern("next_timeout");
    ID id_push = rb_intern("push");

    kq = NUM2INT(rb_iv_get(self, "@kq"));
    next_timeout = rb_funcall(self, id_next_timeout, 0);
    readables = rb_ary_new();
    writables = rb_ary_new();

   events = (struct kevent*) xmalloc(sizeof(struct kevent) * KQUEUE_MAX_EVENTS);

    if (next_timeout == Qnil || NUM2INT(next_timeout) == -1) {
        n = kevent(kq, NULL, 0, events, KQUEUE_MAX_EVENTS, NULL);
    } else {
        timeout.tv_sec = next_timeout / 1000;
        timeout.tv_nsec = next_timeout % 1000 * 1000 * 1000;
        n = kevent(kq, NULL, 0, events, KQUEUE_MAX_EVENTS, &timeout);
    }

    // TODO: Check if n >= 0
    for (i = 0; i < n; i++) {
        event_flags = events[i].filter;
        if (event_flags & EVFILT_READ) {
            obj_io = (VALUE) events[i].udata;
            rb_funcall(readables, id_push, 1, obj_io);
        } else if (event_flags & EVFILT_WRITE) {
            obj_io = (VALUE) events[i].udata;
            rb_funcall(writables, id_push, 1, obj_io);
        }
    }

    result = rb_ary_new2(2);
    rb_ary_store(result, 0, readables);
    rb_ary_store(result, 1, writables);

    xfree(events);
    return result;
}

VALUE method_scheduler_backend() {
    return rb_str_new_cstr("kqueue");
}
#endif
```

## 使用调度器的 HTTP 服务器例子

在实现好调度器后，我们要测试调度器的性能。因此我写了一个简单的 HTTP 服务器 [benchmark](https://github.com/dsh0416/evt-server-benchmark)。

```ruby
require 'evt'

puts "Using Backend: #{Evt::Scheduler.backend}"
Thread.current.scheduler = Evt::Scheduler.new

@server = Socket.new Socket::AF_INET, Socket::SOCK_STREAM
@server.bind Addrinfo.tcp '127.0.0.1', 3002
@server.listen Socket::SOMAXCONN
@scheduler = Thread.current.scheduler

def handle_socket(socket)
  line = socket.gets
  until line == "\r\n" || line.nil?
    line = socket.gets
  end
  socket.write("HTTP/1.1 200 OK\r\nContent-Length: 0\r\n\r\n")
  socket.close
end

Fiber.new(blocking: false) do
  while true
    socket, addr = @server.accept
    Fiber.new(blocking: false) do
      handle_socket(socket)
    end.resume
  end
end.resume

@scheduler.run
```

比起原先阻塞的 I/O，使用 Ruby 3 非阻塞 I/O 后可以达到 3.33x 的性能，而使用 `epoll` 后可以达到 4.21x。服务器的例子很简单，所以当 JIT 启动时，不容易造成 ICache 不命中，因此性能进一步提升到了 4.54x。

![Benchmark Result](/assets/images/ruby-scheduler-benchmark.png)

测试是基于 Intel(R) Xeon(R) CPU E3-1220L V2 @ 2.30GHz CPU 的，而且程序是单线程的。如果有更好的 CPU，`epoll` 和 `poll` 的差距会更大。欢迎尝试，相关 gem 代码已开源。

## 未来工作

未来工作主要是两部分。一个是提升现有 API 的稳定性，还有就是加入 `io_uring` 和 `IOCP` 的支持。`io_uring` 倒是还好，但我是一点都不懂 Windows 开发。所以欢迎大家来提供意见和贡献。

## 源码

[dsh0416/evt](https://github.com/dsh0416/evt)
