---
name: SRv6 SDN Architecture Design
description: Draw.io logical architecture design for SRv6-based SDN lab prototype with control plane, data plane, business layer, and observability
type: design
date: 2026-04-11
---

# SRv6 SDN 架构设计文档

## 1. 项目概述

### 1.1 目标
为 SRv6 / 可编程网络方向的 SDN 实验室原型设计一套完整的逻辑架构图，使用 draw.io 实现，用于项目落地实施。

### 1.2 范围
本设计包含三张架构图，优先级顺序为：
1. **逻辑架构图**（本文档重点）
2. 部署拓扑图（后续）
3. 业务/流量流程图（后续）

### 1.3 技术栈假设
- 控制平面：SRv6 控制器 + 路径计算引擎
- 数据平面：SRv6 节点（Ingress/Transit/Egress）+ PE/vRouter/Linux Router
- 南向协议：NETCONF / gNMI / PCEP / BGP SR Policy
- 北向接口：REST / gRPC
- 观测：Streaming Telemetry / gNMI / sFlow / IPFIX

### 1.4 设计粒度
**实施版**：细化到接口/协议级，适合后续落地拆解和组件开发。

---

## 2. 逻辑架构总体设计

### 2.1 分层结构
采用 **四层 + 右侧观测域** 的架构：

```
┌─────────────────────────────────────────────────────┐
│              业务接入层                              │
└─────────────────────────────────────────────────────┘
                      ↓ REST / gRPC
┌─────────────────────────────────────────────────────┐  ┌──────────────┐
│            控制与编排层                              │  │   观测与     │
│  Intent → Topology DB → Path Compute → Orchestrator │←→│   运维层     │
└─────────────────────────────────────────────────────┘  └──────────────┘
                 ↓ NETCONF / gNMI / PCEP                        ↑
┌─────────────────────────────────────────────────────┐         │
│              数据平面                                │─────────┘
│  Ingress → Transit → Egress → PE/vRouter            │  Telemetry
└─────────────────────────────────────────────────────┘
```

### 2.2 设计原则
1. **控制与转发分离**：控制器只负责计算和下发，不参与实际转发
2. **意图驱动**：业务层提交意图，控制层翻译成网络策略
3. **闭环反馈**：观测层数据反馈到控制层，支持动态路径调整
4. **接口标准化**：南向支持多协议适配，北向统一 API Gateway

---

## 3. 各层详细设计

### 3.1 业务接入层

#### 3.1.1 组件清单
| 组件名称 | 职责 | 接口 |
|---------|------|------|
| Portal / Web UI | 可视化服务申请、策略配置、状态查看 | HTTPS → API Gateway |
| Northbound API Gateway | 统一北向入口、认证路由 | REST / gRPC → Intent Manager |
| Automation / CI Pipeline | 自动化编排、批量下发、实验流程驱动 | REST / gRPC → API Gateway |
| Tenant / Service Request | 业务意图输入、SLA 约束、服务链需求 | 逻辑输入 → Intent Manager |

#### 3.1.2 关键接口
- **Portal → API Gateway**
  - 协议：HTTPS / REST
  - 内容：服务开通请求、策略查询、状态监控
  
- **Automation → API Gateway**
  - 协议：REST / gRPC
  - 内容：批量配置、实验场景编排、自动化测试触发

#### 3.1.3 设计要点
- 所有外部请求必须经过 API Gateway，不允许直接访问控制核心
- 业务层只提交"意图"，不涉及具体网络实现细节
- 支持多租户隔离和权限管理

---

### 3.2 控制与编排层

#### 3.2.1 组件清单

##### 3.2.1.1 Intent / Policy Manager
**职责**：
- 接收业务意图并解析
- 将高层需求转换为网络策略
- 定义带宽、时延、路径或服务链约束
- 触发路径计算流程

**输入**：
- 业务服务请求（来自 API Gateway）
- SLA 约束（时延、带宽、可靠性）
- 服务链需求（经过哪些 Service Function）

**输出**：
- 策略约束对象 → Path Computation Engine
- 策略记录 → Topology & State DB

**关键接口**：
- 北向：REST / gRPC（接收 API Gateway 请求）
- 东向：Read/Write（访问 Topology & State DB）
- 南向：Policy Constraints（向 Path Computation Engine 提交约束）

---

