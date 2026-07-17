# Redis 7 AppPack 业务账号管理

## 背景

Redis 7 AppPack 实施计划见 `openspec/changes/redis-7-master-plan/`。
本 change 是实施路线图中的第 6 阶段，目标是为 Redis Server 实现业务账号管理能力，
通过 `accountProvision` 动作创建、更新、删除业务账号。

## 范围

### 包含

- 业务账号 `app` 的 statements 设计（create/update/delete）
- Redis Server `accountProvision` 脚本实现
- 更新 `componentdefinition-redis-server.yaml` 的 credential 配置
- 配置 `accountProvision.targetPodSelector: All`
- 与 `memberJoin` sync-acl 的集成验证

### 不包含

- Sentinel 业务账号管理
- initAccount 管理（由 phase1-bootstrap 处理）

## 验收标准

| 层级 | 验收项 |
|---|---|
| 单元测试 | accountProvision 脚本 shellspec 测试通过；`shellcheck` 无 error |
| 集成测试 | 在 kind 中通过 accountProvision 创建业务账号；`ACL LIST` 在所有 Server Pod 上正确 |
| e2e | 使用业务账号连接 Redis 写入/读取；scale-out 后新 replica 上业务账号可用 |

## 依赖

- 总控计划：`openspec/changes/redis-7-master-plan/`
- 前置 change：`redis-7-phase2-service-discovery`
