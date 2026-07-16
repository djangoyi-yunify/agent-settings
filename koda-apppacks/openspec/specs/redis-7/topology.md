# Redis 7 AppPack 拓扑规范

本规范定义 Redis 7 AppPack 支持的拓扑结构及其基本约束。

## ADDED Requirements

### Requirement: 提供 Replication + Sentinel 拓扑
Redis 7 AppPack SHALL 提供名为 `replication-sentinel` 的拓扑，
包含 Redis Server 组件和 Redis Sentinel 组件。

#### Scenario: 创建 Application
- **WHEN** 用户创建一个 3+3 的 Redis Application
- **THEN** 系统创建 3 个 Redis Server 实例
- **AND** 系统创建 3 个 Redis Sentinel 实例
- **AND** Redis Server 形成 1 个 primary 和 2 个 replica 的复制关系
- **AND** Redis Sentinel 监控该 Redis Server primary

### Requirement: 不支持 Standalone、Redis Cluster 和 Twemproxy
Redis 7 AppPack 本期 SHALL 只支持 Replication + Sentinel 拓扑，
不支持 standalone、Redis Cluster 和 Twemproxy 等其他拓扑。

#### Scenario: 用户选择不支持的拓扑
- **WHEN** 用户尝试选择 standalone 或 cluster 拓扑
- **THEN** 系统不提供该选项或拒绝创建
- **AND** 提示用户当前版本仅支持 replication-sentinel 拓扑

### Requirement: Sentinel 副本数固定为 3
Redis Sentinel 组件的副本数 SHALL 固定为 3，
本期不支持水平扩缩容。

#### Scenario: 创建 Sentinel 组件
- **WHEN** 用户创建 Redis Application
- **THEN** Sentinel 组件的 replicas 固定为 3
- **AND** 用户无法通过 OpsTask 修改 Sentinel 副本数

### Requirement: Redis Server 最小副本数为 3
Redis Server 组件 SHALL 支持水平扩缩容，
但最小副本数 SHALL 为 3，以保证 1 primary + 2 replicas 的高可用结构。

#### Scenario: 创建 Redis Server 组件
- **WHEN** 用户创建 Redis Application
- **THEN** Redis Server 默认副本数为 3
- **AND** 系统阻止用户将副本数设置为小于 3
