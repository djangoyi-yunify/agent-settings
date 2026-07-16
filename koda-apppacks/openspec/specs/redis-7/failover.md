# Redis 7 AppPack 主从切换与故障转移规范

本规范定义计划内主从切换和自动故障转移的行为。

## ADDED Requirements

### Requirement: 支持计划内主从切换
Redis Server SHALL 提供 `switchover` 动作，
通过触发 Sentinel failover 实现计划内主从切换。

#### Scenario: 无 candidate 切换
- **WHEN** 用户执行 switchover OpsTask 且不指定 candidate
- **THEN** 系统向 Sentinel 发送 `SENTINEL FAILOVER <master-name>`
- **AND** Sentinel 自主选择并提升新 primary
- **AND** 原 replica 重新配置 replicaof 指向新 primary

#### Scenario: 有 candidate 切换
- **WHEN** 用户执行 switchover OpsTask 并指定 candidate
- **THEN** 系统验证 candidate 当前为 secondary
- **AND** 将所有 Server 节点 replica-priority 设置为 100，candidate 设置为 1
- **AND** 触发 `SENTINEL FAILOVER <master-name>`
- **AND** 等待新 primary 变为 candidate
- **AND** 恢复所有节点 replica-priority

### Requirement: 支持自动故障转移
Redis Server 和 Sentinel SHALL 支持原生自动故障转移，
当 primary 不可用时 Sentinel 自动提升新 primary。

#### Scenario: primary Pod 被删除
- **WHEN** primary Redis Server Pod 被删除或不可用
- **THEN** Sentinel 判定主观下线（SDOWN）并达成客观下线（ODOWN）
- **AND** Sentinel 选举 leader
- **AND** Leader Sentinel 向选中的 replica 发送 `SLAVEOF NO ONE`
- **AND** 其他 replica 重新配置 replicaof 指向新 primary

#### Scenario: failover 后 writer service 切换
- **WHEN** failover 完成后
- **THEN** roleProbe 在下一个周期检测到新 primary 的角色变化
- **AND** Koda 更新 Pod 标签
- **AND** `writer` service 路由到新 primary

### Requirement: switchover 不绕过 Sentinel
Redis Server 的 switchover 动作 SHALL 通过 Sentinel 的 failover 机制完成，
不直接修改 Redis 节点的复制关系。

#### Scenario: 执行 switchover
- **WHEN** switchover 动作执行
- **THEN** 只与 Sentinel 交互
- **AND** 不直接对 Redis Server 执行 `SLAVEOF NO ONE`

### Requirement: 故障转移可能导致数据丢失
Redis 7 AppPack SHALL 在文档中说明异步复制下故障转移可能丢失未同步的数据。

#### Scenario: 文档说明
- **WHEN** 用户阅读 Redis 7 AppPack README
- **THEN** README 中 SHALL 包含故障转移限制说明
- **AND** 说明异步复制下可能丢失未同步的写入数据
