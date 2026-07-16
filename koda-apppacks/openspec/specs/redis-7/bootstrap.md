# Redis 7 AppPack Bootstrap 规范

本规范定义 Redis Server 和 Redis Sentinel 的启动、初始化及系统账号管理要求。

## ADDED Requirements

### Requirement: Redis Server 能自动确定主从角色并启动
Redis Server 启动脚本 SHALL 根据 Sentinel 查询结果或默认规则确定自身是 primary 还是 replica，
并生成正确的运行时配置。

#### Scenario: Sentinel 已监控 primary
- **WHEN** Redis Server Pod 启动时 Sentinel 已经监控到当前 primary
- **THEN** 启动脚本查询 Sentinel 获取当前 primary 地址
- **AND** 如果自身不是 primary，则配置 replicaof 指向该 primary
- **AND** 如果自身是 primary，则不配置 replicaof

#### Scenario: Sentinel 尚未监控 primary
- **WHEN** 首次创建集群且 Sentinel 尚未监控任何 primary
- **THEN** 启动脚本回退到按 Pod 字典序最小者作为默认 primary
- **AND** 其他 Pod 配置 replicaof 指向该默认 primary

### Requirement: Redis Server 使用 ACL 文件初始化系统账号
Redis Server SHALL 在启动时通过 `/data/users.acl` 初始化系统账号，
包括 `default`、`replica` 和 `redis-sentinel`。

#### Scenario: 系统账号初始化
- **WHEN** Redis Server Pod 启动
- **THEN** 启动脚本生成或更新 `/data/users.acl`
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

### Requirement: Redis Sentinel 启动时生成运行时配置
Redis Sentinel SHALL 通过启动脚本在 `/data/sentinel/redis-sentinel.conf` 生成或更新运行时配置，
保留已有的监控状态和运行时状态。

#### Scenario: Sentinel 首次启动
- **WHEN** Redis Sentinel Pod 首次启动
- **THEN** 启动脚本创建 `/data/sentinel/redis-sentinel.conf`
- **AND** 写入动态参数：announce-ip、announce-port、port
- **AND** 写入 `sentinel sentinel-user` 和 `sentinel sentinel-pass`

#### Scenario: Sentinel 重启
- **WHEN** Redis Sentinel Pod 重启
- **THEN** 启动脚本 surgical update 配置文件
- **AND** 保留已有的 `sentinel monitor`、`known-replica`、`known-sentinel` 和 epoch
- **AND** 更新动态参数

### Requirement: 采用主容器 entrypoint 方式启动
Redis Server 和 Redis Sentinel SHALL 使用主容器 entrypoint 脚本启动，
entrypoint 负责配置生成和进程启动。

#### Scenario: Pod 启动
- **WHEN** Redis Server 或 Sentinel Pod 启动
- **THEN** entrypoint 脚本执行配置生成
- **AND** 最后通过 `exec` 启动 redis-server 或 redis-sentinel 进程
