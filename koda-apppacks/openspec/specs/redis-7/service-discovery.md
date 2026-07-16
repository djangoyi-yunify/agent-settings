# Redis 7 AppPack 服务发现规范

本规范定义 Redis 7 AppPack 的角色感知、健康探测、服务导出及 Sentinel 注册机制。

## ADDED Requirements

### Requirement: Koda 能识别 Redis Server 当前角色
Redis Server SHALL 提供 `roleProbe`，向 Koda 报告当前 Pod 是 `primary` 还是 `secondary`。

#### Scenario: primary 角色探测
- **WHEN** roleProbe 在 Redis primary Pod 上执行
- **THEN** roleProbe 查询 `INFO replication` 的 `role` 字段
- **AND** 输出 `primary`

#### Scenario: secondary 角色探测
- **WHEN** roleProbe 在 Redis replica Pod 上执行
- **THEN** roleProbe 查询 `INFO replication` 的 `role` 字段
- **AND** 输出 `secondary`

### Requirement: Koda 能判断 Redis Server 和 Sentinel 是否健康
Redis Server 和 Redis Sentinel SHALL 分别提供 `availableProbe`，
供 Koda 判断组件是否可用。

#### Scenario: Redis Server 健康探测
- **WHEN** availableProbe 在 Redis Server Pod 上执行
- **THEN** 使用 redis-cli 执行 `PING`
- **AND** 返回 `PONG` 时输出 `alive`

#### Scenario: Redis Sentinel 健康探测
- **WHEN** availableProbe 在 Redis Sentinel Pod 上执行
- **THEN** 使用 redis-cli 连接 sentinel 端口执行 `PING`
- **AND** 返回 `PONG` 时输出 `alive`

### Requirement: Redis primary 向 Sentinel 注册监控关系
Redis Server 的 `postProvision` 动作 SHALL 在 primary Pod 上执行，
向所有 Sentinel 注册 `SENTINEL monitor`。

#### Scenario: 首次注册监控
- **WHEN** Redis Server 组件首次变为 Ready 且存在 primary Pod
- **THEN** postProvision 在所有 Sentinel 实例上执行 `SENTINEL monitor`
- **AND** 设置 quorum、down-after-milliseconds、failover-timeout、parallel-syncs
- **AND** 设置 auth-user 和 auth-pass

#### Scenario: Sentinel 未就绪时重试
- **WHEN** postProvision 执行时 Sentinel 尚未就绪
- **THEN** 脚本 SHALL 重试连接 Sentinel
- **AND** 直到 Sentinel 就绪后再执行注册

### Requirement: 暴露客户端所需服务
Redis 7 AppPack SHALL 导出以下服务供客户端使用：
- `redis`：指向所有 Redis Server Pod
- `writer`：通过 roleSelector 只指向当前 primary
- `redis-sentinel`：指向所有 Sentinel Pod

#### Scenario: 写操作连接
- **WHEN** 客户端通过 `writer` service 连接 Redis
- **THEN** 连接 SHALL 被路由到当前 primary

#### Scenario: Sentinel 发现主节点
- **WHEN** 客户端通过 `redis-sentinel` service 查询当前 primary
- **THEN** 客户端能获取当前 Redis primary 的地址

### Requirement: Redis Server 能获取 Sentinel 信息
Redis Server SHALL 通过 Koda 的跨组件字段引用获取 Sentinel 的组件名、Pod FQDN 列表和密码。

#### Scenario: 跨组件环境变量注入
- **WHEN** Redis Server Pod 创建
- **THEN** 环境变量 `SENTINEL_COMPONENT_NAME` 注入 Sentinel 组件名
- **AND** 环境变量 `SENTINEL_POD_FQDN_LIST` 注入 Sentinel Pod FQDN 列表
- **AND** 环境变量 `REDIS_SENTINEL_PASSWORD` 注入 Sentinel default 密码
