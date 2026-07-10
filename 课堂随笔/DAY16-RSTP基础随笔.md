# DAY16 ：RSTP

`IEEE 802.1W`中定义的`RSTP`可以视为`STP`的改进版，`RSTP`在许多方面对`STP`进行 了优化，它的收敛速度更快，而且能够兼容`STP`

端口角色：

`RSTP`在`STP`的基础上，增加了两种接口角色（也就是给阻塞端口增加了分类），替代（`Alternate`）和备份（`Backup`）。

在`RSTP`中，共有`4`种接口角色：根接口（RP）、指定接口（DP）、替代接口和备份接口。



根接口可以进行快速切换：

​	替代接口是给根接口做的备份，根接口失效后其会在30s后成为根接口（15秒侦听 + 15秒学习）

​	

**备份接口（Backup Port）**

备份接口是指定接口的备份，当指定接口失效后，备份接口会在30秒后成为新的指定接口（15秒侦听 + 15秒学习）。

备份接口处于阻塞状态，是由于收到自己发送的BPDU而阻塞的端口（一般出现在通过Hub连接的共享链路，同一台交换机的两个口都接到同一个Hub上，一个口发BPDU绕回来被另一个口收到，收到自己的BPDU的口就成为备份接口）。实际生产中极少见，因Hub已淘汰。

补充：非根桥的指定接口也会发BPDU（从根接口收到根桥BPDU后，重新生成并发出，根桥ID不变，网桥ID改成自己的），所以"收到自己的BPDU"是可能发生的。非根桥能否发BPDU取决于端口角色：指定接口能发，根接口只收不发，阻塞口只收不发。

------

**RSTP端口角色总结**

根接口（RP）：每台非根桥选一个，去根桥的最优入口，最终转发

指定接口（DP）：每条链路选一个，离根桥最近的出口，最终转发

替代接口（Alternate Port）：根接口的备份，根接口失效后30秒上位

备份接口（Backup Port）：指定接口的备份，指定接口失效后30秒上位

根桥无RP，所有端口都是DP。只有RP和DP能进入转发状态。



端口状态

RSTP将端口状态从STP的五种缩减为三种，根据端口是否转发用户流量和学习MAC地址来划分：

**Discarding状态（丢弃）**：端口即不转发用户流量，也不学习MAC地址。相当于STP中的禁用、阻塞和侦听三种状态的合并，无论是被阻塞还是处于过渡期，统一归类为Discarding。

**Learning状态（学习）**：端口不转发用户流量，但是学习MAC地址。和STP的学习状态基本一致，为正式转发做准备，持续一个Forward Delay（15秒）。

**Forwarding状态（转发）**：端口既转发用户流量，又学习MAC地址。只有根接口和指定接口才能进入此状态。

------

**STP与RSTP状态对照表**

STP五种状态：禁用、阻塞、侦听、学习、转发
RSTP三种状态：Discarding、Learning、Forwarding

对应关系：禁用+阻塞+侦听 = Discarding；学习 = Learning；转发 = Forwarding

RSTP之所以能快速收敛，部分原因就是减少了状态数量，简化了端口迁移逻辑，不再需要像STP那样严格经过阻塞→侦听→学习→转发的固定顺序。

## 配置BPDU的处理

**BPDU发送**

RSTP对配置BPDU的发送方式做了改进：拓扑稳定后，无论非根桥设备是否收到根桥传来的BPDU，非根桥设备仍然按照Hello Time规定的时间间隔（默认2秒）自主发送配置BPDU，完全由每台设备独立进行，不再依赖上游触发。

STP的发送方式则不同：拓扑稳定后，只有根桥按照Hello Time间隔主动发送BPDU，其他非根桥设备必须收到上游设备发来的BPDU后，才会触发发送自己的BPDU。这种依赖触发的机制使得STP收敛速度较慢。

**邻居失效检测**

RSTP中，如果一个端口在超时时间（三个Hello Time周期，即 Hello Time × 3 = 6秒）内没有收到上游设备发来的配置BPDU，则认为与该邻居的协商失败，立即开始重新计算。