##### 3.2.1.2 Topology & State DB
**职责**：
- 存储网络拓扑（节点、链路、能力）
- 维护 SID 资源分配信息
- 保存链路/节点实时状态
- 记录策略和路径历史

**数据模型**：
- 节点：ID、类型（Ingress/Transit/Egress）、SID 前缀、能力
- 链路：源节点、目标节点、带宽、时延、丢包率、TE Metric
- SID：SID 值、类型（End/End.X/End.DT）、绑定节点/链路
- 策略：Policy ID、约束条件、路径结果、状态
- 状态：实时性能指标、告警事件、历史趋势

**关键接口**：
- 北向：Read/Write API（供 Intent Manager、Path Computation Engine 访问）
- 南向：State Sync（接收 Telemetry Collector 的状态更新）
- 东向：BGP-LS / IGP（可选，用于自动拓扑发现）

---

##### 3.2.1.3 Path Computation Engine
**职责**：
- 根据拓扑和约束计算 SRv6 路径
- 生成 SID List 或 Segment List
- 输出 SR Policy / TE Policy
- 支持约束路径计算（CSPF）

**算法支持**：
- 最短路径（Dijkstra）
- 约束最短路径（CSPF）
- 流量工程优化（TE）
- 服务链路径计算（SFC）

**输入**：
- 策略约束（来自 Intent Manager）
- 拓扑和状态（来自 Topology & State DB）
- 路径质量反馈（来自 Telemetry / OAM）

**输出**：
- SID List / Segment List
- SR Policy（Color、Endpoint、Candidate Path）
- TE Policy（显式路径、动态路径）

**关键接口**：
- 北向：Policy Constraints（接收 Intent Manager 的约束）
- 东向：Topology / Metrics Read（访问 Topology & State DB）
- 南向：SID List / TE Policy（输出到 SRv6 Service Orchestrator）
- 反馈：Path Quality Feedback（接收 OAM 的路径质量数据）

---

##### 3.2.1.4 SRv6 Service Orchestrator
**职责**：
- 将路径计算结果转换为设备级配置
- 编排多节点配置下发顺序
- 管理服务开通与变更流程
- 处理配置回滚和异常恢复

**工作流程**：
1. 接收 Path Computation Engine 的路径结果
2. 查询 Topology DB 获取节点详细信息
3. 生成设备级配置任务（Ingress 封装、Transit 转发、Egress 解封装）
4. 按顺序下发配置到 Southbound Adapter
5. 监控配置结果并处理异常

**输出**：
- 设备配置任务（JSON / YANG 模型）
- 配置下发指令

**关键接口**：
- 北向：SID List / TE Policy（接收 Path Computation Engine 输出）
- 东向：Read（访问 Topology & State DB）
- 南向：Device Intent / Config Push（向 Southbound Adapter 下发）

---

##### 3.2.1.5 Southbound Adapter
**职责**：
- 适配多种南向协议
- 将控制指令转换为设备可理解的格式
- 屏蔽底层设备差异
- 处理设备连接和会话管理

**支持协议**：
- **NETCONF**：标准配置管理协议
- **gNMI**：Google Network Management Interface
- **PCEP**：Path Computation Element Protocol（用于 TE 路径下发）
- **BGP SR Policy**：通过 BGP 分发 SR Policy
- **SSH / Ansible**：传统设备配置方式
- **Container API / K8s API**：容器化节点控制

**关键接口**：
- 北向：Device Intent / Config Push（接收 Orchestrator 指令）
- 南向：NETCONF / gNMI / PCEP / SSH（连接数据平面节点）

---

#### 3.2.2 控制层内部数据流

```
API Gateway
    ↓ REST / gRPC
Intent / Policy Manager
    ↓ Policy Constraints          ↔ Topology & State DB (Read/Write)
Path Computation Engine
    ↓ SID List / TE Policy        ↔ Topology & State DB (Read)
SRv6 Service Orchestrator
    ↓ Device Intent / Config Push ↔ Topology & State DB (Read)
Southbound Adapter
    ↓ NETCONF / gNMI / PCEP
Data Plane Nodes
```

---

### 3.3 数据平面

#### 3.3.1 组件清单

##### 3.3.1.1 SRv6 Ingress Node
**职责**：
- 接收业务流量
- 根据策略匹配流量
- 封装 SRv6 头部（插入 SID List）
- 将流量导入指定 SRv6 路径

**关键能力**：
- 流量分类（基于五元组、DSCP、VRF）
- SRv6 封装（SRH 插入）
- Steering（流量导向）

