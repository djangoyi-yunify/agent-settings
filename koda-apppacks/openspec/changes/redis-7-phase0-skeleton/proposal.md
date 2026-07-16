# Redis 7 AppPack 基础骨架搭建

## 背景

Redis 7 AppPack 的实施计划见 `openspec/changes/redis-7-master-plan/`。
本 change 是实施路线图中的第 0 阶段，目标是搭建最小可运行、可测试的项目骨架，
为后续迭代提供基础。

## 范围

本 change 只关注基础骨架搭建，不实现任何 Redis 运行时功能：

- 创建目录结构
- 创建最小可渲染的 Helm chart
- 创建可 apply 的空壳 CRD（ApplicationDefinition、ComponentDefinition、ComponentMatrix、ConfigFileDefinition、AppPack）
- 创建最小配置模板
- 创建脚本 ConfigMap 占位
- 创建本地单元测试入口（shellspec、shellcheck、helm lint/template）
- 创建集成测试环境抽象层脚本
- 创建 Makefile

## 不在范围内

- lifecycle actions（roleProbe、availableProbe、postProvision、memberJoin、reconfigure、switchover）
- ACL 初始化逻辑
- 跨组件 env 引用
- 详细的配置 CUE 约束和 effect scope
- 完整的 opstasks 示例内容

## 验收标准

| 层级 | 验收项 |
|---|---|
| 单元测试 | `make test-unit` 成功运行；`shellcheck` 无 error；`helm lint` 通过；`helm template` 能渲染 |
| 集成测试 | 在 kind 或已有 k8s 集群中 `kubectl apply` 空壳 CRD 成功，不崩溃 |
| e2e | 无 |

## 依赖

- 总控计划：`openspec/changes/redis-7-master-plan/`
- 顶层规格：`openspec/specs/redis-7/topology.md`