STP中，设备需要先等待一个Max Age（默认20秒），确认收不到BPDU后才会开始重新计算，比RSTP慢得多。

**次优BPDU处理**

RSTP中，当端口收到上游指定桥发来的RST BPDU时，会将自身缓存的BPDU与收到的BPDU进行比较。如果自身缓存的BPDU优于收到的BPDU，则直接丢弃收到的BPDU，立即回应自身缓存的BPDU，无需等待计时器超时。

STP中，只有指定端口会立即处理次优BPDU，其他端口直接忽略，必须等到Max Age（默认20秒）超时后，缓存的次优BPDU才会老化，然后发送自身更优的BPDU进行重新收敛。

**一句话总结**

RSTP遇到次优BPDU立即丢弃并回应自己的更优BPDU；STP只有指定端口会处理，其他端口要等20秒老化才能回应。

## 快速收敛机制

**根端口快速切换**

根端口失效时，最优的替代接口（Alternate Port）直接成为根端口，进入转发（Forwarding）状态。因为替代接口（Alternate Port）连接的网段上必然存在一个通往根桥的指定端口，无需重新选举。切换过程：Discarding → Forwarding，无需经过Learning状态。

**指定端口快速切换**

指定端口失效时，最优的备份接口（Backup Port）直接成为指定端口，进入转发（Forwarding）状态。备份接口（Backup Port）作为指定端口的备份，提供了另一条从根桥到该网段的备份通路。切换过程：Discarding → Forwarding，无需经过Learning状态。

**边缘端口（Edge Port）**

边缘端口连接终端设备，不参与RSTP计算，从丢弃（Discarding）直接进入转发（Forwarding），实现0秒收敛。切换过程：Discarding → Forwarding。边缘端口的UP/Down不会触发拓扑变更。如果收到BPDU，则立即变为普通STP端口参与计算；若开启了BPDU保护（BPDU Guard），收到BPDU直接关闭该端口（error-down）。

**一句话总结**

根口坏→替代接口（Alternate Port）直接转发；指定口坏→备份接口（Backup Port）直接转发；边缘端口→0秒转发，收到BPDU变普通口。RSTP快速收敛的核心就是绕过Learning状态，直接从Discarding切到Forwarding。



P/A机制是为了防环

P/A机制会从出现变化的设备往下依次执行，整个收敛过程会一直导致断网，这个是机制无法避免，不过非常快，几乎是毫秒级的。



RSTP保护

### BPDU保护（BPDU Guard）

**一句话概括**：
防止边缘端口意外收到BPDU，避免非法交换机接入扰乱网络拓扑。

**触发条件**：
开启了BPDU保护的边缘端口收到任何BPDU报文。

**处理动作**：
**立即将该端口关闭（error-down）**，彻底隔离该端口，阻断潜在环路或拓扑变更风险。

**恢复方式**：
需管理员手动介入（`undo shutdown`）或配置自动恢复定时器。

**极简版**：

> **边缘端口收到BPDU就直接“封端口”，防非法接入。**

------

### Root保护（Root Guard）

**一句话概括**：
防止下游交换机通过发送更优BPDU抢占根桥地位，维护网络拓扑的稳定性。

**触发条件**：
开启了Root保护的指定端口，收到优先级**更高**（更优）的RST BPDU报文。

**处理动作**：
**接口始终保持为指定端口（Designated Port）角色**，但状态变为 **Discarding**（丢弃），不转发用户数据。既不让出指定端口角色，也不转发数据，从而阻止下游设备成为根桥。

**恢复方式**：
当不再收到更优BPDU后，端口自动恢复为正常的转发状态。

**极简版**：

> **收到更优BPDU就“赖着不走”，保持指定端口角色但阻塞转发，防根桥被抢。**

### 环路保护（Loop Protection）

**一句话概括**：
防止端口因收不到BPDU而误判链路正常，导致形成环路。

**触发条件**：
根端口或Alternate端口长时间收不到上游BPDU。

**处理动作**：

