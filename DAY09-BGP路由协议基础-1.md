# DAY9：BGP路由协议基础-1

------

## 一、基本概念

BGP是基于TCP（端口179）的矢量性路由协议，用于AS间路由传递和选路。

- **AS**：由单一组织管理，拥有唯一AS号
- BGP属于EGP范畴，但iBGP在AS内部运行
- **核心特点**：邻居建立与路由发布解耦，先建邻居再发路由，使用灵活
- BGP只负责控制平面，转发平面由MPLS等协议处理

------

## 二、邻居建立

### 2.1 邻居类型

- **iBGP**：同一AS内，AS号相同
- **eBGP**：不同AS间，AS号不同

### 2.2 配置与地址选择

**iBGP配置示例：**

- R1：`bgp 200` → `peer 10.0.12.2 as-number 200`
- R2：`bgp 200` → `peer 10.0.12.1 as-number 200`

**建邻地址建议：**

- **iBGP**：使用Loopback接口（需IGP打通underlay，提供链路冗余）
- **eBGP**：使用直连接口（TTL默认1跳，无需额外协议）

**架构分层：**

- Underlay（OSPF/ISIS）→ 传递Loopback路由 → 保证BGP邻居可达
- Overlay（BGP）→ 传递业务路由 → 实现业务互通

------

## 三、路由发布

两种方式：

1. **network**：`network 192.168.1.0 24`（需路由表中已存在）
2. **import-route**：`import-route ospf 1`（引入其他协议路由）

> 两者在选路时有细微优先级差异，功能基本等效

**路由类型判定：**

- 两端对等体均为iBGP → iBGP路由
- 两端对等体均为eBGP → eBGP路由

------

## 四、MP-BGP（多协议BGP）

当前主流为MP-BGP，支持多种协议栈：

| 协议栈 | 用途                |
| :----- | :------------------ |
| IPv4   | 普通IPv4路由        |
| VPNv4  | MPLS VPN（IPv4+RD） |
| IPv6   | IPv6路由            |
| EVPN   | 数据中心/Overlay    |

> **配置注意**：执行 `undo default ipv4-unicast` 可避免配置其他协议栈时自动创建IPv4邻居造成干扰

------

## 五、BGP五种报文

### 5.1 Open报文

- **时机**：TCP连接建立后立即发送
- **作用**：协商建立邻居
- **关键字段**：Version（4）、My AS、Hold Time（180s）、Router ID、Optional Parameters（认证/多协议能力）

### 5.2 Update报文

- **时机**：邻居建立后/路由发生变化时
- **作用**：发布新路由或撤销旧路由（增量更新）
- **关键字段**：
  - Withdrawn Routes → 撤销路由列表
  - Path Attributes → 路径属性（AS_PATH、NEXT_HOP、LOCAL_PREF、MED等）
  - NLRI → 可达路由前缀

### 5.3 Keepalive报文

- **时机**：周期性发送（默认60秒，Hold Time的1/3）
- **作用**：维持邻居关系（仅含BGP头，无正文）

### 5.4 Notification报文

- **时机**：检测到错误时发送
- **作用**：报告错误，中止邻居关系
- **常见错误码**：
  - 1 = 消息头错误
  - 2 = Open错误（AS不匹配、版本不支持、认证失败等）
  - 3 = Update错误（属性非法/缺失/长度错误、NEXT_HOP无效等）
  - 4 = Hold Time超时
  - 6 = Cease（管理性关闭、前缀数超限等）

### 5.5 Route-refresh报文

- **时机**：路由策略变化后/手动执行刷新命令时
- **作用**：请求对等体重新发送全部路由
- **关键字段**：AFI（地址族：1=IPv4，2=IPv6，16388=VPNv4）、SAFI（子地址族：1=单播，2=组播，4=VPN）

> **路由更新触发机制**：属性修改后不会周期性重发，需手动触发（发送方用Update，接收方用Route-refresh请求）

------

## 六、BGP状态机（6状态）

### 6.1 状态流转

**正常建连路径：**
Idle → (TCP成功) → Connect → (发送Open) → OpenSent → (收到Open，发送Keepalive) → OpenConfirm → (收到Keepalive) → Established

**TCP失败路径：**
Connect → (连接失败) → Active → (重试超时) → Connect（反复循环）

### 6.2 各状态说明

