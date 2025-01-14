# psi

## 字段含义解释

以下内容来源于代码注释。

系统的cpu、io、memory资源都是有限的，所以进程之间必定会存在等待这些资源的情况，也就是stall。
pressure是想看系统的整体负载压力情况。

### 单个cpu

先考虑单个cpu的情况：

单个cpu上，有多个进程，但是有些进程会时不时因为io带宽瓶颈而stall。我们想看系统的io压力，应该怎么做呢？

方法是，计算一段时间内，**存在stall状态进程时cpu运行的时间，占整体时间的占比**。这个占比越高，说明压力越大。这个是最基本的思想。这个我们称为some。

有了这个还不满足，因为有可能所有的进程都会因为内存不足而stall，这个时候系统会陷入空转，对大家都不好。那怎么才能把这个指标抓出来，看系统的空转程度呢？

方法是，计算一段时间内，**存在stall状态进程，且没有任何一个可在cpu上运行的进程（idle进程除外）的时间，占整体时间的占比**。这个占比越大，那么压力就越大。这个称为full。

按照这个思路捏造了个例子：

有进程A、B。

* 在t1时刻，A、B都是ready的，但是A在用cpu
* t2时刻，A stall，换B运行
* t3时刻，B stall，内核接管，换上idle进程
* t4时刻，io回来了，换A回来继续运行

那么统计占比，整体时间是t4-t1。

some的时间是，（t3-t2）+（t4-t3）。前者是A stall带来的，后者是A、B同时stall带来的。

full的时间是B的运行时间（t4-t3)。

按照这个思路，对some和full的定义是：

```
 *	SOME = nr_delayed_tasks != 0
 *	FULL = nr_delayed_tasks != 0 && nr_running_tasks == 0

```

而统计some和full的比例是这样的：

```
 *	%SOME = time(SOME) / period
 *	%FULL = time(FULL) / period
```

脑子里要有个概念，就是时间会一直往前走，但是stall进程的数量、running进程的数量会一直变化，所以这个time()是对处于这类状态的时间的累加。

至于怎么实现，那是后面的事情，先以连续的思维来看，不要考虑时间分片什么的。

### 多个cpu

多个cpu的时候，上面的算法会有缺陷。比如：

1. 257个进程想要256个cpu，不管什么时候，都会有一个进程在等待。按照上面的算法，some的比例会是100%。但如果把256个cpu的时间都拆开单算的话，其实只有一个cpu处于some定义的这个状态，因此比例应该是1/256=0.4%
2. 系统中有4个进程和4个cpu，但是一个进程在等待内存资源。按照定义，系统里面是没有full状态的，所以full的比例是0%。但实际上，有1个cpu处于空转但是有进程stall的状态，所以比例是1/4=25%

因此，在多cpu的情况下，要把非idle的进程数量，cpu数量一起考虑进来。

```
 *	threads = min(nr_nonidle_tasks, nr_cpus)
 *	   SOME = min(nr_delayed_tasks / threads, 1)
 *	   FULL = (threads - min(nr_running_tasks, threads)) / threads
```

- threads的计算逻辑是，如果在进程数量很多的情况下，我们最多能统计到nr_cpus个cpu的时间；如果在进程数量很少的情况下，我们只统计这几个进程使用的cpu时间即可，不然分母不能正确反应总使用时间。
- some的计算逻辑是，如果stall的进程数量实在是太多了，比如超过了nr_cpus(不可能超过nr_nonidle_tasks)，那么我们认为负载拉满了，记成1就行，不用再突破到100%以上。
- full的计算逻辑，带入不同参数自行体会。

### 实现

内核只会在进程切换上下文的时候来统计这个信息。每个cpu一个runqueue，所以是以runqueue为单位，在做各种进程切换的时候来统计的。

```
 * For each runqueue, we track:
 *
 *	   tSOME[cpu] = time(nr_delayed_tasks[cpu] != 0)
 *	   tFULL[cpu] = time(nr_delayed_tasks[cpu] && !nr_running_tasks[cpu])
 *	tNONIDLE[cpu] = time(nr_nonidle_tasks[cpu] != 0)
 *
 * and then periodically aggregate:
 *
 *	tNONIDLE = sum(tNONIDLE[i])
 *
 *	   tSOME = sum(tSOME[i] * tNONIDLE[i]) / tNONIDLE
 *	   tFULL = sum(tFULL[i] * tNONIDLE[i]) / tNONIDLE
 *
 *	   %SOME = tSOME / period
 *	   %FULL = tFULL / period
```

注意这里面的逻辑，把纯纯的idle给排除掉了，也就是那种cpu在idle里面空转，而且没有任何stall进程的情况。这种情况下，其实系统是没有压力的。

## 内容解析

用法是要先在cmdline里面配置 `psi=1`，然后才能在 `/proc/pressure/`下面看到io、memory和cpu的压力。