- 根端口 → **进入Discarding状态**（不转发数据），角色变为指定端口
- Alternate端口 → **保持在Discarding状态**（原本就不转发），角色变为指定端口
- 同时向网管发送告警

**恢复方式**：
链路恢复后，重新收到BPDU，端口自动协商恢复原角色和状态。



RSTP实验：

全局基础配置

**工作模式（默认是MSTP，使用RSTP需切换）**

```
[Huawei] stp mode rstp
```

**全局启用STP（默认已启用）**
```
[Huawei] stp enable
```

**配置为根桥**

```
[Huawei] stp root primary
```
配置后优先级强制变为0，不能再手动改。

**配置为备份根桥**
```
[Huawei] stp root secondary
```
配置后优先级强制变为4096，不能再手动改。

**手动配置优先级（步长4096，默认32768）**
```
[Huawei] stp priority 32768
```

**路径开销计算标准（全网需一致，默认dot1t）**
```
[Huawei] stp pathcost-standard dot1t
```
可选值：`dot1d-1998` / `dot1t` / `legacy`。


### 接口基础配置

进入接口视图：
```
[Huawei] interface GigabitEthernet 0/0/1
```

**设置接口路径开销（数值越小越优）**
```
[Huawei-GigabitEthernet0/0/1] stp cost 200
```

**设置接口优先级（步长16，默认128）**
```
[Huawei-GigabitEthernet0/0/1] stp priority 128
```

**配置为边缘端口（接终端的口必须配）**
```
[Huawei-GigabitEthernet0/0/1] stp edged-port enable
```


### 保护功能配置

**BPDU保护（全局，保护所有边缘端口）**
```
[Huawei] stp bpdu-protection
```
边缘端口收到BPDU后直接error-down关闭端口。默认关闭。

**根保护（接口视图，配在指定端口上）**
```
[Huawei-GigabitEthernet0/0/1] stp root-protection
```
收到更优BPDU时保持指定端口角色，但进入Discarding状态。默认关闭。配置了根保护的端口不可以再配置环路保护。

**环路保护（接口视图，配在根端口或Alternate端口上）**
```
[Huawei-GigabitEthernet0/0/1] stp loop-protection
```
长时间收不到BPDU时进入Discarding状态，防止形成环路。默认关闭。配置了环路保护的端口不可以再配置根保护。

**TC保护（全局，防TC报文风暴）**
```
[Huawei] stp tc-protection interval 10
[Huawei] stp tc-protection threshold 15
```
interval设置统计时间窗口（秒），threshold设置窗口内允许处理TC报文的最大次数。超出阈值的部分只计数不刷新转发表。


### 典型配置组合

**最简生产环境**
```
[Huawei] stp mode rstp
[Huawei] stp root primary
[Huawei] interface GigabitEthernet 0/0/1
[Huawei-GigabitEthernet0/0/1] stp edged-port enable
```

**加固防攻击（根保护 + BPDU保护）**
```
[Huawei] stp mode rstp
[Huawei] stp root primary
[Huawei] stp bpdu-protection

[Huawei] interface GigabitEthernet 0/0/24
[Huawei-GigabitEthernet0/0/24] stp root-protection

[Huawei] interface GigabitEthernet 0/0/1
[Huawei-GigabitEthernet0/0/1] stp edged-port enable
```

**防频繁拓扑震荡（TC保护调优）**
```
[Huawei] stp tc-protection interval 10
[Huawei] stp tc-protection threshold 10
```


### 约束条件速记

根桥或备份根桥配置后，优先级被锁定，不能再手动修改：
```
[Huawei] stp root primary
# 之后不能再敲 stp priority ×××
```

根保护和环路保护互斥，不能在同一端口同时开启：
```
[Huawei-GigabitEthernet0/0/1] stp root-protection
# 再敲 stp loop-protection 会报错
```

路径开销计算标准全网必须一致，否则选路结果可能不一致。默认是dot1t，如果改了legacy则所有设备都要改。



对于边缘端口

```
#更优
int g0/0/1
	stp edged-port enable
#虽然功能差不多，但性能上会有问题
int g0/0/1
	undo stp enable 
```