**配置来源**：
- Southbound Adapter（NETCONF / gNMI）

---

##### 3.3.1.2 SRv6 Transit Node
**职责**：
- 按 SRv6 头部信息转发流量
- 执行 Segment 处理（如 End、End.X）
- 承担中间路径承载功能

**关键能力**：
- SRv6 转发表查找
- Segment 处理（SID 弹出、更新）
- 流量工程（TE Metric）

---

##### 3.3.1.3 SRv6 Egress Node
**职责**：
- 接收 SRv6 流量
- 解封装 SRv6 头部
- 将流量交付到目标端
- 执行出口策略

**关键能力**：
- SRv6 解封装
- 流量交付（到 VRF、Namespace、Service Function）

---

##### 3.3.1.4 PE / vRouter / Linux Router
**职责**：
- 作为实验室原型中的实际承载节点
- 执行转发表、策略路由和 SRv6 功能
- 支持 IPv6 + SRv6 转发

**实现方式**：
- 物理设备（支持 SRv6 的路由器）
- 虚拟路由器（VPP、FRRouting）
- Linux 内核（Linux 5.10+ 原生 SRv6 支持）
- 容器化路由器（Docker / K8s）

---

##### 3.3.1.5 Service Function / Workload
**职责**：
- 承载实际业务服务
- 作为 SRv6 Service Chaining 的服务节点
- 处理业务流量

**示例**：
- 防火墙
- 负载均衡器
- DPI（深度包检测）
- 应用容器

---

#### 3.3.2 数据平面流量路径

```
业务流量
    ↓
SRv6 Ingress Node（封装 SRv6 头部，插入 SID List）
    ↓ IPv6 + SRv6
SRv6 Transit Node（按 SID 转发）
    ↓ IPv6 + SRv6
Service Function（可选，服务链场景）
    ↓ IPv6 + SRv6
SRv6 Egress Node（解封装 SRv6 头部）
    ↓
目标业务端
```

---

### 3.4 观测与运维层

#### 3.4.1 组件清单

##### 3.4.1.1 Telemetry Collector
**职责**：
- 汇聚节点和链路状态
- 接收性能、告警和日志数据
- 提供统一采集接口

**支持协议**：
- **Streaming Telemetry**：实时推送
- **gNMI**：订阅模式
- **sFlow / IPFIX**：流量采样
- **Syslog**：日志采集
- **SNMP**：传统监控（可选）

**数据来源**：
- 数据平面所有节点
- 控制器自身状态

---

##### 3.4.1.2 Metrics / Logs Store
**职责**：
- 保存时序数据和日志
- 支持回溯分析和可视化
- 提供查询接口

**技术选型建议**：
- 时序数据库：Prometheus / InfluxDB / TimescaleDB
- 日志存储：Elasticsearch / Loki
- 对象存储：MinIO / S3（长期归档）

---

##### 3.4.1.3 Fault / Alert Manager
**职责**：
- 识别故障和 SLA 异常
- 告警聚合和去重
- 向控制层回传重规划触发信号

**告警类型**：
- 链路故障
- 节点不可达
- 性能劣化（时延、丢包）
- SLA 违约

**输出**：
- 告警通知（邮件、Webhook、Slack）
- 事件触发（向 Intent Manager 发送 Policy Trigger）

---

##### 3.4.1.4 Visualization Dashboard
**职责**：
- 展示路径状态、链路质量、告警信息
- 提供拓扑视图
- 支持实验观测

**展示内容**：
- 网络拓扑图（节点、链路、SID）
- 路径状态（活跃路径、流量分布）
- 性能指标（时延、带宽、丢包率）
- 告警列表

**技术选型建议**：
- Grafana（通用监控看板）
- 自定义 Web UI（基于 D3.js / ECharts）

---

##### 3.4.1.5 OAM / Trace / Ping Module
**职责**：
- 执行网络诊断与验证
- 校验 SRv6 路径是否符合预期
- 主动探测链路质量

**功能**：
- **SRv6 Ping**：验证 SID 可达性
- **SRv6 Traceroute**：追踪路径跳数
- **OAM（Operations, Administration, and Maintenance）**：链路质量探测
- **BFD（Bidirectional Forwarding Detection）**：快速故障检测

**触发方式**：
- 定期探测
- 按需触发（故障排查）
- 控制器主动验证（路径下发后）

---

