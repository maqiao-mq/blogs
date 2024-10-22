# ftrace

所有的内核probe机制，基本上都离不开ftrace，包括：kprobe、livepatch、fprobe、tracepoint等等。

ftrace的前提要求已经很多教程在讲了，简单来说就是依赖gcc的mcount机制，也就是在每个函数的开头加上若干个nop，在需要trace的时候把这几个nop替换成jmp之类的指令。

ftrace的代码看起来还是很难懂的，涉及到的东西太多了。所以这里不会先深入到细节里面，而是先记录下它的设计原则，然后再来看代码。

设计原则：

1. 一张全局表来记录每个ftrace 点的情况，比如有没有被hook，被怎么hook了
2. 每个ftrace ops，可以同时hook到多个函数上的
3. 每个ftrace的点，可以只生效一个ftrace ops，也可以同时生效好几个ftrace ops，但是为了让这几个同时生效，ftrace做了不少的事情。

下面来展开细说一下：

## 全局表

这张全局表从 ftrace_pages_start 开始，它的结构是单链表，链表的每项是一个从 buddy 分出来页，页的 order 可能是0，也可能是1，也可能是 2，不确定。每一页里面有若干个 record，一个 record 对应一个 ftrace 点，所有的 ftrace 点就以这样的形式组织起来。页的 order 是根据 record 的数量来动态分配的，反正一直尝试分配，直到内存分够为止。此外，模块加载的时候，它的 ftrace 点的 record 也会分出来。

```
struct ftrace_page {
	struct ftrace_page	*next;
	struct dyn_ftrace	*records;
	int			index;
	int			order;
};

```

代码里面一般用这个方式遍历所有的 ftrace 点：

```
do_for_each_ftrace_rec(pg, rec) {
...
} while_for_each_ftrace_rec();
```

## ftrace 点

每个 ftrace 点会记录自己有没有被 hook，被怎么 hook 的。

```
struct dyn_ftrace {
	unsigned long		ip; /* address of mcount call-site */
	unsigned long		flags;
	struct dyn_arch_ftrace	arch;
};
```

重点可以关注这么几类：

1. FTRACE_FL_TRAMP，当只有**一个 ftrace ops **在这个点上生效时，x86架构会为这个创建一个 trampoline，用来加速 ftrace 函数的调用速度
2. FTRACE_FL_IPMODIFY，当一个 ftrace ops 会改变函数返回的 ip，从而使得原函数不再被执行的时候，会标记这个 flag。livepatch 就会用这个特性。
3. FTRACE_FL_DIRECT，表示 ftrace 这里会直接调对应的函数，当前只被 bpf trampoline 使用。

关于 ftrace 点的各种切换，可以看这个函数 `__ftrace_hash_rec_update`

有些注意点：

- bpf trampoline 和 livepatch 有一段时间是不兼容的，为了让二者能同时在一个 record 生效，upstream 又引入了 ftrace_ops->ops_func() 来解决
- ftrace trampoline 和 bpf trampoline 不是同一个东西，而且 ftrace trampoline 引入的时间很早，但是它只会在 record 只有一个hook 的时候生效，break 这个限制的时候，将会退化成调用 `ftrace_ops_list_func`，它会遍历每个 ftrace_ops，找到匹配当前 ip 的哪些 ops，然后调用一遍回调，比较慢。
- 为了 `ftrace_ops_list_func`能遍历所有的 ftrace_ops，ftrace 用 `ftrace_ops_list` 这个链表头把所有的 ftrace ops 都串起来了。

## ftrace_ops

这个表示一个 hook 函数，但是一个 hook 函数可以同时在多个 record 上生效，而且可以在部分 record 上生效，部分不生效。

所以，每个 ftrace_ops 都会有一个自己的哈希表来记录自己在哪些 record 上生效。

函数 `ftrace_set_filter_ip`就是用来配置 ftrace 在哪些 record 上生效的，而函数 `register_ftrace_function`则是用来将 ftrace_ops 挂载到对应的 record 上，使其正式生效的。

函数 `ftrace_set_notrace`则是配置 ftrace ops 在哪些 record 暂时不生效。

为此，每个 ftrace ops 有2个哈希表：

* filter_hash 用来配置这个 ftrace ops 关联了哪些 record
* notrace_hash 用来配置这个 ftrace ops 在哪些 record 上生效

# kprobe

# uprobe

# fprobe

# eprobe
