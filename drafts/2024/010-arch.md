# idle

链接：https://www.kernel.org/doc/html/v5.4/admin-guide/pm/cpuidle.html

每个cpu有一个idle task，这个idle task 处于最低优先级，当 cpu 上的进程调无可调的时候，就会走到idle task，然后做一些idle 相关的事情，比如降频、挂起之类的，然后等着被唤醒。

idle loop主要有两个动作：

1. 调governor模块来判定当前cpu应该进入哪个idle state
2. 调driver让cpu陷入这个idle state

在CPUIdle子系统初始化的时候，所有可用的idle state就被初始化放到了一个一维数组上了。每个idle state会有两个参数：

- *target residency*，表示进入这个idle state需要的时间。一般来说，进入的状态越深，就需要更长的时间
- exit latency，从这个idle state退出最长需要的时间



governor在决策cpu进入哪个idle state的时候，主要依赖两个信息：

- 下一次定时器事件（timer event）的触发时间。这个是精确可控的时间，可以用来和idle state里面给的target residency（即想要idle的时间）来比较，决定要不要进入idle
- idle duration。从最近一次idle state切换出来，并不一定是定时器触发的，也可能是其他不确定事件触发的。这种中途被打断，实际 idle 的时间称为idle duration，它也要用来辅助决策。

但是governor实际怎么根据这两个信息做决策，还是取决于实际算法。

目前有4中governor：menu、TEO、ladder和haltpoll。一般不建议在生产环境切换governor，默认配置取决于内核编译的时候的配置，以及一些运行时对cpu能力的判断。可以在 `/sys/devices/system/cpu/cpuidle`中查看 `current_governor_ro`，以确定当前用的是哪个governor。

- tickless的系统（见后文）上一般配置的是menu。从字面意义上来看，你可以理解成硬件提供了好几种不同层次的idle能力，由内核根据这次想要idle的时间，历史上idle的时间等来综合判断应该走哪个idle能力，相当于让内核点菜(menu)。
- TEO和ladder一般都不太关注，很多发行版甚至连他们的config都没开。ladder应该是指内核逐级往更深的idle state里面走。
- haltpoll在服务器场景是个很重要的能力，只在虚拟机里面生效。它的背景是，虚拟机如果要进idle state，通常都需要vm exit，下次来事件了又得vm enter进去，这一来一回非常耗时，已经超过了idle带来的收益了（本质上还是guest os无法正确算出进入idle state的代价，毕竟被虚拟化隔开了一层）。那么，为了避免无意义的，频繁地vm exit，guest os使用poll代替halt以避免真正进入idle state，反而可以提升性能。

driver主要看cpu platform的能力了，内核在构建的时候一般会编译多个driver的代码，但是这些driver在启动的时候会自己根据情况判断是否生效。一般在系统启动的时候就会确定好driver，然后后续就无法更改了。在 `/sys/devices/system/cpu/cpuidle/`里面可以查看 `current_driver`获取这个信息。

- intel上，会提供intel_idle和acpi_idle两个driver
- 在amd上，只有acpi_idle这个driver
- 不管在哪个cpu平台上，haltpoll driver都是有的。haltpoll driver和haltpoll governor严格绑定，它们不会兼容其他的driver和governor。

## scheduler tick和idle

内核是有一个固定周期的scheduler tick的，用来周期性强制将当前进程换出去，换新的进程来运行，保障调度的公平性。

但是，在idle loop的情况下，其实没必要周期性地tick来打扰idle状态。如果要内核不管在什么情况下都要tick的话，可以在编译的时候关掉CONFIG_NO_HZ_IDLE，或者启动参数里面加上 `nohz=off`。

如果在idle的时候默认会停止tick，则系统被称为tickless的。tickless的情况下，governor默认使用menu；非tickless的情况下，governor默认使用ladder。


## 控制idle行为的cmdline

公用：

`cpuidle.off=1`，这个不会禁止cpu运行idle loop，但是会禁止governor和driver的生效

`cpuidle.governor= `，这个用来强行指定governor。

`nohlt/hlt`，会强行进入idle poll的模式，但是不会禁用governor和driver的生效

x86特有：

`idle=poll, idle=halt, idle=nomwait`。

- 前两个都会关掉acpi_idle和intel_idle这两个driver，halt模式会使用HLT指令来直接让cpu停下来，poll模式会让cpu一直循环。poll会阻止cpu进入P-state，但是它可能会影响隔壁超线程的性能。
- nomwait会关掉intel_idle，使用acpi_idle。而且，它会禁止cpu使用mwait指令来要求硬件进入idle状态。
- `intel_idle.max_cstate=<n>`，它限制intel_idle最大进入哪个idle state。
- `processor.max_cstate=<n>`，它限制acpi_idle最大进入哪个idle state。
- `intel_idle.max_cstate=0`，这个等效于关掉intel_idle，使用acpi_idle
- `processor.max_cstate=0`等效于 `processor.max_cstate=1`（退无可退了，只能这么处理）

