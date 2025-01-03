你好，我是周志明。

上一个模块，我们以“分布式的基石”为主题，了解并学习了微服务中的关键技术问题与解决方案。实际上，解决这些技术问题，原本就是我们架构师和程序员的本职工作。

而从今天开始，我们就进入了一个全新的模块，从微服务走到了云原生。在这个模块里，我会围绕“不可变基础设施”的相关话题，以容器、编排系统和服务网格的发展为主线，给你介绍虚拟化容器与服务网格是如何模糊掉软件与硬件之间的界限，如何在基础设施与通讯层面上帮助微服务隐藏复杂性，以此解决原本只能由程序员通过软件编程来解决的分布式问题。

## 什么是不可变基础设施？

“不可变基础设施”这个概念由来已久。2012 年，马丁 · 福勒（Martin Fowler）设想的“[凤凰服务器](https://martinfowler.com/bliki/PhoenixServer.html)”与 2013 年查德 · 福勒（Chad Fowler）正式提出的“[不可变基础设施](http://chadfowler.com/2013/06/23/immutable-deployments.html)”，都阐明了基础设施不变性给我们带来的好处。

而在[云原生基金会](https://en.wikipedia.org/wiki/Cloud_Native_Computing_Foundation)定义的“云原生”概念中，“不可变基础设施”提升到了与微服务平级的重要程度。此时，它的内涵已经不再局限于只是方便运维、程序升级和部署的手段，而是升华为了向应用代码隐藏分布式架构复杂度、让分布式架构得以成为一种能够普遍推广的普适架构风格的必要前提。

>**云原生定义（Cloud Native Definition）**
>
>Cloud native technologies empower organizations to build and run scalable applications in modern, dynamic environments such as public, private, and hybrid clouds. Containers, service meshes, microservices, immutable infrastructure, and declarative APIs exemplify this approach.
>
>These techniques enable loosely coupled systems that are resilient, manageable, and observable. Combined with robust automation, they allow engineers to make high-impact changes frequently and predictably with minimal toil.
>
>云原生技术有利于各组织在公有云、私有云和混合云等新型动态环境中，构建和运行可弹性扩展的应用。云原生的代表技术包括容器、服务网格、微服务、不可变基础设施和声明式 API。
>
>这些技术能够构建容错性好、易于管理和便于观察的松耦合系统。结合可靠的自动化手段，云原生技术使工程师能够轻松地对系统作出频繁和可预测的重大变更。
>
>—— [Cloud Native Definition](https://github.com/cncf/toc/blob/main/DEFINITION.md), [CNCF](https://www.cncf.io/)，2018

不过，不可变基础设施是一种抽象的概念，不太容易直接对它分解描述，所以为了能把云原生这个本来就比较抽象的架构思想落到实处，我选择从我们都比较熟悉的，至少是能看得见、摸得着的容器化技术开始讲起。

## 虚拟化的目标与类型

容器是云计算、微服务等诸多软件业界核心技术的共同基石。**容器的首要目标**是让软件分发部署的过程，从传统的发布安装包、靠人工部署，转变为直接发布已经部署好的、包含整套运行环境的虚拟化镜像。

在容器技术成熟之前，主流的软件部署过程是由系统管理员编译或下载好二进制安装包，根据软件的部署说明文档，准备好正确的操作系统、第三方库、配置文件、资源权限等各种前置依赖以后，才能将程序正确地运行起来。

这样做当然是非常麻烦的，[Chad Fowler](http://chadfowler.com/)在提出“不可变基础设施”这个概念的文章《[Trash Your Servers and Burn Your Code](http://chadfowler.com/2013/06/23/immutable-deployments.html)》里，开篇就直接吐槽：要把一个不知道打过多少个升级补丁，不知道经历了多少任管理员的系统迁移到其他机器上，毫无疑问会是一场灾难。

另外我们也知道，让软件能够在任何环境、任何物理机器上达到“一次编译，到处运行”，曾经是 Java 早年的宣传口号，不过这并不是一个简单的目标，不设前提的“到处运行”，仅靠 Java 语言和 Java 虚拟机是不可能达成的。因为一个计算机软件要能够正确运行，需要通过以下三方面的兼容性来共同保障（这里仅讨论软件兼容性，不去涉及“如果没有摄像头就无法运行照相程序”这类问题）：

**ISA 兼容**：目标机器指令集兼容性，比如 ARM 架构的计算机无法直接运行面向 x86 架构编译的程序。

**ABI 兼容**：目标系统或者依赖库的二进制兼容性，比如 Windows 系统环境中无法直接运行 Linux 的程序，又比如 DirectX 12 的游戏无法运行在 DirectX 9 之上。

**环境兼容**：目标环境的兼容性，比如没有正确设置的配置文件、环境变量、注册中心、数据库地址、文件系统的权限等等，当任何一个环境因素出现错误，都会让你的程序无法正常运行。

>**额外知识：ISA 与 ABI**
>
>[指令集架构](https://en.wikipedia.org/wiki/Instruction_set_architecture)（Instruction Set Architecture，ISA）是计算机体系结构中与程序设计有关的部分，包含了基本数据类型、指令集、寄存器、寻址模式、存储体系、中断、异常处理以及外部 I/O。指令集架构包含一系列的 Opcode 操作码（即通常所说的机器语言），以及由特定处理器执行的基本命令。
>
>[应用二进制接口](https://en.wikipedia.org/wiki/Application_binary_interface)（Application Binary Interface，ABI）是应用程序与操作系统之间或其他依赖库之间的低级接口。ABI 涵盖了各种底层细节，如数据类型的宽度大小、对象的布局、接口调用约定等等。ABI 不同于[应用程序接口](https://en.wikipedia.org/wiki/API)（Application Programming Interface，API），API 定义的是源代码和库之间的接口，因此同样的代码可以在支持这个 API 的任何系统中编译，而 ABI 允许编译好的目标代码在使用兼容 ABI 的系统中无需改动就能直接运行。

这里，我把使用仿真（Emulation）以及虚拟化（Virtualization）技术来解决以上三项兼容性问题的方法，都统称为**虚拟化技术**。那么，根据抽象目标与兼容性高低的不同，虚拟化技术又分为了五类，下面我们就分别来看看：

**指令集虚拟化（ISA Level Virtualization）**

即通过软件来模拟不同 ISA 架构的处理器工作过程，它会把虚拟机发出的指令转换为符合本机 ISA 的指令，代表为[QEMU](https://www.qemu.org/)和[Bochs](https://bochs.sourceforge.io/)。

**指令集虚拟化就是仿真**，它提供了几乎完全不受局限的兼容性，甚至能做到直接在 Web 浏览器上运行完整操作系统这种令人惊讶的效果。但是，由于每条指令都要由软件来转换和模拟，它也是性能损失最大的虚拟化技术。

**硬件抽象层虚拟化（Hardware Abstraction Level Virtualization）**

即以软件或者直接通过硬件来模拟处理器、芯片组、内存、磁盘控制器、显卡等设备的工作过程。

硬件抽象层虚拟化既可以使用纯软件的二进制翻译来模拟虚拟设备，也可以由硬件的[Intel VT-d](https://en.wikipedia.org/wiki/X86_virtualization#Intel-VT-d)、[AMD-Vi](https://en.wikipedia.org/wiki/X86_virtualization#AMD_virtualization_(AMD-V))这类虚拟化技术，将某个物理设备直通（Passthrough）到虚拟机中使用，代表为[VMware ESXi](https://www.vmware.com/)和[Hyper-V](https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/about/)。这里你可以知道的是，如果没有预设语境，一般人们所说的“虚拟机”就是指这一类虚拟化技术。

**操作系统层虚拟化（OS Level Virtualization）**

无论是指令集虚拟化还是硬件抽象层虚拟化，都会运行一套完全真实的操作系统，来解决 ABI 兼容性和环境兼容性的问题，虽然 ISA 兼容性是虚拟出来的，但 ABI 兼容性和环境兼容性却是真实存在的。

而操作系统层虚拟化则不会提供真实的操作系统，而是会采用隔离手段，使得不同进程拥有独立的系统资源和资源配额，这样看起来它好像是独享了整个操作系统一般，但其实系统的内核仍然是被不同进程所共享的。

**操作系统层虚拟化的另一个名字，就是这个模块的主角“容器化”**（Containerization）。所以由此可见，容器化仅仅是虚拟化的一个子集，它只能提供**操作系统内核以上**的部分 ABI 兼容性与完整的环境兼容性。

而这就意味着，如果没有其他虚拟化手段的辅助，在 Windows 系统上是不可能运行 Linux 的 Docker 镜像的（现在可以，是因为有其他虚拟机或者 WSL2 的支持），反之亦然。另外，这也同样决定了，如果 Docker 宿主机的内核版本是 Linux Kernel 5.6，那无论上面运行的镜像是 Ubuntu、RHEL、Fedora、Mint，或者是其他任何发行版的镜像，看到的内核一定都是相同的 Linux Kernel 5.6。

容器化牺牲了一定的隔离性与兼容性，换来的是比前两种虚拟化更高的启动速度、运行性能和更低的执行负担。

**运行库虚拟化（Library Level Virtualization）**

与操作系统虚拟化采用隔离手段来模拟系统不同，运行库虚拟化选择使用软件翻译的方法来模拟系统，它是以一个独立进程来代替操作系统内核，来提供目标软件运行所需的全部能力。

那么，这种虚拟化方法获得的 ABI 兼容性高低，就取决于软件能不能足够准确和全面地完成翻译工作，它的代表为[WINE](https://www.winehq.org/)（Wine Is Not an Emulator 的缩写，一款在 Linux 下运行 Windows 程序的软件）和[WSL](https://learn.microsoft.com/en-us/windows/wsl/about)（特指 Windows Subsystem for Linux Version 1）。

**语言层虚拟化（Programming Language Level Virtualization）**

即由虚拟机将高级语言生成的中间代码，转换为目标机器可以直接执行的指令，代表为 Java 的 JVM 和.NET 的 CLR。

不过，虽然各大厂商肯定都会提供在不同系统下接口都相同的标准库，但本质上，这种虚拟化技术并不是直接去解决任何 ABI 兼容性和环境兼容性的问题，而是将不同环境的差异抽象封装成统一的编程接口，供程序员使用。

## 小结

作为整个模块的开篇，我们这节课的学习目的是要明确软件运行的“兼容性”指的是什么，以及要能理解我们经常能听到的“虚拟化”概念指的是什么。只有理清了这些概念、统一了语境，在后续的课程学习中，我们关于容器、编排、云原生等的讨论，才不会产生太多的歧义。

## 一课一思

这节课介绍的五种层次的虚拟化技术，有哪些是你在实际工作中真正用过的？你是用来达成什么目的呢？

欢迎在留言区分享你的答案。如果觉得有收获，也欢迎你把今天的内容分享给其他的朋友。感谢你的阅读，我们下一讲再见。