- **Idle**：初始状态，准备发起TCP连接
- **Connect**：等待TCP三次握手完成
- **Active**：TCP连接失败，主动重试（**最常见的卡死点**，原因多为IP不可达/ACL过滤/对端未启用BGP）
- **OpenSent**：TCP已通，已发送Open，等待对方Open
- **OpenConfirm**：双方Open参数兼容，已发送Keepalive，等待对方Keepalive
- **Established**：邻居建立成功，开始交换路由（稳态）

### 6.3 异常处理

- 参数不兼容（版本/AS/Hold Time）→ 发送Notification → 退回Idle
- Hold Time超时未收到Keepalive或Update → 退回Idle
- TCP拆链 → 退回Idle

------

## 七、水平分割规则

**iBGP水平分割**：从iBGP邻居学到的路由，不会传递给其他iBGP邻居。

> 解决方式：全互联（Full-mesh）或部署路由反射器（RR）

------

## 八、常用命令

| 命令                                        | 作用                                                         |
| :------------------------------------------ | :----------------------------------------------------------- |
| `peer x.x.x.x as-number AS号`               | 指定BGP邻居（IP和所属AS）                                    |
| `peer x.x.x.x connect-interface LoopBack 0` | 指定用Loopback口作为BGP报文的源接口（用Loopback建邻居时必配） |
| `peer x.x.x.x enable`                       | 在IPv4地址族中激活该邻居                                     |
| `network 网段 mask 掩码`                    | 宣告本地网段到BGP                                            |
| `import-route 协议`                         | 将其他协议（OSPF/Static等）的路由引入BGP                     |
| `router-id x.x.x.x`                         | 配置BGP的Router ID                                           |
| `display bgp peer`                          | 查看BGP邻居状态                                              |
| `display bgp routing-table`                 | 查看BGP路由表                                                |
| `refresh bgp all import`                    | 手动刷新路由（请求邻居重新发送）                             |
| `undo default ipv4-unicast`                 | 关闭默认IPv4单播地址族（配置多地址族时使用）                 |
| `peer x.x.x.x next-hop-local`               | **修改BGP路由的下一跳为本设备地址**                          |

```text
#配置BGP示例
#预先使用ping -a 源IP 目的IP 测试两边接口地址是否可通信

bgp 100
 router-id 2.2.2.2
 peer 1.1.1.1 as-number 100          # 指定邻居的Loopback IP和AS
 peer 1.1.1.1 connect-interface LoopBack0   # ★核心：指定源接口
 #
 ipv4-family unicast
  undo synchronization
  network 172.16.1.0 255.255.255.0   # 宣告本地网段
  peer 1.1.1.1 enable                # 在IPv4地址族中激活该邻居
```

#### `peer x.x.x.x next-hop-local`

- **作用**：将通告给邻居的BGP路由的下一跳修改为本设备地址。
- **为什么用**：IBGP传递EBGP路由时默认保留原始下一跳（外部AS接口IP），该地址在本AS内IGP中不可达，会导致**路由黑洞**。修改后下一跳为本设备Loopback/接口IP（IGP可达），消除黑洞。
- **配置位置**：**路由发送端**（谁把路由传给邻居，谁配置）。
- **华为**：`peer x.x.x.x next-hop-local` / **思科**：`neighbor x.x.x.x next-hop-self`
- **适用场景**：IBGP邻居（尤其跨多跳、用Loopback建邻居时）必配；EBGP邻居间一般不配。



------

## 九、BGP与MPLS的关系

- **BGP**：控制平面，决策路由
- **MPLS**：转发平面，根据标签转发数据
- 标签分发协议根据路由分发标签，生成标签转发表，数据到达后直接查标签表转发，无需修改报文头部

------

## 十、实验要点

1. iBGP建邻居推荐使用Loopback地址，需提前用IGP（OSPF/ISIS）打通
2. 多协议栈场景建议执行 `undo default ipv4-unicast`
3. 状态机卡在Active → 检查TCP连通性（ping、ACL、端口179是否放通）
4. 注意iBGP水平分割限制，合理规划Full-mesh或RR
5. eBGP多跳场景需修改TTL（`peer x.x.x.x ebgp-max-hop`）



 思考题：如何解决ebgp的环路？

使用路径属性将路由进行标记as，当路由回来发现是自己的as标签，就不接受。（AS Path）
