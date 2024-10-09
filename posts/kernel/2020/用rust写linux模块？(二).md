---
title: 用rust写linux模块？(二)
date: 2020-03-06 21:00:00
tags:
    - rust 
    - 内核
---

上一篇结尾讲到，重定位报错了。先重复一下上一篇提到的错误：

在换了个内核版本后得到的错误如下：

> [ 9839.177419] module: testmodule: Unknown rela relocation: 4

在尝试直接调用printk时得到的错误如下：

> [11061.846413] module: testmodule: Unknown rela relocation: 9

<!-- more -->

## 分析

首先得确认报错原因，dmesg里面给的信息看得云里雾里的，得去代码里面分析分析。把dmesg里面的关键字"Unknown rela relocation"拿去内核源码里面搜，看看是哪个函数在报错。

在linux源码目录（这里我用的源码版本是4.16.7）中执行`grep -rin "Unknown rela relocation"`，得到了两个我们想要的结果：

>arch/x86/kernel/machine_kexec_64.c:550:                  pr_err("Unknown rela relocation: %llu\n",
>arch/x86/kernel/module.c:205:                    pr_err("%s: Unknown rela relocation: %llu\n",

打开module.c，报错位于函数`apply_relocate_add`中，其中一段处理逻辑长这样：

```c
switch (ELF64_R_TYPE(rel[i].r_info)) {
		case R_X86_64_NONE:
			break;
		case R_X86_64_64:
			...
			break;
		case R_X86_64_32:
			...
			break;
		case R_X86_64_32S:
			...
			break;
		case R_X86_64_PC32:
		case R_X86_64_PLT32:
			...
			break;
		default:
			pr_err("%s: Unknown rela relocation: %llu\n",
			       me->name, ELF64_R_TYPE(rel[i].r_info));
			return -ENOEXEC;
		}
```

这段代码对不同的重定位方式进行了不同的处理，而我们得到的报错位于default的处理逻辑里面。回看上一章的两个报错，重定位方式的值分别为4和9。继续看内核代码，4和9对应的重定位类型如下：

```c
#define R_X86_64_PLT32		4	/* 32 bit PLT address */
#define R_X86_64_GOTPCREL	9	/* 32 bit signed pc relative offset to GOT */
```

关键字PLT和GOT。哦哦，也就是说，由于代码在编译时产生了使用PLT和GOT表的代码，但是内核在动态链接时并不支持这种方式。关于PLT和GOT，网上的资料一搜一大把，简单来说，GOT是用来实现地址无关代码(PIC)的关键技术，在编译时，那些要调用外部函数的代码被编译为去查GOT表，然后call从GOT表中得到的函数地址，GOT表由链接器进行修正。PLT作为GOT的优化，在装载时链接器并不立即修正这些地址，而是用PLT表来惰式修正，也就是在第一次call到相应函数时再来修正它的地址。

## 错误一的原因

现在来看第一个错误是怎么产生的。内核不识别R_X86_64_PLT32的重定位方式，但是仔细看代码，case语句中对于R_X86_64_PLT32是进行了错误处理的。emmmmm，错误一是在内核版本4.4上出现的，而我们看的代码是4.16的。去[lxr](http://lxr.free-electrons.com/)上翻一下4.4的代码，如下。

```c
switch (ELF64_R_TYPE(rel[i].r_info)) {
		case R_X86_64_NONE:
        	...
		case R_X86_64_64:
			...
		case R_X86_64_32:
			...
		case R_X86_64_32S:
			...
		case R_X86_64_PC32:
			...
		default:
			pr_err("%s: Unknown rela relocation: %llu\n",
			       me->name, ELF64_R_TYPE(rel[i].r_info));
			return -ENOEXEC;
		}
```

果然，4.4没处理R_X86_64_PLT32。唉，内核里面的东西太不稳定了，随时都有可能变。

## 错误二的原因

显然，错误二是因为内核不支持R_X86_64_GOTPCREL重定位方式。无论上面哪个版本，都不支持。

## 试试C++

在分析出错误原因以后，我尝试解决这个问题。只要想办法让rustc编译出不使用plt和got表的代码就行了，在翻阅了很多rust文档，我看到rustc的两个比较像的参数`code-model`和`relocation-model`（详细介绍点[这里](https://doc.rust-lang.org/rustc/codegen-options/index.html)），不幸的是，没有成功。

我打算用更熟悉的知识体系来解决这个问题，于是我写了一段和rust的hello函数很像的c++代码。

```c
// hello.cpp
extern "C" int printk(const char *args, ...);

extern "C" void hello(void)
{
    printk("hello world\n");
}
```

我把这段代码用以下命令编译打包成libhello.a，然后链接到模块代码中。

```shell
g++ -c hello.cpp
ar crs libhello.a hello.o
```

报错了，dmesg消息如下：

> [14837.023890] testmodule: Unknown symbol _GLOBAL_OFFSET_TABLE_ (err 0)

找不到GOT表的起始位置，在链接时没有这个符号。emmm，解决方法是在代码里面显式声明这个符号。如下：

```c
// hello.cpp
char GLOBAL_OFFSET_TABLE;
extern "C" int printk(const char *args, ...);

extern "C" void hello(void)
{
    printk("hello world\n");
}
```

重新编译并链接。在4.15内核版本上，insmod和rmmod都成功了。在4.4版本上，依然是plt表的重定位错误。

> module: testmodule: Unknown rela relocation: 4

唉，继续分析，到底是哪个函数在用plt重定位？

执行命令`readelf -r testmodule.ko | grep PLT`，得到以下结果：

> 000000000039  002200000004 R_X86_64_PLT32    0000000000000000 printk - 4

啊，原来是printk用了PLT重定位。翻GCC文档，怎么禁止PLT（文档链接在[这里](https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html)）。嗯，参数`-fno-plt`。试试用这个选项编译。下面是运行结果

```shell
root# g++ -c hello.cpp -fno-plt
root# readelf -r hello.o | grep PLT
root# readelf -r hello.o | grep printk
000000000012  000b00000029 R_X86_64_GOTPCREL 0000000000000000 printk - 4
```

没有生成用plt重定位代码，但是用的是GOT。。。链接到.ko里面，insmod果然不行。

注意到了GCC文档的开头有一句话，说大部分选项都可以加上`no-`来禁止某些功能。emmm，试试禁止PIC。

```shell
root# g++ -c hello.cpp -fno-pic
root# readelf -r hello.o | grep PLT
root# readelf -r hello.o | grep GOT
```

这次没有用任何的PLT和GOT功能，链接上insmod试试。继续报错。。。

>[16149.495657] module: overflow in relocation type 10 val ffffffffc04e4024
>[16149.495700] module: `testmodule' likely not compiled with -mcmodel=kernel

额，这段报错在哪里呢？翻内核源码之后，找到这段代码就在`apply_relocate_add`的尾部，距离最开始分析报错原因的位置不远。报错很明显，让加上-mmcmodel=kernel标志，为此，我又翻了下GCC文档，链接在[这里](https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html)，总之，先试试这个选项吧。这个选项要配合上-fno-pic一起使用，否则gcc会报错。

```shell
g++ -c hello.cpp -mcmodel=kernel -fno-pic
```

继续链接上看看行不行，当然，这里的module.c里面是直接调用hello，而不是printk(hello())这样的。

`insmod`，`rmmod`，然后`dmesg`，完美，打印出了"hello world"。

> [  277.693179] hello world

## 回到rust

通过gcc的一通操作，成功让c++代码链接上了，虽然这些代码和c基本上差不多。从上面的成功案例，我们得到一个知识点，那就是得要想办法不生成pic代码，也就是不要用plt和got。

rust怎么做才能得到这样的代码呢，至少用`code-model`和`relocation-model`这两个参数是不行的。要想得到解决方法，得追溯到rust代码是怎么生成的才行。于是我翻了rustc的文档，链接在[这里](https://rust-lang.github.io/rustc-guide/high-level-overview.html)。

总体来说，rustc编译rust代码分为以下几步：

- Parsing input，这一步生成语法树AST
- name resolution, macro expansion, and configuration，名字解析，宏展开，处理一些#[cfg]属性
- Lowering to HIR，生成HIR
- 类型检查之类的工作
- 生成MIR，这一步会进行一些高层次的优化
- 翻译成LLVM IR，然后进行LLVM的优化，并产生.o文件
- 链接

知道这些有什么用呢？我的想法是在中间进行截断，把后续工作交给gcc或者llvm，并告知它们不要进行pic处理。

于是，我又翻了文档，找到一个可用的选项`--emit`，链接在[这里](https://doc.rust-lang.org/rustc/command-line-arguments.html#--emit-specifies-the-types-of-output-files-to-generate)。这个选项可以决定rustc编译到哪一步，比如asm, link, mir，llvm-bc，llvm-ir等等。生成.o文件肯定是不行的，因为那时已经生成了用plt/got的代码，所以，生成asm汇编代码。

这次，不使用cargo来编译，而是直接上rustc。相当于c++里不用make，而是手动g++。为此，我们先修改一下rust代码：

```rust
// hello.rs
#![crate_type = "staticlib"]
extern "C" {
    fn printk(s: *const u8, ...) -> i32;
}

#[no_mangle]
pub extern "C" fn hello() {
    unsafe {
        printk("hello world!\n\0".as_ptr() as *const u8);
    }
}
```

接下来用rustc来编译，`rustc hello.rs --emit asm`

现在得到了一个hello.s，打开它，找到导致重定位错误的printk，如下，

```asm
callq   *printk@GOTPCREL(%rip)
```

手动把它改成

```asm
callq   printk
```

接下来，用gcc编译这个汇编文件。

```shell
gcc -c hello.s -mcmodel=kernel -fno-pic
```

然后打包为.a后链接到模块中。然后insmod、rmmod，成功，`dmesg`的结果如下：

> [ 2911.089624] hello world!

需要注意的是，一定要把printk的调用方式给修改了，不然编译出来的.o文件还是有PLT。

## 总结

要编译出能够成功链接到内核模块的代码，得使用`-mcmodel=kernel`和`-fno-pic`两个选项，而且，得保证编译出来的文件确实没有使用PLT/GOT才行。现在对于rust的做法有点原始，还得手动改生成的汇编文件。接下来的得研究一下现有的开源项目是怎么处理这个问题的。