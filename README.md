#Magic Connect – 技术规格文档

1. 项目概述

名称：Magic Connect
类型：混合架构虚拟组网系统（主控 + Agent）
传输基础：支持 TCP 或 QUIC（用户配置），数据平面强制多路复用
实现语言：Rust（stable 1.70+）
设计目标：Agent 轻量高效，主控智能决策，支持分组隔离、动态选路、自定义路径，部署灵活（单机/集群）

---

2. 整体架构

```
[主控集群] <---> [PostgreSQL]
     ↑
     | 控制信令 (TCP/QUIC)
     ↓
[Agent A] <---数据平面(多路复用)---> [Agent B]
```

· 主控（Controller）：负责 Agent 注册、分组策略、路径计算、探测调度。支持单机模式或 Raft 集群模式。
· Agent：部署在用户设备上，可配置为“仅接入”或“接入+中继”。
· 数据库：PostgreSQL（≥13），存储持久化配置、Agent 注册信息、分组策略、审计日志等。

---

3. 核心功能模块

3.1 Agent 注册与地址管理

· Agent 启动时生成唯一 UUID。
· 向主控上报信息（通过控制信令）：
  · 自定义接入 IP（可选）：用户可在配置文件中指定 announce_ip 和 announce_port，主控将使用该地址作为通信目标（适用于固定公网 IP 或端口映射）。
  · 若不提供，主控通过 STUN 或本地 socket 自动探测公网 IP + 端口，同时记录内网 IP。
  · 地理位置：可调用用户配置的 API 获取经纬度，或由面板手动设置地区。
  · 角色（仅接入 / 中继）。
  · 所属分组（一个或多个）。
· 主控将 Agent 信息存入数据库（agents 表）并缓存在内存中。

3.2 分组与访问控制

· 分组定义：每个 Agent 可属于多个组（多对多关系）。
· 默认策略：仅同组内 Agent 允许直接通信。
· 跨组白名单：由管理员配置（例如 allow group A to group B），存储在 group_policies 表。
· 策略实施：
  · 主控计算路径前先校验策略。
  · 主控将策略（带版本号）推送给 Agent，Agent 本地转发时校验数据包头部（源组签名）。
· 防绕过：数据包携带 HMAC 签名的组标签，接收端验证。

3.3 加密与合规

· 默认加密：AES-256-GCM（数据平面）。
· 可选加密：国密 SM2（公钥） + SM4（对称），用户配置开启。
· 流量特征隐藏：加密后负载添加固定模式填充或伪装成 HTTP/3 头部，避免全随机数。
· 密钥交换：主控通过控制信道分发对称密钥（定期轮换）。

3.4 连接与多路复用

· 数据平面传输协议：用户可选 TCP 或 QUIC。
· 多路复用：无论 TCP 还是 QUIC，必须启用流复用（QUIC 原生支持；TCP 上叠加 yamux）。
· 目的：避免队头阻塞，提高吞吐。

3.5 智能选路（核心）

3.5.1 探测机制

· 探测内容：延迟（RTT ms）、带宽（Mbps）、丢包率（%）。
· 探测方式：主控指令两个 Agent 互发探针（UDP 小包或实际业务流）。
· 动态频率：
  · 高流量对（过去 5 分钟平均带宽 > 1 Mbps）：每 10 秒探测一次。
  · 中流量对（偶尔有流量）：每 60 秒一次。
  · 低流量对（几乎无通信）：每 10 分钟一次。
  · 停止探测：Agent 离线超过 5 分钟。
· 结果存储：存入数据库 probe_results 表（带时间戳），内存中保留最新值。

3.5.2 路径计算算法

输入：源 Agent S，目标 Agent T
输出：有序路径列表（主、备），每条路径包含经过的节点序列和评分。

步骤：

1. 策略过滤：若 S 与 T 不在同一组且无跨组规则，返回空（拒绝连接）。
2. 候选路径枚举：
   · 直连路径：S → T
   · 单中继路径：S → R → T，其中 R 的角色为“接入+中继”
   · 双中继路径：S → R1 → R2 → T（中继数量上限可配置，默认 2）
   · 利用地理位置预过滤：只考虑同地区或经纬度距离 < 2000 km 的中继节点。
3. 评分计算：
   · 每条路径的总延迟 = 各段延迟之和。
   · 总成本 = 各段流量成本之和（用户可为每个 Agent 设置每 GB 成本）。
   · 综合评分 = α * 总延迟(ms) + β * 总成本(分/GB)。
     · 延迟优先模式：α=1, β=0.001
     · 经济优先模式：α=0.001, β=1
   · 附加规则：若多条路径的评分相差 ≤ 5%（即延迟相近），则从中优先选择每一跳延迟都 < 100ms 的路径。若没有完全满足的，则选择最大跳延迟最小的路径。
4. 输出：Top 2 路径（主/备），写入内存缓存和数据库 routes 表。

3.5.3 故障切换

· 主控持续监控活跃路径质量（通过后续探测或业务流反馈）。
· 若主路径连续 3 次探测失败，或延迟 > 200ms 且丢包 > 10%，立即触发重算并切换到备路径。
· 切换速度要求 < 3 秒。

3.5.4 流量耗尽处理

· Agent 本地统计每路径流量，达到用户设定的阈值（如 10 GB/月）时上报主控。
· 主控标记该路径不可用，自动切换，并通知用户。

3.6 NAT 穿透与中继降级

