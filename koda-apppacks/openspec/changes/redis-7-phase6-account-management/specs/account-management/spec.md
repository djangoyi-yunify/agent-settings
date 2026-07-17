## ADDED Requirements

### Requirement: Redis Server 支持通过 accountProvision 创建业务账号
Redis Server SHALL 提供 `accountProvision` 生命周期动作，
通过 `ACL SETUSER` 和 `ACL SAVE` 创建、更新、删除业务账号。

#### Scenario: 创建业务账号
- **WHEN** 用户或系统触发 accountProvision 动作创建账号 `app`
- **THEN** 脚本在所有 Redis Server Pod 上执行 `ACL SETUSER app ...`
- **AND** 执行 `ACL SAVE` 持久化到 `/data/users.acl`
- **AND** 新 replica 加入集群时通过 `memberJoin` 同步该账号

#### Scenario: 更新业务账号
- **WHEN** 用户或系统触发 accountProvision 动作更新账号 `app` 的密码或权限
- **THEN** 脚本在所有 Redis Server Pod 上执行 `ACL SETUSER app ...`
- **AND** 执行 `ACL SAVE` 持久化变更

#### Scenario: 删除业务账号
- **WHEN** 用户或系统触发 accountProvision 动作删除账号 `app`
- **THEN** 脚本在所有 Redis Server Pod 上执行 `ACL DELUSER app`
- **AND** 执行 `ACL SAVE` 持久化变更

### Requirement: accountProvision 对所有 Redis Server Pod 生效
Redis Server 的 `accountProvision` 动作 SHALL 在**所有** Redis Server Pod 上执行，
因为 ACL 变更不会通过主从复制自动传播。

#### Scenario: 3 节点 Redis Server 创建账号
- **WHEN** 用户对 3 节点 Redis Server 执行创建账号操作
- **THEN** Koda 分别在 3 个 Pod 上执行 accountProvision 脚本
- **AND** 每个 Pod 的 `ACL LIST` 都包含该账号

### Requirement: 区分 initAccount 与业务账号
Redis Server 的启动脚本和 `accountProvision` 脚本 SHALL 区分系统账号和业务账号：

- 系统账号（`default`、`replica`、`redis-sentinel`）在首次创建时由启动脚本初始化
- 业务账号通过 `accountProvision` 动作管理

#### Scenario: 启动脚本不删除业务账号
- **WHEN** Redis Server Pod 首次创建
- **THEN** 启动脚本生成系统账号行
- **AND** 保留 `/data/users.acl` 中已存在的业务账号行（如果有）
- **AND** 不覆盖业务账号

### Requirement: 新 replica 自动同步业务账号
Redis Server 的 `memberJoin` 动作 SHALL 从 peer 节点同步业务账号到新加入的 replica。

#### Scenario: scale-out 后业务账号同步
- **WHEN** 新 Redis Server Pod 加入集群
- **THEN** `memberJoin` 从其他 Server Pod 获取 `ACL LIST`
- **AND** 仅同步非 init 业务账号
- **AND** 执行 `ACL SAVE` 持久化

### Requirement: Redis Sentinel 本期不实现业务账号管理
Redis Sentinel 组件 SHALL 不实现 `accountProvision` 动作，
Sentinel 自身只使用系统账号 `redis-sentinel` / `default` 进行认证。

#### Scenario: 对 Sentinel 执行 accountProvision
- **WHEN** 用户尝试对 Sentinel 组件执行账号管理操作
- **THEN** 系统拒绝该操作或不提供该动作