#### 3.4.2 观测层数据流

```
Data Plane Nodes
    ↓ Streaming Telemetry / gNMI / sFlow / Syslog
Telemetry Collector
    ↓
Metrics / Logs Store
    ↓
Visualization Dashboard（展示）
    ↓
Fault / Alert Manager（异常检测）
    ↓ Policy Trigger / Fault Event
Intent / Policy Manager（触发重规划）
```

---

## 4. 接口与协议汇总

### 4.1 北向接口（业务接入层 → 控制层）
| 源 | 目标 | 协议 | 内容 |
|----|------|------|------|
| Portal / Web UI | API Gateway | HTTPS / REST | 服务申请、策略配置、状态查询 |
| Automation / CI | API Gateway | REST / gRPC | 批量配置、自动化编排 |
| API Gateway | Intent Manager | REST / gRPC | 业务意图、SLA 约束 |

### 4.2 控制层内部接口
| 源 | 目标 | 协议 | 内容 |
|----|------|------|------|
| Intent Manager | Topology & State DB | Read/Write | 策略对象、资源查询 |
| Intent Manager | Path Computation Engine | Policy Constraints | 约束条件 |
| Path Computation Engine | Topology & State DB | Read | 拓扑、链路状态、SID 资源 |
| Path Computation Engine | Orchestrator | SID List / TE Policy | 路径结果 |
| Orchestrator | Topology & State DB | Read | 节点详细信息 |
| Orchestrator | Southbound Adapter | Device Intent / Config Push | 设备配置任务 |

### 4.3 南向接口（控制层 → 数据平面）
| 源 | 目标 | 协议 | 内容 |
|----|------|------|------|
| Southbound Adapter | PE / vRouter / Linux Router | NETCONF | 配置管理 |
| Southbound Adapter | PE / vRouter / Linux Router | gNMI | 配置 + 状态订阅 |
| Southbound Adapter | PE / vRouter / Linux Router | PCEP | TE 路径下发 |
| Southbound Adapter | PE / vRouter / Linux Router | BGP SR Policy | SR Policy 分发 |
| Southbound Adapter | PE / vRouter / Linux Router | SSH / Ansible | 传统配置方式 |

### 4.4 观测接口（数据平面 → 观测层）
| 源 | 目标 | 协议 | 内容 |
|----|------|------|------|
| Data Plane Nodes | Telemetry Collector | Streaming Telemetry | 实时性能数据 |
| Data Plane Nodes | Telemetry Collector | gNMI | 状态订阅 |
| Data Plane Nodes | Telemetry Collector | sFlow / IPFIX | 流量采样 |
| Data Plane Nodes | Telemetry Collector | Syslog | 日志 |

### 4.5 反馈接口（观测层 → 控制层）
| 源 | 目标 | 协议 | 内容 |
|----|------|------|------|
| Telemetry Collector | Topology & State DB | State Sync / Metrics Update | 状态同步 |
| Fault / Alert Manager | Intent Manager | Policy Trigger / Fault Event | 故障事件、重规划触发 |
| OAM / Analytics | Path Computation Engine | Path Quality Feedback | 路径质量反馈 |

---

## 5. Draw.io 实施指南

### 5.1 画布设置
- **尺寸**：A3 横向（420mm × 297mm）或更大
- **网格**：启用网格，间距 10px
- **对齐**：启用自动对齐

### 5.2 图形元素

#### 5.2.1 容器框（分层）
- **形状**：Rectangle（Container）
- **样式**：
  - 业务接入层：蓝色边框（#1976d2），浅蓝背景（#e3f2fd）
  - 控制与编排层：橙色边框（#f57c00），浅橙背景（#fff3e0）
  - 数据平面：绿色边框（#388e3c），浅绿背景（#e8f5e9）
  - 观测与运维层：粉色边框（#c2185b），浅粉背景（#fce4ec）
- **边框宽度**：2px
- **圆角**：4px

#### 5.2.2 模块框
- **形状**：Rectangle
- **样式**：
  - 边框：深灰色（#424242），1px
  - 背景：白色（#ffffff）
  - 圆角：4-6px
- **文字**：
  - 主标题：14pt，加粗
  - 副标题：10pt，常规，灰色（#666666）

#### 5.2.3 连接线
- **控制下发**：
  - 样式：实线箭头
  - 颜色：蓝色（#1976d2）
  - 宽度：2px
  
