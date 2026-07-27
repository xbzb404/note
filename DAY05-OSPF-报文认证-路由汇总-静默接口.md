# DAY5：OSPF 进阶配置与核心概念

### 一、报文认证（Authentication）

用于保证 OSPF 邻居间报文交互的安全性，防止非法设备接入网络或伪造路由信息。

- **两种认证级别**（接口认证优先级高于区域认证）：

  - **区域认证**：在区域视图下配置，该区域内所有路由器接口必须保持认证模式和口令一致。
  - **接口认证**：在接口视图下配置，仅要求直连的相邻设备认证模式和口令一致，灵活性更高。

- **三种认证方式**：

  - `simple`：明文传输，安全性极低，仅用于测试。
  - `md5` / `hmac-md5`：密文传输，通过 Key-ID 和密钥进行匹配验证，是目前最常用的方式。
  - `keychain`：支持周期性更换密钥，提供极高的安全性。

- **配置示例**：

  - **接口认证 (优先)**：

    ```text
    [Huawei] interface GigabitEthernet0/0/1
    [Huawei-GigabitEthernet0/0/1] ospf authentication-mode md5 1 cipher Huawei@123
    ```

    *(注：`1` 为 Key-ID，两端必须一致；`cipher` 表示配置文件中以密文保存密钥)*

  - **区域认证**：

    ```text
    [Huawei] ospf 1
    [Huawei-ospf-1] area 0
    [Huawei-ospf-1-area-0.0.0.0] authentication-mode hmac-sha256 10 cipher Huawei@456
    ```

>  **【底层协议进阶】：为什么配置写 `md5`，抓包却是“认证类型 2”？**
> 在 OSPF 报文头部（OSPF Packet Header）中，有一个专门标识验证类型的字段（AuType）。根据协议规范：
>
> - **0**：空认证（Null Authentication）
> - **1**：简单口令认证（Simple Password）
> - **2**：密文认证（Cryptographic Authentication）
>
> 当我们在命令行配置 `md5` 时，设备在生成 OSPF 报文时会自动将 AuType 字段设为 **2**。此时，OSPF 报文头部原有的 8 字节 Authentication 字段不再存放密码，而是包含 Key ID、验证数据长度和防重放攻击的序列号。真正的 MD5 验证数据（Message Digest）是被附加在 OSPF 报文尾部的。

- 验证命令

  ：

  ```text
  display ospf peer brief  # 确认邻居状态是否因认证失败而卡在 Init/ExStart
  ```

------

### 二、路由汇总（Route Summarization）

将多条连续的明细路由聚合成一条汇总路由，可有效减少路由表规模、降低内存和 CPU 消耗，并屏蔽局部网络的路由震荡。

- **ABR 路由汇总**：在区域边界路由器（ABR）上对区域间路由进行汇总。

  ```text
  [Huawei] ospf 1 router-id 2.2.2.2
  [Huawei-ospf-1] area 1
  [Huawei-ospf-1-area-0.0.0.1] abr-summary 192.168.0.0 255.255.0.0
  ```

  *(注：将 Area 1 内的 192.168.x.x 网段汇总后，以 Type-3 LSA 的形式发布给 Area 0)*

- **ASBR 路由汇总**：在自治系统边界路由器（ASBR）上对引入的外部路由进行汇总。

  ```text
  [Huawei] ospf 1 router-id 3.3.3.3
  [Huawei-ospf-1] asbr-summary 172.16.0.0 255.255.0.0
  [Huawei-ospf-1] import-route static
  ```

> ️ **核心避坑备注**：
> Loopback 接口默认的 OSPF 网络类型为 `P2P`，在发布路由时默认会生成 32 位的主机路由（如 `10.1.1.1/32`）。若需发布实际的 24 位网段路由，必须修改网络类型：
>
> ```text
> [Huawei] interface LoopBack0
> [Huawei-LoopBack0] ospf network-type broadcast
> ```

------

### 三、静默接口（Silent-interface）

用于抑制接口收发 OSPF 协议报文（如 Hello 报文），但**该接口所在的直连网段依然会被正常通告**到 OSPF 域中。

- **应用场景**：

  - 连接终端 PC、服务器的接入层接口（防止终端伪造 OSPF 报文攻击网络）。
  - 仅需发布直连路由，但不希望建立邻居关系的接口。

- **配置示例**：

  ```text
  [Huawei] ospf 1
  [Huawei-ospf-1] silent-interface GigabitEthernet0/0/2
  # 也可以批量静默：silent-interface all，然后 undo silent-interface GE0/0/1 单独放行
  ```

- **验证命令**：

  ```text
  display ospf interface GigabitEthernet0/0/2  # 查看输出中的 Silent 字段是否为 Yes
  ```

------

### 四、OSPF 核心概念速查

- **核心思想**：每台路由器维护全网拓扑数据库（LSDB），基于 SPF（Dijkstra）算法计算最短路径树，最终生成路由表。

- 特殊区域

  ：

  - `Stub`（末梢区域）：过滤外部路由（Type-5 LSA），ABR 自动下发默认路由。
  - `NSSA`（非完全末梢区域）：允许有限引入外部路由（生成 Type-7 LSA，在 ABR 处转换为 Type-5），同时也会下发默认路由。

- 特殊节点

  ：

  - `ABR`（Area Border Router）：连接不同区域的边界路由器。
  - `ASBR`（AS Boundary Router）：引入其他协议（如静态、BGP）或直连路由的边界路由器。

- **DR/BDR**（指定路由器/备份指定路由器）：在广播网络（Broadcast）和 NBMA 网络中，用于减少邻接关系数量，优化 LSA 泛洪过程。