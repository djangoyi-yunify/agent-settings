## ADDED Requirements

### Requirement: Redis Server 首次创建时以 Pod-0 为 primary
Redis Server 启动脚本 SHALL 在首次创建时以序号为 0 的 Pod 作为 primary，
其他 Pod 配置 `replicaof` 指向 Pod-0 的 `replica-announce-ip` 和 `replica-announce-port`。

#### Scenario: Pod-0 作为 primary
- **WHEN** 序号为 0 的 Redis Server Pod 首次创建
- **THEN** 启动脚本不配置 `replicaof`
- **AND** 该 Pod 以 primary 身份启动

#### Scenario: 其他 Pod 作为 replica
- **WHEN** 序号大于 0 的 Redis Server Pod 首次创建
- **THEN** 启动脚本获取 Pod-0 的 `replica-announce-ip` 和 `replica-announce-port`
- **AND** 配置 `replicaof <pod-0-announce-ip> <pod-0-announce-port>`

### Requirement: Redis Server 在首次创建时初始化系统账号
Redis Server SHALL 在首次创建时通过 `/data/users.acl` 初始化系统账号，
包括 `default`、`replica` 和 `redis-sentinel`。

#### Scenario: 系统账号初始化
- **WHEN** Redis Server Pod 首次创建
- **THEN** 启动脚本生成 `/data/users.acl`
- **AND** `default` 用户拥有全部权限
- **AND** `replica` 用户拥有复制所需权限
- **AND** `redis-sentinel` 用户拥有 Sentinel 访问 Server 所需权限

### Requirement: 系统账号密码通过环境变量注入
Redis Server 和 Redis Sentinel 的系统账号密码 SHALL 通过 Koda 的 `credentialFieldRef` 注入环境变量，
启动脚本读取环境变量生成 ACL。

#### Scenario: 密码注入
- **WHEN** Redis Server Pod 创建
- **THEN** 环境变量 `REDIS_DEFAULT_PASSWORD` 注入 default 账号密码
- **AND** 环境变量 `REDIS_SENTINEL_PASSWORD` 注入 Sentinel 组件 default 密码
- **AND** 启动脚本使用这些环境变量生成 ACL 规则

### Requirement: 明确系统账号与密码来源映射
Redis Server 和 Redis Sentinel 的系统账号 SHALL 有明确的密码来源，
并在启动脚本中通过环境变量获取。

#### Scenario: Redis Server 系统账号密码来源
- **WHEN** Redis Server Pod 首次创建
- **THEN** `default` 用户密码来自本组件 initAccount `default`
- **AND** `replica` 用户密码复用 `default` 用户密码
- **AND** `redis-sentinel` 用户密码来自 Sentinel 组件 initAccount `default`

#### Scenario: Redis Sentinel 自身认证密码来源
- **WHEN** Redis Sentinel Pod 首次创建
- **THEN** `sentinel sentinel-pass` 使用本组件 initAccount `default` 密码
- **AND** 该密码与 Server 上 `redis-sentinel` 用户密码一致

### Requirement: Redis Server 首次创建时生成非 TLS 运行时配置
Redis Server SHALL 在首次创建时通过启动脚本在 `/data/redis.conf` 生成非 TLS 运行时配置，
并通过 `include` 引入只读配置模板。

#### Scenario: Redis Server 首次创建
- **WHEN** Redis Server Pod 首次创建
- **THEN** 启动脚本生成 `/data/redis.conf`
- **AND** 写入 `include /etc/conf/redis.conf`
- **AND** 写入 `port`
- **AND** 写入 `replica-announce-ip` 和 `replica-announce-port`
- **AND** 根据当前角色写入或跳过 `replicaof`
- **AND** 写入 `masteruser replica` 和 `masterauth`
- **AND** 写入 `aclfile /data/users.acl`

### Requirement: Redis Sentinel 首次创建时生成运行时配置
Redis Sentinel SHALL 在首次创建时通过启动脚本在 `/data/sentinel/redis-sentinel.conf` 生成运行时配置。

#### Scenario: Sentinel 首次创建
- **WHEN** Redis Sentinel Pod 首次创建
- **THEN** 启动脚本创建 `/data/sentinel/redis-sentinel.conf`
- **AND** 写入动态参数：announce-ip、announce-port、port
- **AND** 写入 `sentinel sentinel-user` 和 `sentinel sentinel-pass`

### Requirement: 采用主容器 entrypoint 方式启动
Redis Server 和 Redis Sentinel SHALL 使用主容器 entrypoint 脚本启动，
entrypoint 负责配置生成和进程启动。

#### Scenario: 首次创建时启动
- **WHEN** Redis Server 或 Sentinel Pod 首次创建
- **THEN** entrypoint 脚本执行配置生成
- **AND** 最后通过 `exec` 启动 redis-server 或 redis-sentinel 进程

### Requirement: 脚本识别 TLS 开关并适配 redis-cli 参数
Redis Server 和 Redis Sentinel 的脚本 SHALL 通过环境变量 `REDIS_TLS_ENABLED` 识别 TLS 是否开启，
并在调用 `redis-cli` 时设置正确的 TLS 参数。

#### Scenario: TLS 关闭时使用非 TLS redis-cli
- **WHEN** `REDIS_TLS_ENABLED` 为 `"false"`
- **THEN** 脚本调用 `redis-cli` 时不追加 `--tls` 及相关证书参数

#### Scenario: TLS 开启时使用 TLS redis-cli
- **WHEN** `REDIS_TLS_ENABLED` 为 `"true"`
- **THEN** 脚本调用 `redis-cli` 时追加 `--tls --cert <cert> --key <key> --cacert <ca>`

### Requirement: 启动脚本不生成 TLS 配置参数
Redis Server 和 Redis Sentinel 的启动脚本 SHALL 不生成任何 TLS 配置参数（如 `tls-port`、`tls-cert-file` 等），
这些参数由 Koda 配置模板渲染并通过 `include /etc/conf/redis.conf` 引入。

#### Scenario: 生成 /data/redis.conf 时不写入 TLS 参数
- **WHEN** Redis Server Pod 首次创建
- **THEN** 启动脚本生成 `/data/redis.conf`
- **AND** 不写入 `tls-port`、`tls-cert-file`、`tls-key-file`、`tls-ca-cert-file` 等参数
- **AND** 通过 `include /etc/conf/redis.conf` 引入模板渲染的配置

### Requirement: 本期不验证实际 TLS 连接能力
本 change SHALL 不验证 Redis Server 和 Sentinel 的实际 TLS 连接能力，
集成测试和 e2e 测试中 `REDIS_TLS_ENABLED` 始终为 `"false"`。

#### Scenario: 集成测试中 TLS 关闭
- **WHEN** 执行本 change 的集成测试和 e2e 测试
- **THEN** `REDIS_TLS_ENABLED` 设置为 `"false"`
- **AND** 不执行 TLS 连接验证