- **状态反馈**：
  - 样式：虚线箭头
  - 颜色：橙色（#ff9800）
  - 宽度：1.5px
  
- **业务流量**：
  - 样式：粗实线箭头
  - 颜色：绿色（#4caf50）
  - 宽度：3px

#### 5.2.4 协议标注
- **形状**：Label（直接贴在箭头上）
- **样式**：
  - 字体：10pt
  - 背景：白色，半透明
  - 边框：无

### 5.3 布局建议

#### 5.3.1 垂直分层
```
┌─────────────────────────────────────────────┐
│  业务接入层（高度：15%）                     │
├─────────────────────────────────────────────┤
│  控制与编排层（高度：40%）                   │
├─────────────────────────────────────────────┤
│  数据平面（高度：30%）                       │
└─────────────────────────────────────────────┘
```

#### 5.3.2 右侧观测域
- 宽度：画布宽度的 20%
- 高度：与控制层 + 数据平面对齐
- 位置：右侧独立区域

#### 5.3.3 模块间距
- 同层模块间距：20px
- 层间间距：30px
- 容器内边距：15px

### 5.4 导出设置
- **格式**：PNG（高分辨率，300 DPI）或 SVG
- **背景**：白色
- **边距**：10px
- **文件命名**：`srv6-sdn-logical-architecture.drawio`

---

## 6. 后续架构图规划

### 6.1 部署拓扑图
**目标**：展示节点、链路、设备与组件的物理/虚拟部署位置

**内容**：
- 实验室网络拓扑
- 节点 IP 地址和 SID 分配
- 链路带宽和连接关系
- 控制器部署位置
- 观测组件部署位置

### 6.2 业务/流量流程图
**目标**：展示策略下发、路径计算、转发流程的时序关系

**内容**：
- 服务开通流程（从业务请求到路径生效）
- 路径计算流程（拓扑查询 → 约束计算 → 结果下发）
- 转发流程（Ingress 封装 → Transit 转发 → Egress 解封装）
- 故障恢复流程（故障检测 → 告警 → 重规划 → 路径切换）

---

## 7. 设计验证清单

### 7.1 完整性检查
- [ ] 所有必需模块已包含（业务接入、控制编排、数据平面、观测运维）
- [ ] 所有关键接口已定义（北向、南向、东西向、反馈）
- [ ] 所有协议已标注（REST、gRPC、NETCONF、gNMI、PCEP 等）

### 7.2 一致性检查
- [ ] 模块职责无重叠
- [ ] 接口方向正确（控制下发 vs 状态反馈）
- [ ] 数据流闭环完整（业务请求 → 控制下发 → 转发执行 → 状态反馈）

### 7.3 可实施性检查
- [ ] 每个模块都有明确的输入输出
- [ ] 每个接口都有具体的协议和数据格式
- [ ] 技术栈选型合理（SRv6、NETCONF、gNMI 等）

### 7.4 可扩展性检查
- [ ] 支持新增节点和链路
- [ ] 支持新增南向协议适配
- [ ] 支持新增观测数据源

---

## 8. 参考资料

### 8.1 SRv6 相关
- RFC 8986: Segment Routing over IPv6 (SRv6) Network Programming
- RFC 9256: Segment Routing Policy Architecture
- draft-ietf-spring-srv6-network-programming

### 8.2 南向协议
- RFC 6241: NETCONF Protocol
- gNMI Specification (OpenConfig)
- RFC 5440: PCEP (Path Computation Element Protocol)
- draft-ietf-idr-segment-routing-te-policy (BGP SR Policy)

### 8.3 观测协议
- gNMI Telemetry
- RFC 3176: sFlow
- RFC 7011: IPFIX

---

## 9. 附录

### 9.1 术语表
- **SRv6**：Segment Routing over IPv6
- **SID**：Segment Identifier
- **SR Policy**：Segment Routing Policy
- **TE**：Traffic Engineering
- **CSPF**：Constrained Shortest Path First
- **OAM**：Operations, Administration, and Maintenance
- **BFD**：Bidirectional Forwarding Detection
- **NETCONF**：Network Configuration Protocol
- **gNMI**：gRPC Network Management Interface
- **PCEP**：Path Computation Element Protocol

### 9.2 版本历史
- v1.0 (2026-04-11)：初始版本，完成逻辑架构设计

---

**文档状态**：待审核  
**下一步**：用户审核 → 实施计划编写 → draw.io 图绘制
