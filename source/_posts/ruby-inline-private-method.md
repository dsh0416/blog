---
title: Ruby 内联私有方法与原理
date: 2018-07-03 21:35:49
tags: [编程, Ruby]
---

孔乙己有一回对我说道，“你学过 Ruby 么？”我略略点一点头。他说，“学过 Ruby，……我便考你一考。private 私有方法，怎样写的？”我想，讨饭一样的人，也配考我么？便回过脸去，不再理会。孔乙己等了许久，很恳切的说道，“不能写罢？……我教给你，记着！这些字应该记着。将来做 CTO 的时候，写码要用。”我暗想我和 CTO 的等级还很远呢，而且我们 CTO 也从不在 Ruby 里写 private 方法；又好笑，又不耐烦，懒懒的答他道，“谁要你教，不是  `private`  换行之后就是  `def`  的都是私有方法么？”孔乙己显出极高兴的样子，将两个指头的长指甲敲着柜台，点头说，“对呀对呀！…… `private`  的三种写法，你知道么？”我愈不耐烦了，努着嘴走远。孔乙己刚用指甲蘸了酒，想在柜上写字，见我毫不热心，便又叹一口气，显出极惋惜的样子。

孔乙己这里说的 `private` 的三种写法其中有两种是非常常见的，第一种是：

```ruby
class Test
    private
    def a
        'a'
    end
end
```

另一种则是：

```ruby
class Test
    def a
        'a'
    end
 	private :a
end
```

这两种各有优劣，第一种  `private`  后的方法都是私有方法，有效组织了方法顺序。而第二种则可以随意定义方法，之后再定义哪些是  `private`  方法。

之前在看 Rubocop 的代码风格的一些 Issue 讨论的时候，看到了  `private`  的第三种写法，目前 midori 项目中使用的是这种作法：内联私有方法。

```ruby
class Test
    private def a
        'a'
    end
end
```

这种写法的好处是，可以很清楚知道某一个方法是不是私有方法，也可以随意组织方法的顺序。但是奇怪了，似乎很少有看到 Ruby 正式的文档里提到过这种写法，这到底是一种什么写法？Ruby 为什么支持这种写法？

我们运行一下：

```bash
ruby -e "class Test; private def a; 'a'; end; end" --dump parseTree
```

看一下 Ruby 对这段魔法代码的 AST 树：

```
###########################################################
## Do NOT use this node dump for any purpose other than  ##
## debug and research.  Compatibility is not guaranteed. ##
###########################################################

# @ NODE_SCOPE (line: 1, code_range: (1,0)-(1,40))
# +- nd_tbl: (empty)
# +- nd_args:
# |   (null node)
# +- nd_body:
#     @ NODE_PRELUDE (line: 1, code_range: (1,0)-(1,40))
#     +- nd_head:
#     |   (null node)
#     +- nd_body:
#     |   @ NODE_CLASS (line: 1, code_range: (1,0)-(1,40))
#     |   +- nd_cpath:
#     |   |   @ NODE_COLON2 (line: 1, code_range: (1,6)-(1,10))
#     |   |   +- nd_mid: :Test
#     |   |   +- nd_head:
#     |   |       (null node)
#     |   +- nd_super:
#     |   |   (null node)
#     |   +- nd_body:
#     |       @ NODE_SCOPE (line: 1, code_range: (1,0)-(1,40))
#     |       +- nd_tbl: (empty)
#     |       +- nd_args:
#     |       |   (null node)
#     |       +- nd_body:
#     |           @ NODE_BLOCK (line: 1, code_range: (1,10)-(1,35))
#     |           +- nd_head (1):
#     |           |   @ NODE_BEGIN (line: 1, code_range: (1,10)-(1,10))
#     |           |   +- nd_body:
#     |           |       (null node)
#     |           +- nd_head (2):
#     |               @ NODE_FCALL (line: 1, code_range: (1,12)-(1,35))
#     |               +- nd_mid: :private
#     |               +- nd_args:
#     |                   @ NODE_ARRAY (line: 1, code_range: (1,20)-(1,35))
#     |                   +- nd_alen: 1
#     |                   +- nd_head:
#     |                   |   @ NODE_DEFN (line: 1, code_range: (1,20)-(1,35))
#     |                   |   +- nd_mid: :a
#     |                   |   +- nd_defn:
#     |                   |       @ NODE_SCOPE (line: 1, code_range: (1,20)-(1,35))
#     |                   |       +- nd_tbl: (empty)
#     |                   |       +- nd_args:
#     |                   |       |   @ NODE_ARGS (line: 1, code_range: (1,25)-(1,25))
#     |                   |       |   +- nd_ainfo->pre_args_num: 0
#     |                   |       |   +- nd_ainfo->pre_init:
#     |                   |       |   |   (null node)
#     |                   |       |   +- nd_ainfo->post_args_num: 0
#     |                   |       |   +- nd_ainfo->post_init:
#     |                   |       |   |   (null node)
#     |                   |       |   +- nd_ainfo->first_post_arg: (null)
#     |                   |       |   +- nd_ainfo->rest_arg: (null)
#     |                   |       |   +- nd_ainfo->block_arg: (null)
#     |                   |       |   +- nd_ainfo->opt_args:
#     |                   |       |   |   (null node)
#     |                   |       |   +- nd_ainfo->kw_args:
#     |                   |       |   |   (null node)
#     |                   |       |   +- nd_ainfo->kw_rest_arg:
#     |                   |       |       (null node)
#     |                   |       +- nd_body:
#     |                   |           @ NODE_STR (line: 1, code_range: (1,27)-(1,30))
#     |                   |           +- nd_lit: "a"
#     |                   +- nd_next:
#     |                       (null node)
#     +- nd_compile_option:
#         +- coverage_enabled: false
```

关键地方在

```
...
#     |           +- nd_head (2):
#     |               @ NODE_FCALL (line: 1, code_range: (1,12)-(1,35))
#     |               +- nd_mid: :private
#     |               +- nd_args:
#     |                   @ NODE_ARRAY (line: 1, code_range: (1,20)-(1,35))
#     |                   +- nd_alen: 1
#     |                   +- nd_head:
#     |                   |   @ NODE_DEFN (line: 1, code_range: (1,20)-(1,35))
#     |                   |   +- nd_mid: :a
#     |                   |   +- nd_defn:
...
```

可见，这里的  `private`  被视作一个方法，而接收的参数的  `def`  定义的方法的返回。`def`  的返回是什么呢？是方法的  `Symbol`。也就是说：

```ruby
private def a
    'a'
end

# 等价于

tmp = def a
    'a'
end

tmp # => :a

private tmp
```

可见这个方法并不是真正所谓的内联，并没有什么  `private def`  的关键字，只是使用了一些 tricky 的方法使它看起来是内联的。那么这种写法有什么副作用？

让我们梦回一下 2013 年，看一眼 2013 年 Ruby 2.1 的发布[新闻](https://github.com/ruby/ruby/blob/v2_1_0/NEWS)。

```
def-expr now returns the symbol of its name instead of nil.
```

也就是说  `def`  表达式返回其名字的  `Symbol`  正是当时引入的特性。在这个特性的支持下，也就可以实现内联私有方法这种 trick 了。如果你的项目仍需要向下兼容到 Ruby 2.1 就不能使用这种方法了。

考虑到 Ruby 2.2 也已经 [EOL](https://www.ruby-lang.org/en/news/2018/06/20/support-of-ruby-2-2-has-ended/) 了，是否是用这种方法来实现私有方法已经完全变成了一种代码风格的取舍。如果你喜欢这种风格的私有方法，不妨在下一个项目里试一试吧。
