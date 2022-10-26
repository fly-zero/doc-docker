# **Docker**

## **1 在容器中可以看到什么**
- [ ] **文件系统**

    容器可以有独立的文件系统，同时可以将外部的目录挂载进容器的文件系统中。

- [ ] **CPU**

    容器可以看到主机上所有的CPU，通过cgroup机制可以限制CPU的使用。

- [ ] **内存**

    容器可以看到主机上所有内存，通过cgroup机制可以限制内存的使用。

- [ ] **PCIE端口**

    Linux将PCIE端口通过sysfs（挂载在/sys目录）暴露给用户层，Docker容器在创建时自动挂载sysfs到/sys目录。DPDK初始化过程通过/sys/bus/pci下的内容获取端口的信息。

- [ ] **IPC**

    容器内部创建的System V IPC（共享内存、消息队列、信号量）和POSIX IPC（消息队列）容器内部的进程可见。其它容器无法看见，但可以创建跟其它容器中相同ID或者name的IPC对象。

- [ ] **网卡**

    容器内部的网卡是虚拟网卡，与外部通信经过NAT。容器也可以在启动时使用主机的网卡。

## **2 Docker的隔离**

### **2.1 文件系统隔离**
文件系统的隔离是通过chroot和Union FS来实现的。

#### **2.1.1 chroot**
`chroot`是Linux操作系统提供的一个系统调用，该系统调用可以让进程设置特定的路径为根目录。调用了`chroot`的进程所看到文件系统都是从指定路径开始的，无法再看到更外层目录的文件系统。
```
/
|
+---tmp
|    |
|    +---foo
|    |
|    +---bar
|
+---usr
```
假设有上面一个文件系统，程序在初始启动时可以看到完整的文件系统的，当调用`chroot("/tmp")`之后，程序能所看到的文件系统将是下面的情况
```
/
|
+---foo
|
+---bar
```
#### **2.1.2 Union FS**
Union FS是联合文件系统，它允许对多个目录挂载到同一个挂载点，并得到一个合并的目录结构视图。Union FS有上下层关系，上下层中同名的文件在挂载点中是上层覆盖下层。
![](overlay_constructs.jpg)

假设有下面这样一个文件树视图

```
.
|
+---A
|   |
|   +---a.txt
|   |
|   +---x.txt
|
+---B
|   |
|   +---b.txt
|   |
|   +---x.txt
|
+---C
|
+---merge
|
+---work
```

通过Union FS将A,B,C合并挂载到merge目录：
```
# mount \
> -t overlay overlay \
> -o lowerdir=A:B,upperdir=C,workdir=work \
> merge
```
`lowerdir=A:B` 表示将A,B目录作为下层，并且A在B之上，因此B目录中同名的文件`x.txt`将被A目录中的覆盖。

`upperdir=C` 表示C目录作为上层，在merge目录中的发生修改文件将被写到C目录中；在merge中删除了A,B目录中同名文件之后，将在C目录中增加一个字符类设备型的同名文件。

`workdir=work` 是overlay用于工作的目录。

执行上面的`mount`命令后将得到下面一这样一个文件视图
```
.
|
+---A
|   |
|   +---a.txt
|   |
|   +---x.txt
|
+---B
|   |
|   +---b.txt
|   |
|   +---x.txt
|
+---C
|
+---merge
|     |
|     +---a.txt
|     |
|     +---b.txt
|     |
|     +---x.txt
|
+---work
```

### **2.2 namespace隔离**

#### **2.2.1 UTS namespace**
UTS namespace提供了主机名和域名的隔离，它使用docker容器在网络中被视为独立的结点。在`docker run`命令中通过`--hostname`和`--domainname`来设置主机名和域名。通过`--uts=host`来关闭UTS namespace。

#### **2.2.2 IPC namespace**
IPC(进程间通信)资源包括：
* **System V IPC**： 消息队列、信号量、共享内存
* **POSIX IPC**：消息队列（不含信号量、共享内存，它们是基于tmpfs的，通过mount namespace来隔离）

它们有一个共同特征——IPC对象不是通过文件系统路径来标识。在一个IPC namespace中创建的IPC对象仅对该IPC namespace中的进程可见。

#### **2.2.3 PID namespace**
PID namespace隔离进程ID的数字空间，位于不同空间中的两个不同里程可以有相同的PID。内核为PID namespace维护了一个树状结构，父结点可以看到子结点的中的进程。子结点无法看到父结点的进程。最顶层为初始的root PID namespace。最下层的PID namespace进程在每一层中都一个PID。

#### **2.2.4 Mount namespace**
Mount namespace提供了挂载点的隔离。在一个Mount namespace中挂载的文件系统仅能被该namespace下的成员看到。

#### **2.2.5 Network namespace**
Network namespace提供了系统网络资源的隔离，包括IPv4、IPv6 协议栈，IP 路由表，防火墙规则，端口号，socket等。容器内部的虚拟网卡通过veth与docker0网桥通信。
![](docker-network.jfif)

#### **2.2.6 User namespace**
User namespace隔离了安全相关的标识与属性，包括用户ID，组ID，root目录，密钥，特殊权限等。普通用户创建的容器，在容器内部可以拥有超级用户的权限。

#### **2.2.7 Cgroup namespace**
Cgroup namespace限制和隔离一组进程对系统资源的使用。对不同资源的具体管理是由各个子系统分工完成的。
|**子系统**|**作用**             |
|----------|---------------------|
|devices   |设备权限控制          |
|cpuset    |分配指定的CPU和内存节点|
|CPU       |控制CPU使用率         |
|cpuacct   |统计CPU使用情况       |
|memory    |限制内存的使用上限    |
|freezer   |暂停Cgroup 中的进程   |
|net_cls   |配合流控限制网络带宽   |
|net_prio  |设置进程的网络流量优先级|
|perf_event|允许 Perf 工具基于 Cgroup 分组做性能检测|
|huge_tlb  |限制 HugeTLB 的使用   |