此外，还可以在 `/sys/fs/cgroup/`下看到cgroup的情况，文件分别为 `xx.pressure`，比如 `io.pressure`

cat看系统的整体负载。文件内容一般是：

```
some avg10=xx avg60=xx avg300=xx total=xxx
full avg10=xx avg60=xx avg300=xx total=xxx
```

- avg10、avg60、avg300，表示最近10秒、60秒、300秒时间内的负载情况。total则是这些时间加和的绝对值。
- some和full在上文中已经有解释。

有一个用法是用epoll来监听系统负载有没有超过某个值，超过了以后就会触发事件。

方法是先打开这几个文件之一，往里面写入检测的阈值：

```
<some|full> <stall amount in us> <time window in us>
```

然后用epoll监听fd，等待事件返回。


# signal

## sigframe

我们在使用signal的时候，看似只用了这么一个syscall，但实际上glibc用了两个syscall。

- signal函数是调用rt_sigaction这个syscall来传入参数的。
- 在信号处理函数执行完毕后，用rt_sigreturn来返回

```
SYSCALL_DEFINE4(rt_sigaction, int, sig,
		const struct sigaction __user *, act,
		struct sigaction __user *, oact,
		size_t, sigsetsize)
```

- sig指信号的编号
- act是新传入的用于处理信号的结构体，oact是用来接上一个
- sigsetsize用来看传入的sigset_t是否和内核的对得上

x86上sigaction定义如下：

```
struct sigaction {
	__sighandler_t sa_handler;
	unsigned long sa_flags;
	__sigrestore_t sa_restorer;
	sigset_t sa_mask;		/* mask last for extensibility */
};
```

这里面有两个回调：

- sa_handler，用来处理信号的函数，这个就是调用signal()时传入的handler
- sa_restorer，用来在处理信号后返回的函数，一般就是直接调rt_sigaction



单纯来看，还是很难理解为什么会有rt_sigaction。所以需要深入细节。

- 首先，rt_sigaction实际上只是在内核里面填下表，记录信号发生的时候应该用什么handler来处理，其他事情一概不做
- 接下来，由其他进程或者内核向这个进程发送信号，一般是调kill syscall
- 现在，进程响应信号。这里面也很复杂了，如果进程在运行，那就要把它打出来；如果在睡觉，那就要把它拉到rq里面，一堆需要处理的事情。总之，最终进程会拿到cpu，然后对信号进行处理
- 信号一定是在用户态处理的。所以，当进程一路从内核返回到用户态的时候，最后会检查是不是有信号pending，有的话就要准备处理了。这个在 `exit_to_user_mode_loop()`中。

对于x86来说，它要做的事情，是准备sigframe，好让进程回到用户态的时候直接执行signal handler，而不是沿着原来的路径继续运行。这个在 `setup_rt_frame()`中。

它的frame大概长这样：

```
 * -------------------------
 * | alignment padding     |
 * -------------------------
 * | (f)xsave frame        |
 * -------------------------
 * | fsave header          |
 * -------------------------
 * | alignment padding     |
 * -------------------------
 * | siginfo + ucontext    |
 * -------------------------
```

其中xsave和fsave是用来存x86的特殊寄存器的。siginfo和ucontext的信息见下面的结构体定义：

```
struct rt_sigframe {
	char __user *pretcode;
	struct ucontext uc;
	struct siginfo info;
	/* fp state follows here */
};

struct ucontext {
	unsigned long	  uc_flags;
	struct ucontext  *uc_link;
	stack_t		  uc_stack;
	struct sigcontext uc_mcontext;
	sigset_t	  uc_sigmask;	/* mask last for extensibility */
};

typedef struct siginfo {
	union {
		__SIGINFO;
		int _si_pad[SI_MAX_SIZE/sizeof(int)];
	};
} __ARCH_SI_ATTRIBUTES siginfo_t;
```

这里面，ucontext和siginfo都不重要，重要的是有个额外的字段——pretcode，这个字段是一个指针，放的是rt_sigaction传进来的restorer的值。

在sigframe做好以后，内核的下一步动作，是调整用户态的sp和ip寄存器，分别执行sigframe和handler。那么，这样返回的时候就会直接到handler上运行了。

回忆一下，x86上栈是从高地址到低地址的，而pretcode是在rt_sigframe的最低位置，所以它实际上是在最低的位置上。再仔细往下想，handler执行完毕后，调用ret指令，会从栈顶弹一个地址作为返回地址，恰好这个地址就是pretcode的值。而pretcode存放的是restorer的地址，在glibc上就是rt_sigreturn。

所以，整体的流程是：

1. 在进程退出到用户态的时候，构建sigframe
2. 调整ip为handler
3. handler执行完毕后，ret指令走到rt_sigreturn
4. rt_sigreturn清理掉原来的sigframe，复原之前的寄存器，程序继续正常往下执行
