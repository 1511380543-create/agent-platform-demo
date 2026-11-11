# 智能体平台演示文档

Temporal 持久编排 + LangGraph 执行内核 + warrant 去中心化授权 + 双线审批（自主/管理员）+ 代码沙箱（配额与计量）。

## 智能体演示平台架构图

```mermaid
flowchart TB
  %% 布局约定（改图前先读）：
  %% 1. 连线只连"层"（subgraph），不要连层内的节点，否则层内 direction 失效，整图会错位。
  %% 2. 层内节点之间没有连线时：direction TB 让节点横排，direction LR 让节点竖排。
  %% 3. "~~~" 是不可见连线，只用来固定上下位置，不代表调用关系。

  subgraph FE["前端 · React（按租户策略切换界面）"]
    direction TB
    F1["会话线 UI：聊天 / 思考流 / 步骤时间线 / 审批弹窗"]
    F2["任务线 UI：数据确认弹窗 / 审批状态卡 / 任务预览页"]
    F3["管理面：任务审批 / 执行链路 / 用量计费 / 用户与角色"]
  end

  subgraph GW["网关 · 三服务汇聚 + 权限门"]
    direction TB
    G1["conversation-api / task-api / admin-api"]
    G2["RBAC（7+ 权限点）· CSRF · 密码/私钥双模式登录"]
    G3["管理员跨租户：审批队列 / 链路 / 审计"]
  end

  subgraph ORCH["编排层 · Temporal"]
    direction TB
    O1["SessionWorkflow（会话线·自主审批）"]
    O2["TaskApprovalWorkflow（任务线·管理员审批）"]
    O3["HITL 原语 / Saga 补偿 / continue-as-new"]
  end

  subgraph EXEC["执行层 · AgentWorker（4 池）"]
    direction TB
    E1["LangGraph：react 图 + lab/race 自定义图"]
    E2["工具治理链：审批包装 → warrant 过滤 → 参数策略"]
    E3["checkpoint 复用（中断恢复）+ 心跳"]
  end

  subgraph AUTH["授权层 · capability-service"]
    direction LR
    A1["warrant：每跳重签 / 委派深度 1 / 三点验签"]
    A2["审批决议签名（request_hash 绑定）· KMS + JWKS"]
    A3["审计（20 类事件）· 撤销集"]
  end

  subgraph TOOL["工具与沙箱"]
    direction LR
    T1["MCP tools：x-warrant 验签 + 白名单"]
    T2["sandbox-control：E2B（Firecracker microVM）"]
    T3["租户配额（并发/时长/内存 签进凭证）+ 沙箱计量"]
  end

  subgraph STATE["状态与记忆 · P3 坐标制"]
    direction LR
    S1["raw_events 基座（MySQL）+ Redis 热窗口 + seq 分配器"]
    S2["语义记忆：recall / summarize / extract"]
    S3["etcd 定义目录 · MinIO claim-check"]
  end

  subgraph OBS["观测层"]
    direction TB
    B1["执行链路页（attempt/耗时/凭证摘要·脱敏）"]
    B2["litellm /metrics → 租户 token 计费"]
    B3["沙箱 /metrics（9109）→ 沙箱账"]
  end

  subgraph K8S["平台 · k8s"]
    direction TB
    K1["KEDA：按 Temporal 队列深度扩缩 worker"]
    K2["服务间凭证签名（SigV4 家族）"]
    K3["Prometheus / Loki / MySQL / Redis"]
  end

  %% 主调用链：自上而下，逐层调用
  FE --> GW --> ORCH --> EXEC
  EXEC --> AUTH
  EXEC --> TOOL
  EXEC --> STATE

  %% 下面三行的先后顺序决定"授权 / 工具 / 状态"三列的左右次序，不要调换。
  %% 其中只有"工具与沙箱 --> 观测层"是真实调用，两条 ~~~ 仅用于让观测层居中。
  AUTH ~~~ OBS
  TOOL --> OBS
  STATE ~~~ OBS

  %% 仅用于定位：让平台层排在最下方
  OBS ~~~ K8S
```



## 智能体演示平台说明

平台通过两个任务进行能力展示：

1.算法可视化：输入算法名称和数据，系统生成可视化步骤（只支持三种数据结构：线性结构，图，树。）

2.排序大乱斗：输入数组，执行预设的五个排序算法对比性能，并生成表格与可视化步骤

平台目的在于展示 ： 任务中断，任务恢复，审批，租户权限，Agent协作与治理，观测计量  

可通过默认卡片快速开始。演示入口https://web.taild04b47.ts.net/console/

登录后页面如下。如果无法访问，可在面试时现场演示。

![image-20261111185111352](/Users/a15804298237/Library/Application Support/typora-user-images/image-20261111185111352.png)

## 演示账号

| 账号    | 密码     | 角色                   |
| ------- | -------- | ---------------------- |
| admin   | admin123 | 管理员（t-dev）        |
| alice   | alice123 | 一级租户普通用户       |
| vizuser | viz12345 | 二级租户（任务审批线） |

admin：管理员：对话（算法可视化，排序大乱斗），查看Agent目录，计费用量，执行链路，审批任务

alice：一级租户：对话（算法可视化，排序大乱斗），自主审批使用代码沙箱

vizuser：二级租户：对话（算法可视化），管理员审批使用代码沙箱（对应企业中任务关键节点审批），

​                                   预览任务进度。

## 演示截图（可跟随截图步骤操作）

### alice（一级租户）登录后自主审批使用代码沙箱。

| 沙箱执行弹出自助审批 | 审批通过后运行结果 |
|---|---|
| ![自主审批](picture/一级租户自主审批图.png) | ![运行结果](picture/一级租户审批后运行结果.png) |

### 中断与恢复

| 生成中点「停止」 | 中断后界面（继续入口） |
|---|---|
| ![主动中断](picture/一级租户任务主动中断.png) | ![中断后样式](picture/一级租户任务中断后界面样式.png) |

| 点「继续」恢复执行 | 恢复后执行结果 |
|---|---|
| ![恢复过程](picture/一级租户任务中断后恢复过程.png) | ![恢复结果](picture/一级租户中断恢复执行结果.png) |

> 恢复 = LangGraph checkpoint 续跑：图态与审批位置原样续接；Temporal workflow 在 interrupt 处暂停，resume 信号精确唤醒。

### vizuser（二级租户）登录后使用代码沙箱需要管理员审批

| 系统状态卡：已提交管理员审批 | 任务预览页：进度  / 结果 |
|---|---|
| ![等待审批](picture/二级租户等待管理员审批重要功能.png) | ![预览界面](picture/二级租户查看任务预览.png) |

| 切换admin管理员账号审批任务 | 任务预览页：查看结果 |
|---|---|
| ![管理员审批任务](picture/管理员审批任务.png) | ![查看结果](picture/二级租户查看任务结果.png) |

### 管理员视角

| 任务审批看板 | 批准 / 驳回操作 |
|---|---|
| ![审批界面](picture/管理员任务审批界面.png) | ![审批任务](picture/管理员审批任务.png) |

| 执行链路（route/agent/审批/llm/沙箱） | 智能体授权凭证展示（脱敏摘要） |
|---|---|
| ![执行链路](picture/管理员查看执行链路.png) | ![授权凭证](picture/管理员查看执行链路-智能体授权凭证展示.png) |

| agent 定义目录（版本化）可看到运行时所属执行池 | 租户用量计费 |
|---|---|
| ![agent目录](picture/管理员查看agent目录.png) | ![租户计费](picture/管理员查看租户计费.png) |

