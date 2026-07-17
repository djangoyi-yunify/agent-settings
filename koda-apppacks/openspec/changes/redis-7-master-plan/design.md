# Redis 7 AppPack 实施总控设计

## 1. 变更拆分策略

Redis 7 AppPack 实施拆分为多个独立 change：

- 一个总控 change：`redis-7-master-plan`（本 change）
- 多个阶段 change，每个 change 完成一个可独立验收的里程碑

```
redis-7-master-plan
        │
        ├──▶ redis-7-phase0-skeleton
        │           └── 迭代 1-6 的基础依赖
        │
        ├──▶ redis-7-phase1-bootstrap
        │           └── 依赖：phase0-skeleton
        │
        ├──▶ redis-7-phase2-service-discovery
        │           └── 依赖：phase1-bootstrap
        │
        ├──▶ redis-7-phase6-account-management
        │           └── 依赖：phase2-service-discovery
        │
        ├──▶ redis-7-phase3-scaling
        │           └── 依赖：phase2-service-discovery、phase6-account-management
        │
        ├──▶ redis-7-phase4-reconfigure
        │           └── 依赖：phase2-service-discovery
        │
        ├──▶ redis-7-phase5-failover
        │           └── 依赖：phase2-service-discovery
        │
        └──▶ redis-7-phase7-docs-examples
                    └── 依赖：phase1-bootstrap、phase3-scaling、phase4-reconfigure、
                        phase5-failover、phase6-account-management
```

## 2. 阶段 change 清单

| 迭代 | change 名称 | 目标 | 关键输入 | 关键输出 | 依赖 |
|---|---|---|---|---|---|
| 0 | `redis-7-phase0-skeleton` | 搭建最小可运行可测试的 chart 与 CRD 骨架 | 设计文档 | 可渲染 chart、可 apply 的空壳 CRD、测试入口 | 无 |
| 1 | `redis-7-phase1-bootstrap` | Server 与 Sentinel 在首次创建时能独立启动并形成复制拓扑（Pod 重启场景 deferred） | phase0 骨架 | 启动脚本、ACL 初始化、运行时配置生成 | phase0 |
| 2 | `redis-7-phase2-service-discovery` | 角色感知、健康探测、服务注册与导出 | phase1 bootstrap | roleProbe、availableProbe、postProvision、services | phase1 |
| 3 | `redis-7-phase3-scaling` | Server 水平扩缩容 | phase2 服务发现、phase6 账号管理 | memberJoin sync-acl、扩缩容 OpsTask | phase2、phase6 |
| 4 | `redis-7-phase4-reconfigure` | Server 参数热更新 | phase2 服务发现 | reconfigure 脚本、热更新参数清单 | phase2 |
| 5 | `redis-7-phase5-failover` | 计划内切换与自动故障转移 | phase2 服务发现 | switchover 脚本、failover 验证 | phase2 |
| 6 | `redis-7-phase6-account-management` | 业务账号管理（accountProvision） | phase2 服务发现 | accountProvision 脚本、业务账号声明 | phase2 |
| 7 | `redis-7-phase7-docs-examples` | 示例整理与文档编写 | phase1/3/4/5/6 | README、Application 示例、OpsTask 示例、账号管理示例 | phase1/3/4/5/6 |

## 3. 阶段 4 与阶段 5 的并行性

`redis-7-phase4-reconfigure` 和 `redis-7-phase5-failover` 都依赖 `redis-7-phase2-service-discovery`，
但彼此之间没有强依赖，可以并行开发。

```
phase2-service-discovery
        │
        ├──▶ phase6-account-management
        │         │
        │         ▼
        ├──▶ phase3-scaling
        │
        ├──▶ phase4-reconfigure
        │
        └──▶ phase5-failover
```

## 4. 规格工作流

本项目采用 OpenSpec 常规工作流：

1. 每个阶段 change 创建自己的 delta spec（`openspec/changes/<change>/specs/redis-7/<capability>.md`）
2. change 实施完成后，通过 `openspec archive` 将 delta spec 合并到主规格（`openspec/specs/redis-7/`）
3. 主规格随各阶段 change 的完成逐步填充，而不是预先写完整

### 当前状态

- `openspec/specs/redis-7/` 当前为空，等待各阶段 change archive 后填充
- 设计草案已备份到 `/tmp/redis-7-specs-draft/`，供创建 change 时参考

### 各阶段 change 的 delta spec 规划

| 阶段 change | 主要生成的 capability delta |
|---|---|
| phase0-skeleton | 无（只搭骨架） |
| phase1-bootstrap | bootstrap |
| phase2-service-discovery | service-discovery |
| phase3-scaling | scaling |
| phase4-reconfigure | reconfigure |
| phase5-failover | failover |
| phase6-account-management | account-management |
| phase7-docs-examples | examples |

