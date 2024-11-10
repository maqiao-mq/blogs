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

- 下一次定时器事件（timer event）的触发时间。这个是精确可控的时间，可以用来和idle state里面给的target residency来比较，决定要不要进入idle
- idle duration。从最近一次idle state切换出来，并不一定是定时器触发的，也可能是其他不确定事件触发的。这种中途被打断，实际 idle 的时间称为idle duration，它也要用来辅助决策。

但是governor实际怎么根据这两个信息做决策，还是取决于实际算法。

目前有3中governor：menu、TEO和ladder。一般不建议在生产环境切换governor，默认配置取决于内核编译的时候的配置，以及一些运行时对cpu能力的判断。可以在 `/sys/devices/system/cpu/cpuidle`中查看 `current_governor_ro`，以确定当前用的是哪个governor。服务器上一般是menu。

driver主要看cpu platform的能力了。比如intel上，会提供intel_idle和acpi_idle，一般在系统启动的时候就会确定好driver，然后后续就无法更改了。在 `/sys/devices/system/cpu/cpuidle/`里面可以查看 `current_driver`获取这个信息

## scheduler tick和idle

内核是有一个固定周期的scheduler tick的，用来周期性强制将当前进程换出去，换新的进程来运行，保障调度的公平性。

但是，在idle loop的情况下，其实没必要周期性地tick来打扰idle状态。如果要内核不管在什么情况下都要tick的话，可以在编译的时候关掉CONFIG_NO_HZ_IDLE，或者启动参数里面加上 `nohz=off`。

如果在idle的时候默认会停止tick，则系统被称为tickless的。tickless的情况下，governor默认使用menu；非tickless的情况下，governor默认使用ladder。


## 控制idle行为的cmdline

公用：

`cpuidle.off=1`，这个不会禁止cpu运行idle loop，但是会禁止governor和driver的生效

`cpuidle.governor= `，这个用来强行指定governor。

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
