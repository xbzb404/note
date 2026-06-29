# DAY5：OSPF-报文认证-路由汇总-静默接口

#### 一、报文认证

用于保证OSPF邻居间报文交互的安全性。

- **两种认证级别**（接口认证优先于区域认证）：

  - **区域认证**：区域内所有路由器认证模式和口令一致。
  - **接口认证**：直连接口间的认证模式和口令一致。

- **三种认证方式**：

  - `simple`：明文传输，安全性低。
  - `md5`：密文传输，通过Key-ID和密钥匹配。
  - `keychain`：周期性更换密钥，增强安全性。

- **配置示例**：

  - **接口认证** (优先)：

    ```
    interface GigabitEthernet0/0/0
      ospf authentication-mode simple cipher 123456
    ```

    

  - **区域认证**：

    ```
    ospf 1
      area 0
        authentication-mode md5 1 cipher 123123
    ```

    - `md5`：认证类型
    - `1`：Key-ID，邻居必须一致（范围1-255）
    - `cipher 123123`：密钥密文保存

------

#### 二、路由汇总

减少路由表规模，提升网络稳定性。

- **ABR路由汇总**：在区域边界对区域间路由进行汇总。

  ```
  ospf 1 router-id 2.2.2.2
    area 0
      abr-summary 192.168.0.0 255.255.0.0
  ```

  

- **ASBR路由汇总**：对引入的外部路由进行汇总。

  ```
  ospf 1 router-id 3.3.3.3
    asbr-summary 172.16.0.0 255.255.0.0
    import-route direct
  ```

> **备注**：Loopback接口默认OSPF网络类型为P2P，会发布32位主机路由。若需发布24位网段，可修改网络类型为`broadcast`
>
> ```
> interface LoopBack0
>   ospf network-type broadcast
> ```

------

#### 三、静默接口

用于抑制Hello报文收发，降低边缘链路开销，但路由网段仍会正常通告。

- **配置**：

  ```
  ospf 1
    silent-interface GigabitEthernet0/0/0
  ```

  

  适用于连接终端PC的接口。

------

#### 四、OSPF 核心概念速查

- **核心思想**：每台路由器维护全网拓扑数据库（LSDB），基于SPF算法计算最短路径树，生成路由表。
- **特殊区域**：
  - `Stub`：过滤外部路由，ABR自动下发默认路由。
  - `NSSA`：允许有限引入外部路由（类型7 LSA），ABR可转换。
- **特殊节点**：
  - `ABR`：连接不同区域的边界路由器。
  - `ASBR`：引入其他协议或直连路由的边界路由器。
- **DR/BDR**：广播网络中减少邻接关系数量，非重点但需知道作用。