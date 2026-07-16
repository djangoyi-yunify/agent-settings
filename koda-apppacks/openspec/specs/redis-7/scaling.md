# Redis 7 AppPack 扩缩容规范

本规范定义 Redis Server 水平扩缩容及 Sentinel 副本固定策略。

## ADDED Requirements

### Requirement: Redis Server 支持水平扩容
Redis Server SHALL 支持通过 OpsTask 水平增加副本数，
新副本 SHALL 自动加入复制拓扑。

#### Scenario: 执行 scale-out
- **WHEN** 用户执行 horizontal-scale-out OpsTask 增加 Redis Server 副本数
- **THEN** Koda 创建新的 Redis Server Pod
- **AND** 新 Pod 启动脚本查询 Sentinel 获取当前 primary
- **AND** 新 Pod 配置 replicaof 指向当前 primary
- **AND** 新 Pod 加入主从复制链路

### Requirement: 新副本同步非 init 账号
Redis Server 的 `memberJoin` 动作 SHALL 在新副本可用后执行，
从已有 peer 节点同步非 init 账号到新副本。

#### Scenario: scale-out 后账号同步
- **WHEN** 新 Redis Server Pod 变为 Available
- **THEN** Koda 触发 memberJoin 动作
- **AND** 脚本从其他 Server Pod 获取 ACL LIST
- **AND** 仅同步非 init 账号（跳过 default、replica、redis-sentinel）
- **AND** 执行 ACL SAVE 持久化

#### Scenario: 无 peer 可同步时跳过
- **WHEN** memberJoin 执行时无法从任何 peer 获取 ACL LIST
- **THEN** 脚本记录警告并正常退出
- **AND** 不阻塞 scale-out 完成

### Requirement: Redis Server 支持水平缩容
Redis Server SHALL 支持通过 OpsTask 水平减少副本数。

#### Scenario: 执行 scale-in
- **WHEN** 用户执行 horizontal-scale-in OpsTask 减少 Redis Server 副本数
- **THEN** Koda 删除多余的 Redis Server Pod
- **AND** 如果被删除的是 replica，Sentinel 视图中可能残留 disconnected replica
- **AND** 系统功能保持正常

### Requirement: 缩容时不清理 Sentinel known-replica
Redis Server SHALL 不实现 `memberLeave` 动作，
接受 Sentinel 中可能残留的 disconnected replica 记录。

#### Scenario: scale-in 后 Sentinel 视图
- **WHEN** scale-in 删除一个 replica 后
- **THEN** Sentinel 的 `known-replica` 列表中可能保留该 replica 的 disconnected 记录
- **AND** 系统不执行 `SENTINEL RESET` 清理
- **AND** 不影响后续读写和 failover

### Requirement: Redis Sentinel 不支持扩缩容
Redis Sentinel 组件 SHALL 固定为 3 个副本，
本期不提供 Sentinel 扩缩容的 OpsTask。

#### Scenario: 尝试修改 Sentinel 副本数
- **WHEN** 用户尝试对 Sentinel 组件执行 horizontal scaling
- **THEN** 系统拒绝该操作
- **AND** 提示 Sentinel 副本数固定为 3
