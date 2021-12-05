# alpine

​		**Alpine Linux 是一个基于 musl libc 和 busybox 的面向安全的轻量级的、独立的、非商业的、通用的  Linux 发行版，专为欣赏安全性、简单性和资源效率的高级用户而设计。**

### Alpine特点

- 小的

​		Alpine Linux 是围绕 musl libc 和 busybox 构建的。这使得它比传统的 GNU/Linux 发行版更小，资源效率更高。一个容器需要不超过 8 MB 的空间，最小安装到磁盘需要大约 130 MB 的存储空间。您不仅可以获得成熟的 Linux 环境，还可以获得来自存储库的大量软件包选择。

​	二进制包被精简和拆分，让您可以更好地控制安装的内容，从而使您的环境尽可能小而高效。

- 简单的

​	Alpine Linux 是一个非常简单的发行版，它会尽量避开你。它使用自己的名为 apk 的包管理器、OpenRC 初始化系统、脚本驱动的设置，仅此而已！这为您提供了一个简单、清晰的 Linux 环境，没有任何噪音。然后，您可以在此基础上添加项目所需的软件包，因此无论是构建家庭 PVR、iSCSI 存储控制器、超薄邮件服务器容器还是坚如磐石的嵌入式交换机，其他任何东西都不会挡道。

- 安全的

​	Alpine Linux 的设计考虑了安全性。所有用户态二进制文件都被编译为具有堆栈粉碎保护的位置独立可执行文件 (PIE)。这些主动安全功能可防止利用整类零日漏洞和其他漏洞。

### musl libc简介

​	musl 是构建在 Linux 系统调用 API 之上的 C 标准库的实现，包括在基本语言标准、POSIX 和广泛认可的扩展中定义的接口。 musl 是轻量级的、快速的、简单的、免费的，并且在标准一致性和安全性方面力求正确。

### busybox简介

​	BusyBox 是一个集成了三百多个最常用Linux命令和工具的软件。BusyBox 包含了一些简单的工具，例如ls、cat和echo等等，还包含了一些更大、更复杂的工具，例grep、find、mount以及telnet。有些人将 BusyBox 称为 Linux 工具里的[瑞士军刀](https://baike.baidu.com/item/瑞士军刀/152816)。简单的说BusyBox就好像是个大工具箱，它集成压缩了 Linux 的许多工具和命令，也包含了 Linux 系统的自带的shell。



### 参考资料

https://www.alpinelinux.org/about/

http://musl.libc.org/

https://busybox.net/about.html

https://baike.baidu.com/item/busybox/427860?fr=aladdin