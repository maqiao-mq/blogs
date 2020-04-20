---
title: 用rust写linux模块？(一)
date: 2020-03-06 19:30:00
tags:
---
​	最近研究了下rust，感觉要是能够用来写linux module，开发体验应该会好很多。这篇文章记录我尝试用rust开发linux模块踩坑的全过程（可能会弃坑）。如果同样有人想要用rust开发linux模块，我的文章应该没有什么帮助，建议直接使用github上已有的一些开源项目，点[这里](https://github.com/fishinabarrel/linux-kernel-module-rust)和[这里](https://github.com/tsgates/rust.ko)。

<!-- more -->
## 思路

以下是目前已知的一些知识点：

1. linux模块文件，也就是.ko文件，其实是一个特殊的由内核加载的重定位文件。所以，不论是什么语言，只要能够通过编译器生成这个.ko文件就可以了。
2. 在编译linux模块文件时，可以指定链接一些.a文件
3. rust与c语言进行交互，有两种方法，一种是c语言调rust回调，另一种是rust以FFI的方式调c函数。具体的方式点[这里](https://doc.rust-lang.org/nomicon/ffi.html)

综上，我的思路是用rust写一些核心的功能，然后编译为静态库.a文件，然后在编译linux模块时链接这个.a文件。

这么做的一个好处是比较好操控这些中间文件，毕竟我用c比用rust更熟；另一个是如果这样做确实可行，那么我可以写一些跑在用户态的中间库，让rust的代码链接到这些中间库，而不是内核的函数中，这样可以在用户态做一些核心功能的调试，减少重启系统的次数。

## 最简单的hello world尝试

我们让rust提供一个叫`hello`的函数，这个函数返回字符串"`hello world"`，然后用c语言来调用这个函数打印出字符串。

先来处理rust部分，首先`cargo new --lib hello`生成一个rust工程，然后修改lib.rs和Cargo.toml，代码分别如下。接下来`cargo build`生成libhello.a.

```rust
#[no_mangle]
pub extern "C" fn hello()->*const u8{
    "hello world\n\0".as_ptr() as *const u8
}
```

```toml
[package]
name = "hello"
version = "0.1.0"
authors = ["root"]
edition = "2018"

[dependencies]

[lib]
crate-type = ["staticlib"]
```

接下来处理c语言的部分，包括Kbuild，Makefile和module.c三个文件，整个项目的结构如下：

```shell
.
├── hello
│   ├── Cargo.lock
│   ├── Cargo.toml
│   └── src
│       └── lib.rs
├── Kbuild
├── Makefile
└── module.c
```

Kuild、Makefile和module.c三个文件的内容如下：

```makefile
MODULE_NAME=testmodule
OBJS := module.o
LIBS := hello/target/debug/libhello.a

ccflags-y := -Wall -g -DDEBUG
clean-files := ${OBJS}

obj-m = $(MODULE_NAME).o
$(MODULE_NAME)-y = $(OBJS) $(LIBS)
```

```makefile
KDIR = /lib/modules/`uname -r`/build

kbuild:
	make -C $(KDIR) M=`pwd`

clean:
	make -C $(KDIR) M=`pwd` clean
```

```c
#include <linux/module.h>

MODULE_DESCRIPTION("this is a test module");
MODULE_LICENSE("GPL");

extern const char * hello(void);

int driver_initialize(void)
{
    return 0;
}

void driver_uninitialize(void)
{
    printk(hello());
}

module_init(driver_initialize);
module_exit(driver_uninitialize);
```

编译后，`insmod`和`rmmod`都没有遇到问题，用`dmesg | tail`后看到最后一行有如下结果：

> [ 6872.348098] hello world

看似还行。

##  存在的问题

### 问题一

以上流程我是在ubuntu18.04，内核版本4.15.0-38-generic上跑的，没有遇到问题，但当我将同样的代码放到ubuntu18.04，内核版本4.4.0-87-generic的系统上跑时，就出现问题了。

在insmod时，程序报错如下：

> insmod: ERROR: could not insert module testmodule.ko: Invalid module format

dmesg看到的信息如下：

> [ 9839.177419] module: testmodule: Unknown rela relocation: 4

### 问题二

我们想让hello直接调用printk进行打印，而不是返回字符串，所以修改一下代码，lib.rs的代码如下：

```rust
extern "C"
{
    fn printk(s : *const u8, ...)->i32;
}

#[no_mangle]
pub extern "C" fn hello(){
    unsafe{
        printk("hello world\n\0".as_ptr() as *const u8);
    }
}
```

在module.c里面也要进行相应的修改：

```c
...
extern void hello(void);
...
void driver_uninitialize(void)
{
    hello();
}
```

make之后，insmod报错：

> insmod: ERROR: could not insert module testmodule.ko: Invalid module format

dmesg信息如下：

> [11061.846413] module: testmodule: Unknown rela relocation: 9

## 总结

显然，都是重定位出错了，内核在加载module的时候，不识别一些重定位方式。我在写这篇文章时，才刚让直接调用printk的hello跑起来，这里面的坑还挺大的，得单开一篇文章来讲讲这个事情，所以，后续解决方法见本系列第二篇。

