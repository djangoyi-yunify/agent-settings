# Redis 7 AppPack 实施总控计划

## 背景

koda-apppacks Redis 已经具备完整的设计文档：

- `redis/docs/research/kubeblocks-redis-addon-research.md`
- `redis/docs/research/koda-pod-env-projection-realtime-and-restart.md`
- `redis/docs/design/redis-background.md`
- `redis/docs/design/redis-config-rewrite.md`
- `redis/docs/design/redis-technical-approach.md`

本 change 作为 Redis 7 AppPack 实施的总控路线图，将上述设计转化为分阶段的实施计划，并追踪各阶段 change 的进度与依赖关系。

## 范围

- 交付 Redis 7.2 AppPack，支持 **主从复制 + 哨兵（Replication + Sentinel）** 拓扑。
- 不 standalone，不 Redis Cluster，不 Twemproxy。
- 不实现备份恢复、Grafana dashboard、GitHub Actions CI workflow（后续版本补充）。

## 实施策略

采用 **1 个准备阶段 + 7 个功能迭代** 的方式推进：

```
阶段 0: 基础骨架（准备）
迭代 1: Bootstrap / 创建
迭代 2: 角色感知与服务发现
迭代 3: 扩缩容
迭代 4: 配置变更
迭代 5: 主从切换与故障转移
迭代 6: 业务账号管理（deferred）
迭代 7: 示例整理与文档编写
```

每个功能迭代同时覆盖 Redis Server 与 Sentinel，按生命周期垂直切片。

## 关键设计决策

1. **按生命周期垂直切片**：每个迭代同时覆盖 Server 和 Sentinel，避免先做完一个组件再回头适配另一个组件。
2. **测试左移**：每个迭代都有单元测试、集成测试，部分迭代有 e2e 验收。
3. **启动方式**：采用主容器 entrypoint 方式启动 Redis Server 与 Sentinel。
4. **配置策略**：Redis Server 配置通过 Koda 模板 + `/data/redis.conf` include 方式管理；Sentinel 配置完全由启动脚本生成在持久卷中。
5. **Sentinel 副本固定**：Sentinel 固定 3 副本，本期不支持扩缩容。

## 非目标

- 不实现 Redis Cluster 拓扑。
- 不实现备份恢复（PITR、ActionSet 等）。
- 不实现 Grafana dashboard。
- 不实现业务账号管理（iteration 6 deferred）。
- 不立即建立 GitHub Actions CI workflow。

## 相关规格

本期能力规格见 `openspec/specs/redis-7/`：

- `topology.md`
- `bootstrap.md`
- `service-discovery.md`
- `scaling.md`
- `reconfigure.md`
- `failover.md`
- `examples.md`