在amd上，因为没有intel_idle这个模块，所以一般是使用 `processor.max_cstate=0`来控制。


//todo： 虚拟化下的idle优化：https://docs.kernel.org/virt/kvm/halt-polling.html

//todo：cpu的idle和外设的idle区分

## 代码分析

### idle loop

内核在cpu启动的时候，会构造一个kthread来完成各种操作。

这个kthread先会调用 `init_idle() `对自己完成初始化，然后在后续的初始化中，它最终走到 `cpu_startup_entry()`，在这里执行完各种操作后，会进入 `do_idle()`的无限循环。

- `init_idle()`会将这个kthread自身变为一个完整的kthread，最重要的是会给自己调整调度类为 `idle_sched_class`。
- `cpu_startup_entry()`最终会以idle调度类的身份无限循环。

```c
void cpu_startup_entry(enum cpuhp_state state)
{
	current->flags |= PF_IDLE;
	arch_cpu_idle_prepare();
	cpuhp_online_idle(state);
	while (1)
		do_idle();
}

```

所以idle loop的重点，都在 `do_idle()`里面。所谓loop，就是 `cpu_startup_entry()`里面 `while(1)`，`do_idle()`做一次idle。

`do_idle()`的主要逻辑如下：

- 抬手就是一个 `while(!need_resched())`，以保证从idle出来后还没有其他可调度程序的话，就继续idle。一般会因为调度的时钟中断，或者等待的io来了数据触发中断，设置了需要调度的标志位，才导致这里退出。
  - 如果 `cpu_idle_force_poll`的值不为0，或者其他cpu正在准备广播ipi之类的中断过来，则进入idle poll 模式
    - `cpu_idle_force_poll`的值一般会被两个cmdline影响。`nohlt`会把它初始化为1，`idle=poll`会让它自增1。部分firmware或者arm的一些板子的驱动也会让它自增1.
  - 否则，会调用 `cpuidle_idle_call()`进入idle_call模式。
    - 这里就会governor和driver，来决定并进入哪个idle state来idle一次。
- 在判定 `need_resched()`之后，从循环退出来。此时调用 `schedule_idle()`将当前idle线程换出去，换其他进程进来运行。记住，idle永远是最低优先级的进程，只有调无可调的时候才会走到这里来。

执行idle动作的线程每个cpu一个，它们的pid都是0，线程名是 `swapper/x`（x是cpu id）。内核没有把它们的信息注册到/proc/下面，ps命令也捞不到他们的信息。如果需要看到他们的话，则需要在机器上运行crash命令，在里面执行ps强行捞出来所有内核的信息才行。

### governor 和 driver 的初始化

governor的选择是通过系统默认给他们配置的优先级来实现的。在没有显式通过cmdline来指定governor的情况下，每个governor在向内核注册时，内核根据他们代码里面写死的优先级(rating)来决定用哪个。这段逻辑参考 `cpuidle_register_governor()`。

比如说，menu的优先级是20，teo是19，ladder是10，haltpoll是9。但是，teo和ladder没有编译，menu在任何时候都会注册，haltpoll只有检测到自己在半虚拟化的条件下才会注册。那么，物理机上只有menu生效，而虚拟机上则是haltpoll生效。

在虚拟机里面，我们可以观察到governor切换的现象：

```
[    0.728913] cpuidle: using governor menu
[    1.998614] cpuidle: using governor haltpoll
```


driver在注册前，需要把自己能提供的各种state准备好。有些driver是静态配置好state的，有些是运行时动态根据cpu的能力初始的。

比如haltpoll就是静态的：

```
static struct cpuidle_driver haltpoll_driver = {
	.name = "haltpoll",
	.governor = "haltpoll",
	.states = {
		{ /* entry 0 is for polling */ },
		{
			.enter			= default_enter_idle,
			.exit_latency		= 1,
			.target_residency	= 1,
			.power_usage		= -1,
			.name			= "haltpoll idle",
			.desc			= "default architecture idle",
		},
	},
	.safe_state_index = 0,
	.state_count = 2,
};
```

intel_idle和acpi_idle就是动态的：

```
static void __init intel_idle_cpuidle_driver_init(struct cpuidle_driver *drv)
{
	// 用于 polling 的 state
	cpuidle_poll_state_init(drv);

	if (disabled_states_mask & BIT(0))
		drv->states[0].flags |= CPUIDLE_FLAG_OFF;

	drv->state_count = 1;

	// 从cpu能力探测出来的，各种不同类型的idle state
	if (icpu && icpu->state_table)
		intel_idle_init_cstates_icpu(drv);  
	else
		intel_idle_init_cstates_acpi(drv);
}
```


driver没有优先级的概念，所有他们的生效又要通过另一套逻辑来做。TODO
