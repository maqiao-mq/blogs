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
