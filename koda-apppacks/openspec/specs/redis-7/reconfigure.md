# Redis 7 AppPack 配置热更新规范

本规范定义 Redis Server 参数热更新能力及边界。

## ADDED Requirements

### Requirement: Redis Server 支持部分参数热更新
Redis Server SHALL 支持通过 OpsTask 对部分参数执行 `CONFIG SET` + `CONFIG REWRITE`，
无需重启 Pod。

#### Scenario: 热更新 maxmemory
- **WHEN** 用户执行 reconfigure OpsTask 修改 `maxmemory`
- **THEN** 系统在所有 Redis Server Pod 上执行 `CONFIG SET maxmemory <value>`
- **AND** 执行 `CONFIG REWRITE` 持久化到 `/data/redis.conf`
- **AND** Pod 不被重建

#### Scenario: 热更新参数清单
- **WHEN** 用户修改热更新参数清单中的任意参数
- **THEN** 系统 SHALL 支持热更新
- **AND** 初始热更新参数清单包括：maxmemory、maxmemory-policy、timeout、maxclients、tcp-keepalive、slowlog-log-slower-than、slowlog-max-len、lazyfree-lazy-eviction、lazyfree-lazy-expire、lazyfree-lazy-server-del

### Requirement: 非热更新参数需要重启生效
Redis Server SHALL 不支持对非热更新参数执行 CONFIG SET，
用户修改这类参数后 SHALL 通过模板更新触发 Pod 重建。

#### Scenario: 修改 port 参数
- **WHEN** 用户尝试修改 `port` 参数
- **THEN** 系统不执行 CONFIG SET
- **AND** 模板更新后触发 Redis Server Pod 重建

### Requirement: 热更新在所有 Server Pod 上独立执行
Redis Server 的 `reconfigure` 动作 SHALL 在每个 Server Pod 上独立执行，
因为 `CONFIG SET` 不会通过复制传播到 replica。

#### Scenario: 3 节点 Redis Server 热更新
- **WHEN** 用户对 3 节点 Redis Server 执行 reconfigure
- **THEN** Koda 分别在 3 个 Pod 上执行 reconfigure 脚本
- **AND** 每个 Pod 独立执行 CONFIG SET 和 CONFIG REWRITE

### Requirement: Sentinel 配置不开放用户变更
Redis Sentinel SHALL 不支持 `reconfigure` 动作，
Sentinel 配置参数由启动脚本和运行时自动管理。

#### Scenario: 尝试对 Sentinel reconfigure
- **WHEN** 用户尝试对 Sentinel 组件执行 reconfigure OpsTask
- **THEN** 系统拒绝该操作或不提供该动作
