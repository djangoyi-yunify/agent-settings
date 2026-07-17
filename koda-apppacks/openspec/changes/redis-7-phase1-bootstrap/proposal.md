# Redis 7 AppPack Bootstrap / 创建

## 背景

Redis 7 AppPack 实施计划见 `openspec/changes/redis-7-master-plan/`。
本 change 是实施路线图中的第 1 阶段（迭代 1），目标是在首次创建时让 Redis Server 与 Redis Sentinel 能独立启动并形成 Replication + Sentinel 拓扑。

## 范围

本 change 只处理**首次创建**场景，不涉及 Pod 重启场景。

### 包含

- Redis Server 启动脚本：首次创建时生成 `/data/redis.conf`、初始化 ACL、确定主从角色
- Redis Sentinel 启动脚本：首次创建时生成 `/data/sentinel/redis-sentinel.conf`
- 系统账号初始化：`default`、`replica`、`redis-sentinel`
- 跨组件密码引用
- 脚本层面的 TLS 开关识别：
  - 从环境变量读取 TLS 是否开启
  - `redis-cli` 调用时根据 TLS 开关设置正确参数

### 不包含

- Pod 重启时的配置局部更新（保留 `sentinel monitor`、`known-replica`、`known-sentinel`、epoch 等运行时状态，仅更新 announce-ip/port 等动态参数）—— deferred 到后续涉及 Pod 重建的阶段
- 启动脚本生成 TLS 配置参数（`tls-port`、`tls-cert-file` 等由 Koda 配置模板渲染，通过 `include /etc/conf/redis.conf` 引入）
- 实际 TLS 能力验证（TLS 开关始终关闭）
- 业务账号管理（由 `redis-7-phase6-account-management` 处理）
- `accountProvision`

## 验收标准

| 层级 | 验收项 |
|---|---|
| 单元测试 | Server / Sentinel 启动脚本的 shellspec 测试通过；`shellcheck` 无 error |
| 集成测试 | 在 kind 中创建 Application，Redis Server 与 Sentinel 均正常启动；`redis-cli INFO replication` 显示主从关系正确 |
| e2e | 通过 primary service 写入数据，在 replica 上能读取；Sentinel 返回的主节点地址与 primary service 一致 |

## 依赖

- 总控计划：`openspec/changes/redis-7-master-plan/`
- 前置 change：`redis-7-phase0-skeleton`
