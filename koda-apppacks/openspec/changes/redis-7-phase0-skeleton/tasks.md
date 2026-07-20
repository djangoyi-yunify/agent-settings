# Redis 7 AppPack 基础骨架任务清单

## 目录与 chart 结构

- [x] 创建 `redis/apppack/chart/` 目录结构
- [x] 创建 `redis/apppack/chart/config/` 目录
- [x] 创建 `redis/apppack/chart/scripts/` 目录
- [x] 创建 `redis/apppack/chart/scripts-ut-spec/` 目录
- [x] 创建 `redis/apppack/chart/templates/` 目录
- [x] 创建 `redis/examples/application/` 目录结构
- [x] 创建 `redis/examples/opstasks/` 目录结构
- [x] 创建 `redis/test/hack/` 目录结构

## Helm chart 基础

- [x] 编写 `redis/apppack/chart/Chart.yaml`
- [x] 编写 `redis/apppack/chart/values.yaml`
- [x] 编写 `redis/apppack/chart/README.md`（最小说明）
- [x] 编写 `redis/apppack/chart/templates/_helpers.tpl`

## AppPack 与定义层 CRD

- [x] 编写 `redis/apppack/apppack.yaml`
- [x] 编写 `redis/apppack/chart/templates/applicationdefinition.yaml`
- [x] 编写 `redis/apppack/chart/templates/componentmatrix.yaml`
- [x] 编写 `redis/apppack/chart/templates/componentdefinition-redis-server.yaml`（骨架，lifecycle 留空）
- [x] 编写 `redis/apppack/chart/templates/componentdefinition-redis-sentinel.yaml`（骨架，lifecycle 留空）
- [x] 编写 `redis/apppack/chart/templates/configfiledefinition-redis-server.yaml`（骨架，managedParams 留空）

## 配置文件

- [x] 编写 `redis/apppack/chart/config/redis7-config.tpl`（最小模板）
- [x] 编写 `redis/apppack/chart/config/redis7-config-constraint.cue`（占位）
- [x] 编写 `redis/apppack/chart/config/redis-config-effect-scope.yaml`（占位）

## 配置模板 ConfigMap

- [x] 编写 `redis/apppack/chart/templates/redis-config-template.yaml`

## 脚本 ConfigMap 占位

- [x] 编写 `redis/apppack/chart/scripts/redis-start.sh`（占位，exec redis-server）
- [x] 编写 `redis/apppack/chart/scripts/redis-sentinel-start.sh`（占位，exec redis-server --sentinel）
- [x] 编写 `redis/apppack/chart/scripts/redis-role-probe.sh`（占位，exit 1）
- [x] 编写 `redis/apppack/chart/scripts/redis-available-probe.sh`（占位，exit 1）
- [x] 编写 `redis/apppack/chart/scripts/redis-register-to-sentinel.sh`（占位，exit 0）
- [x] 编写 `redis/apppack/chart/templates/redis-scripts-template.yaml`
- [x] 编写 `redis/apppack/chart/templates/sentinel-scripts-template.yaml`

## 本地单元测试工具链

- [x] 编写 `redis/apppack/chart/scripts-ut-spec/utils.sh`
- [x] 编写至少一个最小 shellspec 示例
- [x] 确认 shellcheck 能运行
- [x] 确认 helm lint / helm template 能运行

## 集成测试环境抽象层

- [x] 编写 `redis/test/hack/setup-cluster.sh`
- [x] 编写 `redis/test/hack/install-koda.sh`
- [x] 编写 `redis/test/hack/teardown-cluster.sh`
- [x] 编写 `redis/test/README.md`（测试环境说明）

## 入口脚本

- [x] 编写 `redis/Makefile`，包含 `test-unit`、`test-integration`、`test-e2e` 目标

## 示例占位

- [x] 编写 `redis/examples/application/namespace.yaml`（占位）
- [x] 编写 `redis/examples/application/application.yaml`（占位）
- [x] 编写 `redis/examples/application/kustomization.yaml`（占位）
- [x] 编写 `redis/examples/opstasks/switchover.yaml`（占位）
- [x] 编写 `redis/examples/opstasks/reconfigure.yaml`（占位）
- [x] 编写 `redis/examples/opstasks/horizontal-scale-out.yaml`（占位）
- [x] 编写 `redis/examples/opstasks/horizontal-scale-in.yaml`（占位）

## 阶段 0 验收

- [x] `make test-unit` 通过
- [x] 空壳 CRD 可在 kind / 已有 k8s 集群中 apply
