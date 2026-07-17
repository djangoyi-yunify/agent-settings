# Redis 7 AppPack 拓扑规格

## ADDED Requirements

### Requirement: Redis 7 AppPack 提供 replication-sentinel 拓扑
Redis 7 AppPack SHALL 提供一个名为 `replication-sentinel` 的 ApplicationDefinition 拓扑。

#### Scenario: 拓扑名称可被 Application 引用
- **WHEN** Application 资源在 `spec.topology` 中指定为 `replication-sentinel`
- **THEN** Koda 控制器能够匹配到 Redis 7 的 ApplicationDefinition

### Requirement: replication-sentinel 拓扑包含 server 组件槽位
replication-sentinel 拓扑 SHALL 声明一个名为 `server` 的组件槽位，用于运行 Redis Server。

#### Scenario: ApplicationDefinition 声明 server 组件
- **WHEN** 用户查看 Redis 7 ApplicationDefinition 的 `spec.topologies[0].components`
- **THEN** 存在 `name` 为 `server` 的组件槽位

### Requirement: server 组件槽位绑定到 redis-7-server ComponentDefinition
`server` 组件槽位 SHALL 引用名为 `redis-7-server` 的 ComponentDefinition。

#### Scenario: 组件槽位指向正确的 ComponentDefinition
- **WHEN** 用户查看 `server` 组件槽位的 `componentDef` 字段
- **THEN** 其值为 `redis-7-server`

### Requirement: replication-sentinel 拓扑包含 sentinel 组件槽位
replication-sentinel 拓扑 SHALL 声明一个名为 `sentinel` 的组件槽位，用于运行 Redis Sentinel。

#### Scenario: ApplicationDefinition 声明 sentinel 组件
- **WHEN** 用户查看 Redis 7 ApplicationDefinition 的 `spec.topologies[0].components`
- **THEN** 存在 `name` 为 `sentinel` 的组件槽位

### Requirement: sentinel 组件槽位绑定到 redis-7-sentinel ComponentDefinition
`sentinel` 组件槽位 SHALL 引用名为 `redis-7-sentinel` 的 ComponentDefinition。

#### Scenario: 组件槽位指向正确的 ComponentDefinition
- **WHEN** 用户查看 `sentinel` 组件槽位的 `componentDef` 字段
- **THEN** 其值为 `redis-7-sentinel`

### Requirement: Server 组件暴露 redis 服务
`redis-7-server` ComponentDefinition SHALL 在 `contracts.services.exports` 中声明名为 `redis` 的服务，端口为 6379。

#### Scenario: Server 服务定义存在
- **WHEN** 用户查看 `redis-7-server` 的 `contracts.services.exports`
- **THEN** 存在 `name` 为 `redis` 的服务，且 `serviceSpec.ports[0].port` 等于 6379

### Requirement: Sentinel 组件暴露 sentinel 服务
`redis-7-sentinel` ComponentDefinition SHALL 在 `contracts.services.exports` 中声明名为 `sentinel` 的服务，端口为 26379。

#### Scenario: Sentinel 服务定义存在
- **WHEN** 用户查看 `redis-7-sentinel` 的 `contracts.services.exports`
- **THEN** 存在 `name` 为 `sentinel` 的服务，且 `serviceSpec.ports[0].port` 等于 26379

### Requirement: Server 组件声明 data 持久卷
`redis-7-server` ComponentDefinition SHALL 在 `contracts.storage.volumes` 中声明名为 `data` 的持久卷，挂载路径为 `/data`。

#### Scenario: Server 存储卷定义存在
- **WHEN** 用户查看 `redis-7-server` 的 `contracts.storage.volumes`
- **THEN** 存在 `name` 为 `data` 的卷，且 `mount.path` 等于 `/data`

### Requirement: Sentinel 组件声明 data 持久卷
`redis-7-sentinel` ComponentDefinition SHALL 在 `contracts.storage.volumes` 中声明名为 `data` 的持久卷，挂载路径为 `/data`。

#### Scenario: Sentinel 存储卷定义存在
- **WHEN** 用户查看 `redis-7-sentinel` 的 `contracts.storage.volumes`
- **THEN** 存在 `name` 为 `data` 的卷，且 `mount.path` 等于 `/data`

### Requirement: Server 组件声明 redis-config 配置模板
`redis-7-server` ComponentDefinition SHALL 在 `contracts.config.templates` 中声明名为 `redis-config` 的配置模板，并启用 `externalManaged`。

#### Scenario: Server 配置模板契约存在
- **WHEN** 用户查看 `redis-7-server` 的 `contracts.config.templates`
- **THEN** 存在 `name` 为 `redis-config` 的模板，且 `externalManaged` 为 true

### Requirement: Server 配置模板挂载到 /etc/conf
`redis-config` 配置模板 SHALL 挂载到 Server Pod 的 `/etc/conf` 路径。

#### Scenario: 配置模板卷挂载路径正确
- **WHEN** 用户查看 `redis-7-server` 的 `runtime.workload.podSpec.containers[0].volumeMounts`
- **THEN** 存在 `name` 为 `redis-config` 的挂载，且 `mountPath` 等于 `/etc/conf`

### Requirement: Server 脚本挂载到 /scripts
`redis-7-server` ComponentDefinition SHALL 将 Server 脚本 ConfigMap 挂载到 Pod 的 `/scripts` 路径。

#### Scenario: Server 脚本挂载路径正确
- **WHEN** 用户查看 `redis-7-server` 的 `runtime.workload.podSpec.containers[0].volumeMounts`
- **THEN** 存在 `name` 为 `redis-scripts` 的挂载，且 `mountPath` 等于 `/scripts`

### Requirement: Sentinel 脚本挂载到 /scripts
`redis-7-sentinel` ComponentDefinition SHALL 将 Sentinel 脚本 ConfigMap 挂载到 Pod 的 `/scripts` 路径。

#### Scenario: Sentinel 脚本挂载路径正确
- **WHEN** 用户查看 `redis-7-sentinel` 的 `runtime.workload.podSpec.containers[0].volumeMounts`
- **THEN** 存在 `name` 为 `sentinel-scripts` 的挂载，且 `mountPath` 等于 `/scripts`
