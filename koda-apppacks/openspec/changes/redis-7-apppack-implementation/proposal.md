# Redis 7 AppPack 实施计划

## 背景

koda-apppacks Redis 已经具备完整的设计文档：

- `redis/docs/research/kubeblocks-redis-addon-research.md`
- `redis/docs/research/koda-pod-env-projection-realtime-and-restart.md`
- `redis/docs/design/redis-background.md`
- `redis/docs/design/redis-config-rewrite.md`
- `redis/docs/design/redis-technical-approach.md`

本 change 将这些设计转化为可执行的开发和验证计划。

## 范围

- 交付 Redis 7.2 AppPack，支持 **主从复制 + 哨兵（Replication + Sentinel）** 拓扑。
- 不 standalone，不 Redis Cluster，不 Twemproxy。
- 不实现备份恢复（后续版本补充）。

## 高层实施框架

采用 **1 个准备阶段 + 6 个功能迭代 + 1 个文档收尾迭代** 的方式推进。每个功能迭代同时覆盖 Redis Server 与 Sentinel，按生命周期垂直切片。

```
阶段 0: 基础骨架（准备）
├── 开发: 目录结构、Helm chart、空壳 CRD
│   - ApplicationDefinition
│   - ComponentDefinition (redis-7-server, redis-7-sentinel)
│   - ComponentMatrix
│   - ConfigFileDefinition（仅 Redis Server，Sentinel 配置由启动脚本生成）
│   - 脚本 ConfigMap 占位
├── 测试: 本地单元测试工具链
│   - shellspec
│   - shellcheck
│   - helm lint / helm template
└── 测试环境: 集群抽象层脚本
    - setup-cluster.sh（默认 kind，支持已有 k8s）
    - install-koda.sh
    - teardown-cluster.sh

迭代 1: Bootstrap / 创建
├── 开发: Redis Server 与 Sentinel 能独立启动
│   - redis.conf 模板 + 可写主配置生成
│   - sentinel.conf 由启动脚本生成 + surgical update
│   - initAccount 密码初始化（default 用户）
└── 测试: 单组件 Pod Ready，PING 通过

迭代 2: 角色感知与服务发现
├── 开发: 拓扑跑起来
│   - Server: roleProbe / availableProbe
│   - Server: 通过 Sentinel 发现当前主节点
│   - Sentinel: availableProbe / postProvision 注册 master
└── 测试: 3+3 拓扑部署成功，角色正确，Sentinel 已监控 master

迭代 3: 扩缩容
├── 开发: 水平扩容/缩容
│   - Server: memberJoin / memberLeave
│   - Sentinel: memberJoin / memberLeave
└── 测试: scale out/in，复制关系正确
    - 输出: horizontal-scale-out.yaml / horizontal-scale-in.yaml

迭代 4: 配置变更
├── 开发: 参数热更新
│   - Server: reconfigure（CONFIG SET + CONFIG REWRITE）
│   - Sentinel: reconfigure（如需要）
└── 测试: 热更新参数生效且持久化
    - 输出: reconfigure.yaml

迭代 5: 主从切换与故障转移
├── 开发: 计划内切换与自动故障转移
│   - Server: switchover 动作（触发 Sentinel FAILOVER）
│   - Sentinel: 原生 failover
└── 测试: switchover opstask + 自动 failover 业务恢复
    - 输出: switchover.yaml

迭代 6: 账号管理（非 initAccount）
├── 开发: 系统账号通过 accountProvision 创建
│   - Server: replication / sentinel 账号
│   - Sentinel: 如启用 ACL
└── 测试: ACL LIST 正确，业务账号可连接

迭代 7: 示例整理与文档编写
├── 开发: 对外交付物整理
│   - README.md
│   - examples/ 下的 Application 与 opstasks 示例
│   - opstasks 整理与说明
└── 测试: 端到端走查

可选后续:
└── .github/workflows/redis-test.yaml（CI 标准化，非当前急需）
```

## 关键设计决策

1. **按生命周期垂直切片**：每个迭代同时覆盖 Server 和 Sentinel，避免先做完一个组件再回头适配另一个组件。
2. **测试左移**：每个迭代都有单元测试、集成测试，部分迭代有 e2e 验收。
3. **测试环境灵活**：默认 kind，支持已有 k8s 集群；CI 使用 kind，但 GitHub Workflow 非当前急需。
4. **initAccount 在 Bootstrap 处理**：`default` 等 initAccount 在 Server 启动脚本中初始化，非 initAccount 走 `accountProvision` 动作。

## 非目标

- 不实现 Redis Cluster 拓扑。
- 不实现备份恢复（PITR、ActionSet 等）。
- 不实现 Grafana dashboard。
- 不立即建立 GitHub Actions CI workflow。

## 风险

| 风险 | 影响 | 缓解 |
|---|---|---|
| 跨组件 env 注入方案不确定 | 高 | 迭代 2 前先验证 Koda 双向 serviceDependency 是否可行 |
| kind 环境无法模拟真实故障转移 | 中 | e2e 可在已有 k8s 集群上补充验证 |
| Sentinel 与 Server 启动顺序依赖 | 中 | 脚本通过重试和查询机制解耦，不依赖固定启动顺序 |
| initAccount 密码注入时机 | 中 | 在 Bootstrap 脚本中显式处理，不依赖 accountProvision |
