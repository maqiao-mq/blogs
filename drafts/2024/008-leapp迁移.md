leapp 是 redhat 用于给用户升级 OS 的软件，帮助用户从 RHEL 7 升到 8、9，或者从 RHEL 8 升到 9.

这个软件是 inplace 地升级，它强大的地方在于它的产品化做的很好，把各种复杂的细节都封装起来了，而且管理地紧紧有条。

leapp 分为两块：

- [leapp 工具](https://github.com/oamg/leapp)本身，这个工具实际上是个框架，里面定义了各种基本单元，并把这些单元串成了一个 workflow
- [leapp-repository](https://github.com/oamg/leapp-repository)，这个是基于框架的各种具体实现，它定义了 RHEL 7、8在往高版本升级的时候，应该做什么样的检查，怎么迁移，怎么处理各种细节。

leapp 工具定义的各种基本单元，可以参考[这个文档](https://leapp.readthedocs.io/en/latest/terminology.html)。

简单来说：

1. 它提供了一个消息总线一类的抽象，每个Actor 挂在总线上，等待某些事件，然后消费它，并再次生成某些事件。
2. 它在整体上定义了一个 workflow，workflow 被拆解成多个 phase，每个 phase 又分成 3 个 stage，分别是pre-xx、main-xxx 和 post-xxx。
3. Actor 通过 tag 的方式和某个 phase 绑定在一起

所以，当整个 workflow 开始执行的时候，就会依次过每个 phase，每个 phase 就会通过消息机制各种传来传去，然后触发 actor 执行。


这是个很通用的框架，但是目前 redhat 只用它来实现了 RHEL 的迁移。整个迁移的 workflow 可以参考[这个文档](https://leapp.readthedocs.io/en/latest/inplace-upgrade-workflow.html)。

整体来说，它分成三大块：

1. 迁移前检查
   这一步检查是不是满足迁移的条件，记录各种已经安装的旧的rpm包，并且下载对应的新包，而且会通过一个overlayfs来尝试性的验证是不是这些新包可以被安装上来。这种尝试在失败的时候是有回滚的，所以相对来说做得是非常完善的，能大概率避免把机器搞到一个中间状态里面去。
2. 重启后进入 ramfs，开始替换 rpm 包进行迁移。
   这一步会用 upgrade 的方式升级新 rpm 包，然后检查所有的老包，把该卸载的卸载掉。
   以及，如果有些包的配置在两个版本间发生了不兼容的改变，还需要手动处理下这些配置项，将它们适当地转换成新的配置方式。
3. 再次重启，收尾，完成迁移

从上面的流程可以看出来，如果有某个自定义的配置文件放到了/etc、/usr之类的目录下了，那么迁移完成后，这些配置其实还是在的。

整个流程只会处理能通过yum数据库识别到的那些包，所以如果是自己 `make install`的软件，那么是不会得到处理的。那如果因为glibc的升级导致老的软件包没办法动态链接上的话，也就只能自行重编了。

另外，如果有一些非官方源搞下来的rpm包，那么在迁移的过程中，很可能因为找不到新的对应的包而中断迁移过程。
