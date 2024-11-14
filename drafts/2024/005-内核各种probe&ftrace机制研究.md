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


## ftrace的生效、修改、执行

这里记录的先不考虑ftrace trampoline的情况（这个情况目前只在x86上有，是一种优化措施)。

这里先说下执行的逻辑：

在x86上，如果没有使用ftrace trampoline的话，那么全局所有的record都只会共享同两个handler。这意味着不管是哪个函数的ftrace，生效后入口都是一样的。这个入口是汇编写的，在 `arch/x86/kernel/ftrace_64.S`里面。如果ftrace在处理的时候需要原函数的regs，则handler是 `ftrace_regs_caller `，否则函数是 `ftrace_caller`。

这两个handler都会做一些准备工作，然后调用到真正的handler里面去。这个真正的handler也会根据ftrace的情况发生变化：

- 一般都是 `ftrace_ops_list_func`，这个函数会遍历当前生效的所有ftrace_ops，判断当前ftrace的入口ip是否在ftrace_ops的filter_hash表中，如果是的话就调用这个ftrace_ops的回调。
- 有些优化场景（比如仅一个ftrace生效），这个handler会直接是ftrace_ops的回调
- 有时会是 `ftrace_ops_assist_func`，具体情况还未分析
- 在关闭ftrace的时候，这个函数会是ftrace_stub

这个真正的handler，需要替换到 `ftrace_regs_caller `和 `ftrace_caller`函数的中间位置去，所以内核提前把这两个位子给用symbol标记出来了，分别是 `ftrace_call`和 `ftrace_regs_call`。这个替换的过程，`ftrace_update_ftrace_func()`


生效是这样的流程：

1. 先调用 `ftrace_set_filter_ip` 来给ftrace_ops配置要在哪个record上生效
2. 调用 `register_ftrace_function`来让ftrace_ops生效。

`register_ftrace_function`往下会调用到 `ftrace_startup`，这里是真正让 ftrace 生效的代码。这里的代码涉及到的东西很多，这里只记录一个简化版的：

- 调用 `__register_ftrace_function`将这个ftrace挂到 `ftrace_ops_list`这个全局链表上
- 调用 `ftrace_hash_ipmodify_enable`，遍历所有record，如果record的ip和这个ftrace要生效的ip对上了，而且这个ftrace是要改返回ip的，那么就要标记这个record为 `FTRACE_FL_IPMODIFY`，方便后续做一些特殊处理。同时，还需要调用 `ops->ops_func`来处理下自己的一些代码，避免livepatch和bpf trampoline这种ip modify 和 direct call 不兼容的情况。
- 调用 `ftrace_hash_rec_enable`，再次遍历所有record，如果record的ip和这个ftrace要生效的ip对上了，则根据这个record上的情况来判断这个record对应的位置，后续是不是要更新，应该怎么更新，置好对应的flags
- 调用 `ftrace_startup_enable`来正式做更新。这个实际会调用 `arch_ftrace_update_code`，让不同的平台来自己处理。
  - 这里会先更新 `ftrace_regs_caller `和 `ftrace_caller`函数里面的 `ftrace_call`和 `ftrace_regs_call`，见上文
  - 然后，会调用 `ftrace_replace_code()`来真正更新。这里的逻辑是：
    - 先过一遍所有的record，看下这个即将要更新的动作有没有bug
    - 再过一遍所有的record，把所有要更新的地址，要更新的指令记录到text_poke的缓冲区中
    - 调用 `text_poke_finish`一次性将所有的指令都刷上去。


这里再记录下 `text_poke_finish`的过程。这里并没有做stop_machine的动作，而是用int3指令巧妙地绕开了。

具体来说，是这样的：

- 先把所有要更新的地址，都替换成int3，因为int3只有一个字节，所以更新是原子的，安全的
- 把int3后面的几个字节的部分替换成新的指令
- 最后一把把所有的int3再换成新的指令，这样新的指令就被完全被更新上去了

期间，如果踩到了更新到一半的指令，也最多是触发int3，然后只需要在int3里面处理下就行了。


注意，ftrace的生效其实会换两个地方的指令：

- 第一个地方是 `ftrace_regs_caller `和 `ftrace_caller`函数里面的 `ftrace_call`和 `ftrace_regs_call`，这个地方的原理和上文中的也类似，只不过是单条指令的更新，而不是批量的更新
- 第二个地方是上文说到的所有record的地址的更新，这里是批量的

# kprobe

kprobe 根据实际情况分为几种：

1. 如果probe的ip是ftrace的那几个nop，则直接调用ftrace的更新机制
2. 否则，要单独处理。

kprobe提供了两个handler，pre_handler和post_handler，含义是在probe的那条指令执行前和执行后分别执行。

//TODO

# uprobe

# fprobe

# eprobe