所有阶段 change 都应在 delta spec 中引用 topology 约束。

## 5. 状态追踪

本 design 文档维护各阶段 change 的实施状态：

| 阶段 | change 名称 | 状态 | 备注 |
|---|---|---|---|
| 0 | `redis-7-phase0-skeleton` | 待开始 | |
| 1 | `redis-7-phase1-bootstrap` | 已创建 | 依赖 phase0 |
| 2 | `redis-7-phase2-service-discovery` | 待开始 | 依赖 phase1 |
| 3 | `redis-7-phase3-scaling` | 待开始 | 依赖 phase2、phase6 |
| 4 | `redis-7-phase4-reconfigure` | 待开始 | 依赖 phase2 |
| 5 | `redis-7-phase5-failover` | 待开始 | 依赖 phase2 |
| 6 | `redis-7-phase6-account-management` | 待开始 | 依赖 phase2 |
| 7 | `redis-7-phase7-docs-examples` | 待开始 | 依赖 phase1/3/4/5/6 |

状态说明：
- 待开始：尚未创建 change 或尚未开始实施
- 进行中：已创建 change 并正在实施
- 已完成：change 已验证并通过
- 已归档：change 已完成并归档

## 6. 风险与缓解

| 风险 | 影响 | 缓解 |
|---|---|---|
| 阶段 change 边界不清 | 中 | 每个 change 有明确的验收标准，避免范围蔓延 |
| 跨阶段依赖导致阻塞 | 中 | 总控 change 持续追踪状态，及时调整优先级 |
| Koda 跨组件 env 注入不支持 | 高 | phase2 服务发现前优先验证 credentialFieldRef 和 componentFieldRef |
| kind 环境无法模拟真实故障转移 | 中 | failover 测试在已有 k8s 集群上补充验证 |

## 7. GitHub Issue 跟踪

本项目使用 GitHub issue 和 sub-issue 跟踪 Redis 7 AppPack 的实施计划和各阶段变更的概要、状态变化。

### 7.1 Issue 与 change 的映射关系

每个阶段 change 对应一个 GitHub issue。总控 change 对应一个顶层 issue，用于汇总整体进度。

| 迭代 | change 名称 | GitHub Issue | 状态 |
|---|---|---|---|
| 总控 | `redis-7-master-plan` | #223 | 已创建 |
| 0 | `redis-7-phase0-skeleton` | #329 | 已创建 |
| 1 | `redis-7-phase1-bootstrap` | #330 | 已创建 |
| 2 | `redis-7-phase2-service-discovery` | #331 | 已创建 |
| 3 | `redis-7-phase3-scaling` | #332 | 已创建 |
| 4 | `redis-7-phase4-reconfigure` | #333 | 已创建 |
| 5 | `redis-7-phase5-failover` | #334 | 已创建 |
| 6 | `redis-7-phase6-account-management` | #339 | 已创建 |
| 7 | `redis-7-phase7-docs-examples` | #335 | 已创建 |

### 7.2 Issue 内容模板

#### 顶层 issue（总控）

- **标题**：`Redis 7 AppPack 实施路线图`
- **标签**：`redis`, `apppack`, `roadmap`
- **正文**：
  - 项目背景与范围
  - 阶段划分与依赖关系
  - 各阶段 issue 链接
  - 当前整体状态

#### 阶段 issue

- **标题**：`Redis 7 AppPack <阶段名>`，例如 `Redis 7 AppPack 基础骨架搭建`
- **标签**：`redis`, `apppack`, `implementation`
- **正文**：
  - 本阶段目标
  - 关联 change 路径
  - 引用的顶层规格
  - 验收标准
  - sub-issue 列表（由具体任务拆分）

### 7.3 Sub-issue 拆分原则

每个阶段 issue 可根据 tasks.md 中的任务拆分为 sub-issue，建议按以下维度拆分：

- 按组件：Server、Sentinel、公共脚本
- 按工作类型：开发、测试、文档
- 按交付物：chart、CRD、示例、Makefile

### 7.4 状态更新规则

- issue 创建时状态为 `Open`
- change 开始实施时，issue 状态保持 `Open`，并在评论中更新进度
- change 完成并通过验收后，关闭 issue
- 如某阶段被阻塞，在 issue 中添加 `blocked` 标签并说明原因

### 7.5 与 OpenSpec 的关系

GitHub issue 是项目管理跟踪工具，OpenSpec change 是规格与实施计划载体。二者关系：

- OpenSpec change 负责记录 WHAT、HOW、WHEN
- GitHub issue 负责跟踪执行状态、汇总变更概要、便于团队协作
- 状态变化优先在 OpenSpec design.md 中更新，然后同步到 GitHub issue
