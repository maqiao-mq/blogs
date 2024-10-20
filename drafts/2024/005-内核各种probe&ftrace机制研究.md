# ftrace

所有的内核probe机制，基本上都离不开ftrace，包括：kprobe、livepatch、fprobe、tracepoint等等。

ftrace的前提要求已经很多教程在讲了，简单来说就是依赖gcc的mcount机制，也就是在每个函数的开头加上若干个nop，在需要trace的时候把这几个nop替换成jmp之类的指令。

ftrace的代码看起来还是很难懂的，涉及到的东西太多了。所以这里不会先深入到细节里面，而是先记录下它的设计原则，然后再来看代码。


设计原则：

- 每个ftrace的点，最初的设计是最多只能有一个ftrace ops生效。注意，是ftrace ops生效，多个kprobe是共享了同一个ftrace ops，所以是可以一起生效的。就是这个机制会带来很多东西
- 一张全局表来记录每个ftrace 点的情况，比如有没有被hook，被怎么hook了
- 每个ftrace ops，可以同时hook到多个函数上的


## Q

试试trampoline和kprobe同时hook一个函数

# kprobe

# uprobe

Q: 一个进程在启动的时候是怎么被打上uprobe的断点的？

# fprobe

# eprobe
