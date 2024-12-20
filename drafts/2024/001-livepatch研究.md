---
title: livepatch研究
date: 2024-10-09 22:00:00
tags:
    - 内核
---
# 写在前面

这篇文档用于记录自己对 livepatch 机制的各种研究，不一定站在小白的角度来详细叙述。

# 总览

livepatch 的整个机制分成两块：

- 内核侧，负责将以 module 形式组织的 livepatch hook到内核对应的函数上。这块的代码位于 `kernel/livepatch`。
- 用户态侧，负责根据 patch 的代码构建 livepatch 的 module。这块的代码位于[github](https://github.com/dynup/kpatch)上

# 内核侧

内核侧的很多细节，可以看[内核文档](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/livepatch/livepatch.rst)

## overview

内核侧的示例代码位于 `sample/livepatch` 中，这里贴一个示例上来：

```c
// SPDX-License-Identifier: GPL-2.0-or-later
/*
 * livepatch-sample.c - Kernel Live Patching Sample Module
 *
 * Copyright (C) 2014 Seth Jennings <sjenning@redhat.com>
 */

#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/livepatch.h>

/*
 * This (dumb) live patch overrides the function that prints the
 * kernel boot cmdline when /proc/cmdline is read.
 *
 * Example:
 *
 * $ cat /proc/cmdline
 * <your cmdline>
 *
 * $ insmod livepatch-sample.ko
 * $ cat /proc/cmdline
 * this has been live patched
 *
 * $ echo 0 > /sys/kernel/livepatch/livepatch_sample/enabled
 * $ cat /proc/cmdline
 * <your cmdline>
 */

#include <linux/seq_file.h>
static int livepatch_cmdline_proc_show(struct seq_file *m, void *v)
{
	seq_printf(m, "%s\n", "this has been live patched");
	return 0;
}

static struct klp_func funcs[] = {
	{
		.old_name = "cmdline_proc_show",
		.new_func = livepatch_cmdline_proc_show,
	}, { }
};

static struct klp_object objs[] = {
	{
		/* name being NULL means vmlinux */
		.funcs = funcs,
	}, { }
};

static struct klp_patch patch = {
	.mod = THIS_MODULE,
	.objs = objs,
};

static int livepatch_init(void)
{
	return klp_enable_patch(&patch);
}

static void livepatch_exit(void)
{
}

module_init(livepatch_init);
module_exit(livepatch_exit);
MODULE_LICENSE("GPL");
MODULE_INFO(livepatch, "Y");
```

livepatch的模块是比较特殊的，它有以下几个特点：

1. module_info 里面会标记它是 livepatch，以便内核对它做一些特殊处理
2. 整个api分成3层结构：
   a. `struct klp_patch`，这个表示一个patch
   b. `struct klp_object`，这个表示一个编译单元，所以是object
   c. `struct klp_func`，这个表示一个编译单元里面的某个函数的修改

## 特殊的链接

正常来说，一个 module 只能使用 EXPORT_SYMBOL 一类的宏导出的函数。但是，对于 livepatch 来说，这就很强人所难了。因为它是要去 hook 某个函数，如果那个函数原来就调用了某些 static 或者未导出的函数，那这个 livepatch 里面的新函数也是会去调的，所以它一定要去链接某些未导出的函数。

为了解决这个问题，livepatch 的 module 引入了特殊的 section，用于在 load module 的时候做重定位。
这个section一般是 `.klp.rela.vmlinux.*` 或者 `.klp.rela.{module}.*`

## hook 的机制

底层还是调用的 ftrace 的接口来进行 hook 的，本质上还是利用 ftrace 在函数开头预留的 5 个 nop。
livepatch 在 ftrace 机制里面注册的回调函数叫做 klp_ftrace_handler ，这个函数的一个很核心的动作，是把 pt_regs 里面的 ip 给换成新的函数的，从而达到一个替换的目的。

```c
static void notrace klp_ftrace_handler(unsigned long ip,
				       unsigned long parent_ip,
				       struct ftrace_ops *fops,
				       struct ftrace_regs *fregs)
{
    ...
    ftrace_regs_set_instruction_pointer(fregs, (unsigned long)func->new_func);
    ...
}
```

## hook 的时机

一般 livepatch 都会 hook 好几个地方，而不止一个函数。因为改一处代码，可能这个函数的优化版本和非优化版本都要改，也可能因为这个函数是 inline 的，导致所有用到它的地方都要改。
那么，在 hook 的时候，一定要一起改掉，这就是所谓的一致性模型。

所以，内核文档给出了确保一致性的几个办法：

1. 如果能确保当前 arch 能够获得精确的栈回溯（HAVE_RELIABLE_STACKTRACE），那么会去检查所有处于 sleep 状态的进程的栈有没有用到被 hook 的函数。一般是第一次尝试 hook 的时候会去检查，如果用到了的话，就会放弃 hook，并进行周期性检查。注意，**内核并不会用stop_machine的方式来停掉机器来做检查，而是无限通过work来尝试**，碰运气，每次work都执行去做一遍检查，哪次成功了就哪次打上去。
2. 如果需要的话，把所有的进程打出内核态，也就是说，让他们都回用户态那边，那自然就不会用到内核函数了。IO密集型的进程给它发信号（SIGSTOP and SIGCONT），cpu密集型的等一个中断。
3. swapper线程，也就是那些一直处于内核态的，在 cpu 处于 idle 的时候会用到的进程，他们是不会到用户态的。这些线程会显式感知要打 livepatch 的状态来配合。`klp_update_patch_state()`

## 对模块函数的hook

有一个容易想到的问题是，如果我的 livepatch 要修改的是一个 module，而在我加载 livepatch 到内核中的时候，这个 module 还没有被加载上去，那我的 livepatch 岂不是失效了？
这个问题其实早就被想到了。
目前的机制是这样的：

1. livepatch 加载到内核以后，会先挂到一个全局的链表上。
   `LIST_HEAD(klp_patches);`
2. livepatch 里面的新函数，会尽可能地被打到对应的原函数上，如果原函数是位于 vmlinux 或者 已加载的 module 里面的话。
3. 在其他的 module 被加载的时候，会调用到 `klp_module_coming`函数。这个时候会挨个检查 livepatch 要 hook 的 module 和正在被加载的 module 是不是对得上的，如果对得上，就会把这个模块的函数给 hook 掉。

## 内核接口

每个livepatch的信息都会出现在这里：`/sys/kernel/livepatch`

## 其他问题待确认

livepatch 和 module 的对应关系？有没有MD5之类的校验

# 用户态

用户态做的事情很杂，主要的目的是捏一个内核认识的，以 elf 格式组织的 livepatch 的 ko 出来。
由于我们的输入只有一个文本形式的patch，它要把它变成 elf binary，所以里面要做的事情就多了。

相关的代码位于[github的kpatch项目](https://github.com/dynup/kpatch)中。

整个livepatch module的构建过程，项目的 README 已经写的很清楚了。

```
- Build the unstripped vmlinux for the kernel
- Patch the source tree
- Rebuild vmlinux and monitor which objects are being rebuilt. These are the "changed objects".
- Recompile each changed object with -ffunction-sections -fdata-sections, resulting in the changed patched objects
- Unpatch the source tree
- Recompile each changed object with -ffunction-sections -fdata-sections, resulting in the changed original objects
- For every changed object, use create-diff-object to do the following:
  - Analyze each original/patched object pair for patchability
  - Add .kpatch.funcs and .rela.kpatch.funcs sections to the output object. The kpatch core module uses this to determine the list of functions that need to be redirected using ftrace.
  - Add .kpatch.dynrelas and .rela.kpatch.dynrelas sections to the output object. This will be used to resolve references to non-included local and non-exported global symbols. These relocations will be resolved by the kpatch core module.
  - Generate the resulting output object containing the new and modified sections
- Link all the output objects into a cumulative object
- Generate the patch module
```

## 怎么检查哪些 object 发生了变化

先构建一遍内核，然后打上 patch 重新构建一遍，这个时候 make 就会根据源码文件的变化，自动将部分 .o 文件进行重编。
问题的关键是怎么感知到这部分重编的文件，kpatch的解法是搞一个脚本来替换 gcc。
首先，在 make 的时候指定 kpatch-cc,用它来替代 gcc：

```shell
KPATCH_CC_PREFIX="$TOOLSDIR/kpatch-cc "
declare -a MAKEVARS
if [[ -n "$CONFIG_CC_IS_CLANG" ]]; then
# 这里指定 CC 为 kpatch-cc
	MAKEVARS+=("CC=${KPATCH_CC_PREFIX}${CLANG}")
	MAKEVARS+=("HOSTCC=clang")
else
	MAKEVARS+=("CC=${KPATCH_CC_PREFIX}${GCC}")
fi

if [[ -n "$CONFIG_LD_IS_LLD" ]]; then
	MAKEVARS+=("LD=${KPATCH_CC_PREFIX}${LLD}")
	MAKEVARS+=("HOSTLD=ld.lld")
else
	MAKEVARS+=("LD=${KPATCH_CC_PREFIX}${LD}")
fi


# $TARGETS used as list, no quotes.
# shellcheck disable=SC2086
make "${MAKEVARS[@]}" "-j$CPUS" $TARGETS 2>&1 | logger || die
```

然后，在 kpatch-cc 里面，先用 gcc 来编译下 .o 文件，然后开始记录这个 .o 文件被重编了

```shell
...

TOOLCHAINCMD="$1"
shift
# 这里执行 gcc 构建 .o 文件
if [[ -z "$KPATCH_GCC_TEMPDIR" ]]; then
	exec "$TOOLCHAINCMD" "$@"
fi

...

    *.o)
        mkdir -p "$KPATCH_GCC_TEMPDIR/orig/$(dirname "$relobj")"
        [[ -e "$obj" ]] && cp -f "$obj" "$KPATCH_GCC_TEMPDIR/orig/$relobj"
        # 这里记录 .o 文件已经被重编了
        echo "$relobj" >> "$KPATCH_GCC_TEMPDIR/changed_objs"
        break
        ;;
...
```

## 怎么 diff 出两个 object 的哪些函数发生了变化

在编译的时候，会使用 `-ffunction-sections -fdata-sections` 这个选项，这个选项会把每个函数、每个变量放到一个独立的 section 中。
那么，识别函数的变化，就变成了对比 section 是否有差异了，而对比 section 只需要简单的 memcmp 就可以了。
这块可以在 `create-diff-object.c` 的 `kpatch_compare_correlated_section()` 和 `kpatch_compare_correlated_nonrela_section()` 中可以看到。

```c
static void kpatch_compare_correlated_nonrela_section(struct section *sec)
{
	struct section *sec1 = sec, *sec2 = sec->twin;

	if (sec1->sh.sh_type != SHT_NOBITS &&
	    memcmp(sec1->data->d_buf, sec2->data->d_buf, sec1->data->d_size))
		sec->status = CHANGED;
	else
		sec->status = SAME;
}
```

## 怎么在抽取变化函数的时候，把它涉及的变量等一起抽取出来

TODO

# livepatch 的限制

从上面的分析中，可以看到，livepatch 只是 hook 到原始函数，然后让代码的分支走到新的函数上，而且老函数并不会被删除，原有的 text 段并不会受到影响。
因此，常见的一些限制就可以被理解了，比如：

1. 不能改变量
2. 不能往结构体里面加变量
3. 不能 hook .__init 段里面的函数（因为已经被执行过了，而且大概率不会再被执行）
   ...
