# Redis 7 AppPack 示例与文档规范

本规范定义 Redis 7 AppPack 交付的示例、文档及端到端走查要求。

## ADDED Requirements

### Requirement: 提供 Application 创建示例
Redis 7 AppPack SHALL 提供完整的 Application 创建示例，
包括 namespace、application 和 kustomization。

#### Scenario: 用户安装并创建 Redis Application
- **WHEN** 用户按照 README 安装 Redis 7 AppPack
- **THEN** 用户能找到 `redis/examples/application/` 下的示例
- **AND** 示例包含 3+3 拓扑的 Application 配置
- **AND** 用户能直接 apply 这些示例

### Requirement: 提供运维任务示例
Redis 7 AppPack SHALL 为每种运维操作提供 OpsTask 示例。

#### Scenario: 用户执行运维操作
- **WHEN** 用户需要执行 switchover、reconfigure 或扩缩容
- **THEN** 用户能在 `redis/examples/opstasks/` 下找到对应示例
- **AND** 示例包括 switchover.yaml、reconfigure.yaml、horizontal-scale-out.yaml、horizontal-scale-in.yaml

### Requirement: README 覆盖安装、使用与运维
Redis 7 AppPack SHALL 提供 `redis/README.md`，
包含安装、创建 Application、连接、扩缩容、切换、热更新和故障转移说明。

#### Scenario: 新用户阅读 README
- **WHEN** 用户首次使用 Redis 7 AppPack
- **THEN** README SHALL 说明如何安装 AppPack
- **AND** 说明如何创建 3+3 拓扑的 Application
- **AND** 提供通过 writer service 写入和通过 redis service 读取的示例
- **AND** 说明扩缩容、切换、热更新的使用方式
- **AND** 说明故障转移限制和拓扑约束

### Requirement: 端到端走查验证交付物
Redis 7 AppPack 交付前 SHALL 完成端到端走查，
验证安装、创建、读写、切换、扩缩容、热更新和故障转移全流程。

#### Scenario: 交付前走查
- **WHEN** 进入迭代 7 文档整理阶段
- **THEN** 按照 README 完整操作一遍
- **AND** 验证安装 AppPack、创建 Application、写入/读取数据
- **AND** 验证执行 switchover、scale out/in、reconfigure
- **AND** 验证删除 primary Pod 后自动 failover 业务恢复
