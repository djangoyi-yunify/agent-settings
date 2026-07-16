# Redis 7 AppPack 基础骨架设计

## 1. 设计原则

本阶段遵循 **最小可运行可测试** 原则：

- 只搭建能渲染、能 lint、能 apply、能跑测试入口的骨架。
- CRD 填最小有效字段，能被 Koda 控制器接受。
- 脚本占位能通过 shellcheck。
- 测试链能跑起来，即使 spec 文件很少。

## 2. 目录结构

```
redis/
├── apppack/
│   ├── apppack.yaml
│   └── chart/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── README.md
│       ├── .helmignore
│       ├── config/
│       │   ├── redis7-config-constraint.cue
│       │   ├── redis7-config.tpl
│       │   └── redis-config-effect-scope.yaml
│       ├── scripts/
│       │   ├── redis-start.sh
│       │   ├── redis-sentinel-start.sh
│       │   ├── redis-role-probe.sh
│       │   ├── redis-available-probe.sh
│       │   └── redis-register-to-sentinel.sh
│       ├── scripts-ut-spec/
│       │   └── utils.sh
│       └── templates/
│           ├── _helpers.tpl
│           ├── applicationdefinition.yaml
│           ├── componentmatrix.yaml
│           ├── componentdefinition-redis-server.yaml
│           ├── componentdefinition-redis-sentinel.yaml
│           ├── configfiledefinition-redis-server.yaml
│           ├── redis-config-template.yaml
│           ├── redis-scripts-template.yaml
│           └── sentinel-scripts-template.yaml
├── examples/
│   ├── application/
│   │   ├── namespace.yaml
│   │   ├── application.yaml
│   │   └── kustomization.yaml
│   └── opstasks/
│       ├── switchover.yaml
│       ├── reconfigure.yaml
│       ├── horizontal-scale-out.yaml
│       └── horizontal-scale-in.yaml
├── test/
│   ├── README.md
│   └── hack/
│       ├── setup-cluster.sh
│       ├── install-koda.sh
│       └── teardown-cluster.sh
└── Makefile
```

## 3. 命名约定

| 资源 | 名称 |
|---|---|
| AppPack | `redis-7` |
| ApplicationDefinition | `redis-7` |
| 拓扑名 | `replication-sentinel` |
| ComponentDefinition (Server) | `redis-7-server` |
| ComponentDefinition (Sentinel) | `redis-7-sentinel` |
| ConfigFileDefinition (Server) | `redis-7-server-conf` |
| ComponentMatrix | `redis-7` |
| Server 脚本 ConfigMap | `redis-7-server-scripts` |
| Sentinel 脚本 ConfigMap | `redis-7-sentinel-scripts` |

## 4. Helm chart 基础

### Chart.yaml

```yaml
apiVersion: v2
name: redis-7
description: Koda Redis 7.2 AppPack chart
type: application
version: 0.1.0
appVersion: "7.2.7"
```

### values.yaml

```yaml
nameOverride: redis-7

images:
  redis: redis:7.2.7
  redisExporter: oliver006/redis_exporter:v1.55.0
```

## 5. 空壳 CRD 设计

### ApplicationDefinition

- 声明 `server` 和 `sentinel` 两个组件。
- 声明 `replication-sentinel` 拓扑。
- 组件引用对应的 ComponentDefinition。

### ComponentDefinition (Redis Server)

- 声明角色：primary、secondary。
- 声明端口：redis（6379）。
- 声明 volumes：`data` PVC 挂载到 `/data`。
- 声明 configs：`redis-config` ConfigMap 挂载到 `/etc/conf`。
- 声明 scripts：ConfigMap 挂载到 `/scripts`。
- lifecycle actions 留空或仅声明占位。

### ComponentDefinition (Redis Sentinel)

- 声明端口：sentinel（26379）。
- 声明 volumes：`data` PVC 挂载到 `/data`。
- 声明 scripts：ConfigMap 挂载到 `/scripts`。
- lifecycle actions 留空或仅声明占位。

### ConfigFileDefinition (Redis Server)

- 声明 `redis.conf` 模板引用。
- 声明文件格式和最小 CUE schema 占位。
- managedParams 留空，后续迭代填充。

### ComponentMatrix

- 声明版本 `7.2.7` 到镜像的映射。

### AppPack

- 声明 chart 引用和元数据。

## 6. 配置模板

### redis7-config.tpl

最小 redis.conf 模板，包含：

```
# Redis 7.2 config template
protected-mode no
tcp-backlog 511
timeout 0
tcp-keepalive 300
```

### redis7-config-constraint.cue

最小 CUE 约束占位，后续迭代完善。

### redis-config-effect-scope.yaml

最小 effect scope 占位，后续迭代完善。

## 7. 脚本占位

脚本占位需满足：

- 能通过 shellcheck。
- 能在 Pod 中执行不崩溃。
- 后续迭代逐步替换为真实逻辑。

```bash
#!/bin/sh
set -eu
exec redis-server /data/redis.conf
```

```bash
#!/bin/sh
set -eu
exec redis-server /data/sentinel/redis-sentinel.conf --sentinel
```

```bash
#!/bin/sh
set -eu
exit 1
```

## 8. 测试入口

### Makefile

```makefile
.PHONY: test-unit test-integration test-e2e

test-unit:
	shellcheck --severity=error apppack/chart/scripts/*.sh
	helm lint apppack/chart
	helm template apppack/chart >/dev/null
	test -d apppack/chart/scripts-ut-spec && shellspec --load-path ./shellspec --default-path "apppack/chart/scripts-ut-spec" --shell bash || true

test-integration:
	redis/test/integration/run.sh

test-e2e:
	redis/test/e2e/run.sh
```

### shellspec

- 创建 `scripts-ut-spec/utils.sh` 作为公共工具函数。
- 至少一个最小 spec 示例。

### 集成测试脚本

- `setup-cluster.sh`：启动 kind 或复用已有 k8s。
- `install-koda.sh`：安装 local-path-provisioner 和 Koda（占位或最小实现）。
- `teardown-cluster.sh`：清理资源。

## 9. 验收标准

| 层级 | 验收项 |
|---|---|
| 单元测试 | `make test-unit` 成功运行；`shellcheck` 无 error；`helm lint` 通过；`helm template` 能渲染 |
| 集成测试 | 在 kind 或已有 k8s 集群中 `kubectl apply` 空壳 CRD 成功，不崩溃 |
| e2e | 无 |