· P2P 尝试顺序：UDP 打洞（STUN）→ TCP 同时打开 → UPnP/NAT-PMP。
· 失败回退：请求主控分配中继 Agent。
· 中继选择：延迟低、负载轻、与两端均可连通的节点。
· 中继转发：使用相同多路复用协议，中继节点不解密数据。

3.7 手动档与自定义权重

· 手动档：用户可通过 API 或面板为 (src, dst) 强制指定固定路径（如 A→D→E→C），优先级最高。
· 自定义权重：为多条路径分配百分比（如主 70%，备 30%），主控按权重分配新流。

3.8 部署与运维

· 部署形式：
  · 二进制原生（Linux x86_64/arm64, Windows, macOS）
  · Docker 容器（推荐 host 网络模式）
· 数据库：PostgreSQL 13+（必须）。提供 schema 迁移脚本。
· 配置文件（YAML）：
  · 主控地址、协议（tcp/quic）
  · Agent ID、分组、角色
  · 自定义接入 IP（可选）
  · 加密选择
  · 成本、流量阈值
  · 偏好策略
  · 手动档路径列表
· 监控：暴露 Prometheus 指标。

---

4. 主控架构（单机 → 集群扩展）

4.1 设计原则

· 统一代码基：通过配置项 mode 区分 standalone 或 cluster。
· 无外部依赖（除数据库外）：不使用 Redis、etcd。
· 嵌入式 Raft：使用 openraft，状态机操作通过 Raft 日志同步。

4.2 单机模式

· 单个主控进程，直接操作本地状态机和数据库。
· 适合 ≤1000 Agent，开发/测试/小型生产。
· 可靠性依赖数据库备份和进程监控（如 systemd 自动重启）。

4.3 集群模式

· 3 或 5 个主控节点组成 Raft 集群。
· 状态机：所有写操作（注册、探测结果、路径更新）通过 Raft 提交。
· Leader：负责所有写请求、路径计算、探测调度。
· Follower：转发写请求到 Leader；可处理读请求（支持 stale read 以降低延迟）。
· 故障转移：Leader 宕机后自动选举新 Leader（约 1-3 秒），期间写操作不可用，读操作仍可由 Follower 服务（若配置允许）。
· 数据库：每个主控节点连接同一个 PostgreSQL（或主从 PG 集群），Raft 日志和快照存储在数据库中（raft_logs 表），也可使用本地磁盘（推荐本地磁盘 + 数据库双写）。

4.4 单机到集群的平滑扩展

· 配置从 standalone 改为 cluster 时，需要初始化 Raft 集群并将现有数据导入。提供迁移工具。
· 生产环境建议直接部署集群模式。

---

5. 数据库设计（PostgreSQL）

核心表：

```sql
CREATE TABLE agents (
    id UUID PRIMARY KEY,
    name TEXT,
    announce_ip INET,
    announce_port INT,
    internal_ips INET[],
    location TEXT, -- 地区代码
    role TEXT, -- 'access' or 'relay'
    groups TEXT[],
    last_seen TIMESTAMP,
    config JSONB
);

CREATE TABLE group_policies (
    id SERIAL PRIMARY KEY,
    src_group TEXT,
    dst_group TEXT,
    allowed BOOLEAN
);

CREATE TABLE probe_results (
    agent_a UUID,
    agent_b UUID,
    latency_ms REAL,
    bandwidth_mbps REAL,
    loss_rate REAL,
    measured_at TIMESTAMP,
    PRIMARY KEY (agent_a, agent_b, measured_at)
);

CREATE TABLE routes (
    src UUID,
    dst UUID,
    path_nodes UUID[], -- 节点序列
    is_active BOOLEAN,
    weight INT, -- 权重百分比
    created_at TIMESTAMP,
    PRIMARY KEY (src, dst)
);

CREATE TABLE raft_logs ( -- 集群模式使用
    index BIGINT PRIMARY KEY,
    term BIGINT,
    data BYTEA
);
```

---

6. 接口定义（概要）

6.1 主控 API（REST + WebSocket）

端点 方法 说明
/api/v1/register POST Agent 注册，携带自定义 IP
/api/v1/agents GET 获取 Agent 列表
/api/v1/probe POST 要求两个 Agent 互探
/api/v1/route GET 查询当前路径
/api/v1/route/force PUT 设置手动档
/api/v1/policy/group POST 修改分组策略
/ws/control WebSocket 实时推送策略、路径更新

6.2 Agent 内部接口

· TUN 设备：三层 IP 隧道（L3），使用 tun-tap crate。
· 多路复用会话管理器：维护到其他 Agent 的连接池。
· 流量统计：每 60 秒上报主控。

---

7. 实现建议（Rust 技术栈）

· 异步运行时：tokio（含 tokio-tungstenite, tokio-postgres）
· QUIC：quinn
· 多路复用：TCP + yamux 或 QUIC 原生流
· 加密：ring（AES-GCM），RustCrypto（SM2/SM4）
· NAT 穿透：stun-rs
· TUN 设备：tun-tap
· Raft：openraft
· 配置解析：serde_yaml
· HTTP 框架：axum
· 数据库连接：sqlx（异步，支持 PostgreSQL）

---

8. 非功能性要求

· 性能：单主控支持 5000 Agent 同时在线，路径计算延迟 < 200ms。
· 内存：5000 Agent 状态 < 256 MB。
· 可靠性：集群模式下 Raft 自动故障转移；单机模式支持数据库恢复。
· 安全性：所有控制信令 TLS 1.3 或 QUIC 加密。

---
