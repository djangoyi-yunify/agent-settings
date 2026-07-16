# Redis 7 AppPack 实施设计

## 1. 总体设计原则

### 1.1 启动方式：主容器 entrypoint

本 AppPack 采用**主容器 entrypoint** 方式启动 Redis Server 与 Sentinel。entrypoint 脚本一次性承担三类职责：

1. 加载公共函数库与 Pod 身份、Service DNS 等静态信息；
2. 生成或 surgical update 运行时配置文件；
3. 通过 `exec` 启动 `redis-server` / `redis-sentinel` 进程。

这一选择与 `redis/docs/design/redis-technical-approach.md` 中设想的“init container 生成配置、主容器只启动进程”方案不同。我们最终采用 entrypoint，基于以下判断：

- **与 Koda MySQL Demo 保持一致**：Koda 当前示例（如 MySQL Demo）均使用主容器 entrypoint 模式。采用相同模式可降低维护者理解成本，也便于复用平台侧对 entrypoint 脚本的日志、重启、调试假设。
- **KubeBlocks Redis addon 的实际实现也是 entrypoint 模式**：kubeblocks-addons 的 chart 里虽然有一个 `init-dbctl` init container，但它只负责拷贝 `dbctl` 工具，配置生成与进程启动实际由 `redis-start.sh` / `redis-sentinel-start-v2.sh` 在主容器完成。因此 entrypoint 并非 Koda 特例，而是社区 addon 的既有实践。
- **减少挂载与容器间协调**：init container 方案需要把生成的配置文件传递到主容器（通常通过 emptyDir 或同一 PVC 的特定子路径），会增加挂载设计和排障复杂度；entrypoint 方案中配置生成与进程启动在同一容器、同一文件系统视图内，更简单直接。
- **失败语义一致**：entrypoint 脚本失败会触发容器重启，与 Redis/Sentinel 启动失败的恢复语义一致；init container 失败则会阻塞 Pod 启动，需要额外区分“配置生成失败”与“进程启动失败”的两条恢复路径。

基于以上原因，本实施设计以**主容器 entrypoint** 为准。`redis-technical-approach.md` 中关于“init container 生成配置”的描述视为早期探索性思路，不再作为本期实施依据。

entrypoint 脚本的职责边界：

- **只生成配置，不启动进程**：脚本前半段生成 `/data/redis.conf` 或 `/data/sentinel/redis-sentinel.conf`。
- **不查询运行时拓扑做复杂决策**：启动时只需要知道自身身份、Service 地址、Sentinel 地址等静态信息；主从关系通过 Sentinel 查询或默认规则确定。
- **最后 `exec` 启动进程**：`exec redis-server /data/redis.conf` / `exec redis-server /data/sentinel/redis-sentinel.conf --sentinel`。

| 维度 | init container | 主容器 entrypoint |
|---|---|---|
| 配置生成时机 | Pod 初始化阶段 | 容器启动时 |
| 失败处理 | init 失败直接阻断 Pod | entrypoint 失败容器重启 |
| 工具依赖 | 可单独镜像 | 需主镜像带脚本 |
| surgical update sentinel.conf | init 做，主容器只做启动 | 同一段脚本承担生成与启动 |
| 与 Koda 示例契合 | 与 MySQL Demo 不同 | 与 MySQL Demo 一致 |

### 1.2 按生命周期垂直切片

Redis Server 与 Sentinel 是配合工作的两个组件，不能先做完一个再做另一个。实施按生命周期切片，每个切片同时覆盖两个组件：

- Bootstrap：两者都要能启动
- 服务发现：Server 通过 Sentinel 找主节点，Sentinel 监控 Server
- 扩缩容：两者同时处理成员变更
- 配置变更：两者同时支持热更新
- 切换/容灾：Server 的 switchover 触发 Sentinel failover
- 账号管理：initAccount 在 Bootstrap 处理，非 initAccount 走 accountProvision

### 1.3 配置文件与存储策略

参考 `redis/docs/design/redis-config-rewrite.md`，结合 entrypoint 启动方式与本期实现约束：

- **存储卷**：统一使用一个数据卷 `/data`，同时存放数据文件和运行时配置文件，简化挂载设计。
- **`redis.conf`**：由 Koda 模板渲染为只读 ConfigMap 并挂载到 `/etc/conf`；运行时主配置 `/data/redis.conf` 由 entrypoint 脚本生成，通过 `include /etc/conf/redis.conf` 引入只读模板；`CONFIG REWRITE` 只修改 `/data/redis.conf`。
- **`sentinel.conf`**：不经过 Koda 模板渲染，完全由 Sentinel entrypoint 脚本生成在 `/data/sentinel/redis-sentinel.conf`；Sentinel 运行时自动重写该文件。
- **ACL 文件**：独立放在 `/data/users.acl`，由运行时自主管理。initAccount（default）密码以及后续 accountProvision 创建的系统账号均通过 ACL 规则写入该文件。

挂载路径 summary：

```
Redis Server Pod:
├── /data                     # 持久化数据卷（PVC）
│   ├── redis.conf            # 可写主配置
│   ├── users.acl             # ACL 文件
│   └── dump.rdb / appendonly.aof  # 数据文件
├── /etc/conf                 # 只读配置模板 ConfigMap（仅 Server）
└── /scripts                  # 脚本 ConfigMap

Redis Sentinel Pod:
├── /data                     # 持久化数据卷（PVC）
│   └── sentinel/
│       └── redis-sentinel.conf   # 可写 Sentinel 配置
└── /scripts                  # 脚本 ConfigMap
```

这样设计的原因：

- Redis Server 配置需要开放用户热更新，因此需要 Koda 模板 + include 拆分。
- Sentinel 配置不开放用户变更，且需要保留运行时状态（monitor、known-replica、epoch 等），由脚本直接管理更简单可靠。
- 把 Server 主配置也放在数据卷 `/data/redis.conf`，主要是为了**简化挂载设计**（少一个 emptyDir），与 Sentinel 保持统一的卷使用方式。

#### 持久化主配置的骨架生成原则

`/data/redis.conf` 位于 PVC，容器重启和 Pod 重建后文件仍然存在。entrypoint 脚本每次容器启动时**清空并重新生成主配置骨架**，原因如下：

- **热更新参数来自模板**：用户通过 Koda 控制台热更新的参数（如 `maxmemory`、`timeout` 等）会被写回 ConfigFileDefinition 对应的运行时 ConfigMap（`/etc/conf/redis.conf`）。启动脚本通过 `include /etc/conf/redis.conf` 引入最新模板，自然获得最新值，无需保留 `/data/redis.conf` 中的旧回写内容。
- **动态参数必须重新计算**：`port`/`tls-port`、`replica-announce-ip`、`replica-announce-port`、`replicaof`、`masteruser`、`masterauth`、`tls-*` 等必须在启动时根据当前 Pod 身份和拓扑重新生成。
- **ACL 文件独立管理**：`/data/users.acl` 由启动脚本幂等地清理并重建系统账号行，业务账号变更通过 `accountProvision` 写入（本期 deferred）。

> **注意**：Sentinel 的 `/data/sentinel/redis-sentinel.conf` **不能**完全清空，因为其中包含 Sentinel 运行时写入的仲裁状态（`sentinel monitor`、`known-replica`、`known-sentinel`、epoch 等），必须采用 surgical update。

具体生成顺序：

```
> /data/redis.conf          # 清空，开始生成骨架
include /etc/conf/redis.conf
port / tls-port
replica-announce-ip
replica-announce-port
replicaof / # primary 不写入
masteruser replica
masterauth $REDIS_REPL_PASSWORD
tls-*（如启用）
aclfile /data/users.acl
```

> **已确认决策**：Redis Server `/data/redis.conf` 每次启动可安全清空重建。热更新参数由 Koda 模板保证，无需从旧主配置中恢复。
>
> **已确认决策**：Redis Server 与 Sentinel 均采用主容器 entrypoint 方式启动。配置生成、配置 surgical update、进程启动由同一份脚本完成，不引入单独的 init container。

### 1.4 环境变量使用原则

参考 `redis/docs/research/koda-pod-env-projection-realtime-and-restart.md`：

- **非实时 env**：用于 Pod 身份、Service DNS、静态端口等创建时已知信息。
- **实时 env**：用于 lifecycle action 上下文（如 `KODA_MEMBER_JOIN_POD_NAME`、`KODA_CONFIG_CHANGED_PARAMETERS`）。
- **不能依赖非实时 env 获取当前存活成员、leader、实际副本数**，必须通过 Redis/Sentinel 协议查询。

### 1.5 测试左移

每个迭代都有明确的验收标准：

- 单元测试：脚本 shellspec、shellcheck、helm lint/template。
- 集成测试：在 kind 或已有 k8s 集群中部署验证。
- e2e 测试：故障转移、扩缩容、端到端走查。

测试环境抽象层支持：

- 默认使用 kind。
- 通过环境变量复用已有 k8s 集群。
- GitHub Actions workflow 列为可选后续。

### 1.6 关键决策记录

| 决策 | 当前选择 | 备选 | 备注 |
|---|---|---|---|
| 启动方式 | 主容器 entrypoint | init container | 与 Koda MySQL Demo 及 KB addon 实践一致 |
| redis.conf 位置 | `/data/redis.conf`（PVC） | `/etc/redis/redis.conf`（emptyDir） | 简化挂载，但需处理持久化残留 |
| Sentinel 配置 | 脚本生成，不开放用户变更 | ConfigFileDefinition 模板 | 保留运行时状态 |
| 系统账号 | initAccount 在启动脚本初始化 | accountProvision | 本期无业务账号，简化实现 |
| 跨组件密码 | `credentialFieldRef` 跨组件引用 | serviceDependencyFieldRef | 需验证 Koda 支持情况 |
| memberJoin/Leave | Server 自发现；Sentinel leave 可选 RESET | 完全实现 KB 风格 | 见迭代 3 详细讨论 |
| switchover candidate | 支持指定 candidate（推荐） | 仅无 candidate failover | 见迭代 5 |
| Sentinel reconfigure | 本期不实现 | 实现 | Sentinel 配置不开放用户变更 |

---

## 2. 阶段 0：基础骨架

### 2.1 目标

搭好目录结构、Helm chart、空壳 CRD、测试基础设施，使项目能渲染、能 apply、能跑单元测试。

### 2.2 目录结构

chart 内部尽量与 `kubeblocks-addons/addons/redis` 保持一致，便于直接复用脚本和配置模板；外层再叠加 koda-apppacks 特有的 AppPack manifest、Application 示例、运维任务和测试基础设施。

```
redis/
├── apppack/
│   ├── apppack.yaml
│   └── chart/                          # 与 kubeblocks-addons/addons/redis/ 内部结构一致
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── README.md
│       ├── .helmignore
│       ├── config/                     # Redis Server 配置模板与 CUE 约束
│       │   ├── redis7-config-constraint.cue
│       │   ├── redis7-config.tpl
│       │   └── redis-config-effect-scope.yaml
│       │   # 注：Sentinel 配置不开放用户变更，模板直接放在 templates/ 下
│       ├── scripts/                    # 非 cluster 模式生命周期脚本（文件名后续细化）
│       │   └── ...
│       ├── scripts-ut-spec/            # 脚本单元测试（与 kubeblocks-addons 保持一致，文件名后续细化）
│       │   └── ...
│       └── templates/                  # Helm 模板，渲染 Koda CR 与 ConfigMap
│           ├── _helpers.tpl
│           ├── applicationdefinition.yaml
│           ├── componentmatrix.yaml
│           ├── componentdefinition-redis-server.yaml
│           ├── componentdefinition-redis-sentinel.yaml
│           ├── configfiledefinition-redis-server.yaml
│           ├── redis-config-template.yaml       # Server 只读配置模板
│           ├── redis-scripts-template.yaml
│           └── sentinel-scripts-template.yaml
│           # 注：Sentinel 配置文件不由模板渲染，由启动脚本生成在持久卷中
├── examples/                           # Koda 使用示例（与 koda/examples 结构一致）
│   ├── application/
│   │   ├── namespace.yaml
│   │   ├── application.yaml
│   │   └── kustomization.yaml
│   └── opstasks/
│       ├── switchover.yaml
│       ├── reconfigure.yaml
│       ├── horizontal-scale-out.yaml
│       └── horizontal-scale-in.yaml
├── test/                               # 集成/e2e 测试（不放在 chart 内）
│   ├── README.md
│   ├── hack/
│   │   ├── setup-cluster.sh
│   │   ├── install-koda.sh
│   │   └── teardown-cluster.sh
│   ├── integration/
│   │   ├── run.sh
│   │   └── fixtures/
│   └── e2e/
│       ├── run.sh
│       └── failover-test.sh
└── README.md
```

### 与 kubeblocks-addons 的对应关系

| kubeblocks-addons/redis | koda-apppacks/redis | 说明 |
|---|---|---|
| `Chart.yaml` | `apppack/chart/Chart.yaml` | chart 元数据 |
| `values.yaml` | `apppack/chart/values.yaml` | 默认参数 |
| `config/` | `apppack/chart/config/` | 配置模板与 CUE 约束 |
| `scripts/` | `apppack/chart/scripts/` | 非 cluster 生命周期脚本 |
| `scripts-ut-spec/` | `apppack/chart/scripts-ut-spec/` | 脚本单元测试 |
| `templates/` | `apppack/chart/templates/` | Helm 模板 |
| `redis-cluster-scripts/` | 暂无 | 本期不支持 Redis Cluster |
| `dataprotection/` | 暂无 | 本期不支持备份恢复 |
| `dashboards/` | 暂无 | 本期不支持 Grafana dashboard |
| — | `apppack/apppack.yaml` | Koda AppPack manifest |
| — | `examples/application/` | Application 示例 |
| — | `examples/opstasks/` | 运维任务示例 |
| — | `test/` | 集成/e2e 测试基础设施 |

### 2.3 命名约定

| 资源 | 名称 |
|---|---|
| AppPack | `redis-7` |
| ApplicationDefinition | `redis-7` |
| 拓扑名 | `replication-sentinel` |
| ComponentDefinition (Server) | `redis-7-server` |
| ComponentDefinition (Sentinel) | `redis-7-sentinel` |
| ConfigFileDefinition (Server) | `redis-7-server-conf` |
| Sentinel 内部配置模板 | `redis-7-sentinel-template` | 不对应 ConfigFileDefinition，不开放用户变更 |
| ComponentMatrix | `redis-7` |
| Server ConfigMap 模板 | `redis-7-server-template` |
| Sentinel ConfigMap 模板 | `redis-7-sentinel-template` |
| Server 脚本 ConfigMap | `redis-7-server-scripts` |
| Sentinel 脚本 ConfigMap | `redis-7-sentinel-scripts` |

### 2.4 Helm chart 基础

#### Chart.yaml

```yaml
apiVersion: v2
name: redis-7
description: Koda Redis 7.2 AppPack chart
type: application
version: 0.1.0
appVersion: "7.2.7"
```

#### values.yaml

```yaml
nameOverride: redis-7

images:
  redis: redis:7.2.7
  redisExporter: oliver006/redis_exporter:v1.55.0
```

### 2.5 空壳 CRD

阶段 0 的 CRD 只定义结构和引用关系，不填具体生命周期动作细节。例如：

- `ApplicationDefinition` 声明 `server` 和 `sentinel` 两个组件。
- `ComponentDefinition` 声明角色、端口、service 导出、系统账号、volume 挂载等骨架。
  - Redis Server：挂载 `/data`（data PVC）、`/etc/conf`（配置模板 ConfigMap）、`/scripts`（脚本 ConfigMap）。
  - Redis Sentinel：挂载 `/data`（data PVC）、`/scripts`（脚本 ConfigMap）。
- `ConfigFileDefinition`（仅 Server）声明 `redis.conf` 模板引用、文件格式、CUE schema 占位。Sentinel 配置不经过模板渲染，由启动脚本生成在持久化卷中。
- `ComponentMatrix` 声明版本与镜像映射。

### 2.6 脚本 ConfigMap 占位

阶段 0 中脚本先放最小占位，保证 chart 能渲染。例如：

```yaml
# redis-scripts-template.yaml
data:
  <server-start-script>.sh: |
    #!/bin/sh
    set -eu
    exec redis-server /etc/koda-redis/redis.conf
  <role-probe-script>.sh: |
    #!/bin/sh
    exit 1
  # ... 其他脚本占位（文件名后续细化）
```

### 2.7 测试基础设施

#### 本地单元测试

与 `kubeblocks-addons/addons/redis` 保持一致，采用以下工具：

| 工具 | 用途 | 说明 |
|---|---|---|
| `shellspec` | 脚本单元测试框架 | 参考 `kubeblocks-addons/scripts-ut-spec/` |
| `shellcheck` | 脚本静态检查 | CI 中强制运行，本地开发建议启用 |
| `helm lint` / `helm template` | chart 语法与渲染检查 | helm 自带 |
| `kubeconform` | manifest schema 校验 | 可选，阶段 0 不强制 |

入口：

```bash
make test-unit
```

建议的 `make test-unit` 目标：

```bash
shellspec --load-path ./shellspec --default-path "**/scripts-ut-spec" --shell bash
shellcheck --severity=error **/*.sh
helm lint apppack/chart
helm template apppack/chart >/dev/null
```

#### 集成测试环境抽象层

脚本：

- `test/hack/setup-cluster.sh`：启动 kind 或接入已有 k8s。
- `test/hack/install-koda.sh`：安装 local-path-provisioner 和 Koda。
- `test/hack/teardown-cluster.sh`：清理资源。

环境变量：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `KODA_TEST_CLUSTER_PROVIDER` | `auto` | `auto` / `kind` / `existing` |
| `KODA_TEST_KUBECONFIG` | `~/.kube/config` | kubeconfig 路径 |
| `KODA_TEST_CONTEXT` | 当前上下文 | 已有集群上下文 |
| `KODA_TEST_SKIP_KIND_CREATE` | `false` | 为 `true` 时不创建 kind |
| `KODA_TEST_SKIP_TEARDOWN` | `false` | 为 `true` 时不清理 |

入口：

```bash
make test-integration
```

#### e2e 测试

- 在 kind 或已有集群上验证故障转移、数据持久化等。
- 入口：`make test-e2e`。

### 2.8 阶段 0 验收标准

| 层级 | 验收项 |
|---|---|
| 单元测试 | `shellspec` 能运行；`shellcheck` 无 error；`helm lint` 通过；`helm template` 能渲染 |
| 集成测试 | 空壳 CRD 在 Koda 集群中 `kubectl apply` 成功 |
| e2e | 无 |

---

## 3. 迭代 1：Bootstrap / 创建

### 3.1 目标

完成 Redis Server 与 Redis Sentinel 的启动能力，使两者首次创建后能自动形成 **Replication + Sentinel** 拓扑：

- Redis Server 启动时确定自身是 primary 还是 replica，生成正确的 `redis.conf` 与 ACL。
- Redis Sentinel 启动后能被 primary 注册，开始监控 Redis Server 组件。
- 不依赖 `accountProvision`；所有系统账号在启动脚本中初始化。

### 3.2 账号与 ACL 设计

本期只使用 initAccount，不引入业务账号，因此**不触发 `accountProvision`**。

#### 系统账号清单

| 用户名 | 所在组件 | 用途 | 密码来源 |
|---|---|---|---|
| `default` | Redis Server | 业务客户端访问 | Redis Server 自己的 initAccount |
| `replica` | Redis Server | 主从复制（`masteruser` / `masterauth`） | 复用 Redis Server `default` 密码 |
| `redis-sentinel` | Redis Server | 供 Sentinel 组件连接 Server | Redis Sentinel 的 initAccount `default` 密码 |
| `default` | Redis Sentinel | Sentinel 进程自身认证 | Redis Sentinel 自己的 initAccount |

#### 跨组件密码引用

```yaml
# Redis Server ComponentDefinition
env:
- name: REDIS_DEFAULT_PASSWORD
  valueFrom:
    credentialFieldRef:
      name: default
      password: Required
- name: REDIS_REPL_PASSWORD
  valueFrom:
    credentialFieldRef:
      name: default
      password: Required
- name: REDIS_SENTINEL_PASSWORD
  valueFrom:
    credentialFieldRef:
      compDefSelector: redis-7-sentinel
      name: default
      password: Required

# Redis Sentinel ComponentDefinition
env:
- name: SENTINEL_USER
  value: "redis-sentinel"
- name: SENTINEL_PASSWORD
  valueFrom:
    credentialFieldRef:
      name: default
      password: Required
```

#### ACL 初始化

启动脚本在每次容器启动时重新生成 `/data/users.acl` 中的系统账号行，保证幂等：

```bash
rebuild_redis_acl_file() {
  if [ -f /data/users.acl ]; then
    sed -i "/user default on/d" /data/users.acl
    sed -i "/user replica on/d" /data/users.acl
    sed -i "/user redis-sentinel on/d" /data/users.acl
  else
    touch /data/users.acl
  fi
}

build_redis_default_accounts() {
  rebuild_redis_acl_file

  # default
  default_hash=$(echo -n "$REDIS_DEFAULT_PASSWORD" | sha256sum | cut -d' ' -f1)
  echo "user default on #$default_hash ~* &* +@all" >> /data/users.acl

  # replica
  repl_hash=$(echo -n "$REDIS_REPL_PASSWORD" | sha256sum | cut -d' ' -f1)
  echo "masteruser replica" >> /data/redis.conf
  echo "masterauth $REDIS_REPL_PASSWORD" >> /data/redis.conf
  echo "user replica on +psync +replconf +ping #$repl_hash" >> /data/users.acl

  # redis-sentinel
  sentinel_hash=$(echo -n "$REDIS_SENTINEL_PASSWORD" | sha256sum | cut -d' ' -f1)
  echo "user redis-sentinel on allchannels +multi +slaveof +ping +exec +subscribe +config|rewrite +role +publish +info +client|setname +client|kill +script|kill #$sentinel_hash" >> /data/users.acl

  echo "aclfile /data/users.acl" >> /data/redis.conf
}
```

### 3.3 Redis Server 启动流程

```
启动脚本 redis-start.sh
│
├─ 1. 加载公共函数库
├─ 2. 解析当前 Pod 的 announce 地址（FQDN / NodePort / HostNetwork）
├─ 3. 生成 /data/redis.conf
│   ├─ include /etc/conf/redis.conf（只读模板）
│   ├─ replica-announce-ip / replica-announce-port
│   ├─ port / tls-port
│   ├─ TLS 配置（如启用）
│   ├─ 确定主从关系（见 3.3.1）
│   ├─ 重建 /data/users.acl 并写入系统账号
│   └─ aclfile /data/users.acl
└─ 4. exec redis-server /data/redis.conf
```

#### 3.3.1 主从关系确定

1. 如果环境变量中存在 `SENTINEL_COMPONENT_NAME` 与 `SENTINEL_POD_FQDN_LIST`，则遍历 Sentinel 节点，执行：
   ```bash
   sentinel get-master-addr-by-name <REDIS_COMPONENT_NAME>
   ```
2. 如果 Sentinel 返回有效主节点地址，当前 Pod 比较自身身份：
   - 匹配 → 作为 primary 启动
   - 不匹配 → 配置 `replicaof <primary> <port>`
3. 如果 Sentinel 尚未监控（首次创建），回退到字典序最小 Pod 作为默认 primary。

### 3.4 Redis Sentinel 启动流程

```
启动脚本 redis-sentinel-start.sh
│
├─ 1. 加载公共函数库
├─ 2. 解析当前 Pod 的 announce 地址
├─ 3. surgical update /data/sentinel/redis-sentinel.conf
│   ├─ 保留已有的 monitor / known-replica / known-sentinel / epoch
│   ├─ 删除并重新写入动态参数：announce-ip、announce-port、port、tls-*、aclfile
│   └─ 写入 sentinel sentinel-user / sentinel sentinel-pass
└─ 4. exec redis-server /data/sentinel/redis-sentinel.conf --sentinel
```

Sentinel 刚启动时还没有 `SENTINEL MONITOR`，监控关系由 Redis primary 的 `postProvision` 注册。

### 3.5 ComponentDefinition 关键配置

#### Redis Server

- **podManagementPolicy**: `OrderedReady`
- **roles**: `primary` / `secondary`
- **volumes**: `data` PVC 挂载到 `/data`
- **configs**: `redis-config` ConfigMap 挂载到 `/etc/conf`
- **lifecycle.actions**:
  - `roleProbe`: 检测当前是 primary 还是 secondary
  - `availableProbe`: 健康检查
  - `postProvision`: 在 primary Pod 上执行，向所有 Sentinel 注册 `SENTINEL monitor`
  - `accountProvision`: 本期不定义（无业务账号）

#### Redis Sentinel

- **podManagementPolicy**: `Parallel`
- **volumes**: `data` PVC 挂载到 `/data`
- **lifecycle.actions**:
  - `availableProbe`: 健康检查
  - `accountProvision`: 本期不定义

### 3.6 与 kubeblocks-addons 的关键差异

| 项目 | kubeblocks-addons | 本 AppPack |
|---|---|---|
| 主从复制用户名 | `kbreplicator` | `replica` |
| Sentinel 访问用户名 | `kbreplicator-sentinel` | `redis-sentinel` |
| redis.conf 主配置位置 | `/etc/redis/redis.conf`（emptyDir） | `/data/redis.conf`（PVC） |
| accountProvision | 用于业务账号 | 本期不定义 |
| memberJoin / sync-acl | 存在 | 本期不需要 |

### 3.7 验收标准

| 层级 | 验收项 |
|---|---|
| 单元测试 | `redis-start.sh` 与 `redis-sentinel-start.sh` 的 shellspec 测试通过；`shellcheck` 无 error |
| 集成测试 | 在 kind 中创建 Application，Redis Server 与 Sentinel 均正常启动；`redis-cli INFO replication` 显示主从关系正确 |
| e2e | 通过 primary service 写入数据，在 replica 上能读取；Sentinel 返回的主节点地址与 primary service 一致 |

---

## 4. 迭代 2：角色感知与服务发现

### 4.1 目标

- Koda 能正确识别每个 Redis Server Pod 的当前角色（primary/secondary）。
- Koda 能判断 Redis Server 与 Redis Sentinel 是否健康可用。
- Redis primary 能向 Sentinel 注册监控关系，形成完整的 Replication + Sentinel 拓扑。
- 暴露客户端需要的服务：`writer`（指向 primary）和 `redis-sentinel`（指向所有 Sentinel）。

### 4.2 总体数据流

```
Redis Server Pod-0 (primary)
  ├─ roleProbe ──► stdout: "primary" ──► Koda 打标签 role=primary
  ├─ availableProbe ──► stdout: "alive"
  └─ postProvision(Role=primary)
      └─► 向所有 Sentinel 执行 SENTINEL monitor ...

Redis Server Pod-1 (secondary)
  ├─ roleProbe ──► stdout: "secondary"
  └─ availableProbe ──► stdout: "alive"

Koda Services
  ├─ redis-7-server-writer  ──► roleSelector: primary
  └─ redis-7-sentinel       ──► 所有 Sentinel Pod
```

### 4.3 roleProbe 设计

连接本地 Redis，读取 `INFO replication` 的 `role` 字段，输出 `primary` 或 `secondary`。

```bash
#!/bin/sh
set -eu

service_port=${SERVICE_PORT:-6379}
auth_args=""
if [ -n "$REDIS_DEFAULT_PASSWORD" ]; then
  auth_args="-a $REDIS_DEFAULT_PASSWORD"
fi

role=$(redis-cli $auth_args -h localhost -p $service_port INFO replication \
       | awk -F: '/^role:/ {gsub(/\r/, "", $2); print $2}')

case "$role" in
  master)  echo "primary"; exit 0 ;;
  slave)   echo "secondary"; exit 0 ;;
  *)       exit 1 ;;
esac
```

要点：
- 只报告角色，不检查复制链路健康。
- Redis 未就绪时退出码非 0，Koda 会重试。
- failover 后角色改变，roleProbe 会在下一个周期更新标签。

### 4.4 availableProbe 设计

#### Redis Server

```bash
#!/bin/sh
set -eu

service_port=${SERVICE_PORT:-6379}
auth_args=""
if [ -n "$REDIS_DEFAULT_PASSWORD" ]; then
  auth_args="-a $REDIS_DEFAULT_PASSWORD"
fi

redis-cli $auth_args -h localhost -p $service_port PING | grep -q PONG
echo "alive"
```

#### Redis Sentinel

```bash
#!/bin/sh
set -eu

sentinel_port=${SENTINEL_SERVICE_PORT:-26379}
auth_args=""
if [ -n "$SENTINEL_PASSWORD" ]; then
  auth_args="-a $SENTINEL_PASSWORD"
fi

redis-cli $auth_args -h localhost -p $sentinel_port PING | grep -q PONG
echo "alive"
```

Sentinel 没有 primary/secondary 角色，因此不需要 roleProbe。

### 4.5 postProvision：向 Sentinel 注册监控

`postProvision` 是 **Component 级**生命周期 hook，不是 Pod 级。一个 Component 实例只会成功执行一次。

```yaml
lifecycle:
  actions:
    postProvision:
      exec:
        container: redis
        command:
        - /bin/sh
        - /scripts/redis-register-to-sentinel.sh
      targetPodSelector: Role
      matchingKey: primary
      preCondition: ComponentReady
```

执行逻辑：

```bash
#!/bin/sh
set -eu

master_name="${REDIS_COMPONENT_NAME}"
primary_host="$(get_current_pod_fqdn_or_announce_host)"
primary_port="${SERVICE_PORT:-6379}"

for sentinel in $(echo "$SENTINEL_POD_FQDN_LIST" | tr ',' '\n'); do
  # 等待 Sentinel 就绪
  until redis-cli -h "$sentinel" -p 26379 -a "$SENTINEL_PASSWORD" PING | grep -q PONG; do
    sleep 2
  done

  # 检查是否已监控
  existing=$(redis-cli -h "$sentinel" -p 26379 -a "$SENTINEL_PASSWORD" \
             SENTINEL get-master-addr-by-name "$master_name" 2>/dev/null || true)

  if [ -z "$existing" ]; then
    redis-cli -h "$sentinel" -p 26379 -a "$SENTINEL_PASSWORD" \
      SENTINEL monitor "$master_name" "$primary_host" "$primary_port" 2

    redis-cli -h "$sentinel" -p 26379 -a "$SENTINEL_PASSWORD" \
      SENTINEL set "$master_name" down-after-milliseconds 20000
    redis-cli -h "$sentinel" -p 26379 -a "$SENTINEL_PASSWORD" \
      SENTINEL set "$master_name" failover-timeout 60000
    redis-cli -h "$sentinel" -p 26379 -a "$SENTINEL_PASSWORD" \
      SENTINEL set "$master_name" parallel-syncs 1
    redis-cli -h "$sentinel" -p 26379 -a "$SENTINEL_PASSWORD" \
      SENTINEL set "$master_name" auth-user redis-sentinel
    redis-cli -h "$sentinel" -p 26379 -a "$SENTINEL_PASSWORD" \
      SENTINEL set "$master_name" auth-pass "$SENTINEL_PASSWORD"
  fi
done
```

参数说明：

| 参数 | 值 | 说明 |
|---|---|---|
| quorum | 2 | 3 个 Sentinel 中至少 2 个同意才判定故障 |
| down-after-milliseconds | 20000 | 20 秒无响应判定主观下线 |
| failover-timeout | 60000 | failover 超时 60 秒 |
| parallel-syncs | 1 | 一次同步 1 个 replica |

### 4.6 跨组件环境变量注入

#### Redis Server 需要的环境变量

```yaml
env:
# 本组件 identity
- name: REDIS_COMPONENT_NAME
  valueFrom:
    componentFieldRef:
      componentName: Required
- name: REDIS_POD_FQDN_LIST
  valueFrom:
    componentFieldRef:
      podFQDNs: Required

# Sentinel 信息（跨组件引用）
- name: SENTINEL_COMPONENT_NAME
  valueFrom:
    componentFieldRef:
      compDefSelector: redis-7-sentinel
      componentName: Required
- name: SENTINEL_POD_FQDN_LIST
  valueFrom:
    componentFieldRef:
      compDefSelector: redis-7-sentinel
      podFQDNs: Required
- name: REDIS_SENTINEL_PASSWORD
  valueFrom:
    credentialFieldRef:
      compDefSelector: redis-7-sentinel
      name: default
      password: Required
```

#### Redis Sentinel 需要的环境变量

Sentinel 启动时不需要知道 Redis 在哪，它等待被注册。只需自己的认证信息：

```yaml
env:
- name: SENTINEL_USER
  value: "redis-sentinel"
- name: SENTINEL_PASSWORD
  valueFrom:
    credentialFieldRef:
      name: default
      password: Required
```

### 4.7 Service 导出

```yaml
contracts:
  services:
    exports:
    # Redis Server
    - name: redis
      serviceName: redis
      serviceSpec:
        ports:
        - name: redis
          port: 6379
          targetPort: redis
    - name: writer
      serviceName: writer
      roleSelector: primary
      serviceSpec:
        ports:
        - name: redis
          port: 6379
          targetPort: redis
    # Redis Sentinel
    - name: redis-sentinel
      serviceName: redis-sentinel
      serviceSpec:
        ports:
        - name: sentinel
          port: 26379
          targetPort: sentinel
```

- `redis`：指向所有 Redis Server Pod。
- `writer`：只指向当前 primary（通过 roleSelector），用于写操作。
- `redis-sentinel`：指向所有 Sentinel Pod，客户端用来发现当前 primary。

### 4.8 关键时序与风险

#### 启动时序

```
t0: Redis Server pod-0 启动
    └─ 启动脚本：Sentinel 未就绪，回退到 pod-0 为 primary
    └─ 启动 redis-server

t1: roleProbe 在 pod-0 返回 primary
    └─ Koda 给 pod-0 打标签 role=primary

t2: postProvision(Role=primary) 在 pod-0 执行
    └─ 重试连接 Sentinel
    └─ 执行 SENTINEL monitor ...

t3: Redis Server pod-1 启动
    └─ 启动脚本询问 Sentinel：get-master-addr-by-name
    └─ Sentinel 返回 pod-0 地址
    └─ pod-1 配置 replicaof pod-0

t4: roleProbe 在 pod-1 返回 secondary
```

#### 主要风险

1. **postProvision 执行时 Sentinel 未就绪**
   - 脚本必须带重试逻辑。
   - Koda 的 postProvision 失败会让 Component 保持 `Ready=False` 并重新 reconcile。

2. **postProvision 是 Component 级，只执行一次**
   - 扩容新 Pod 不会触发 postProvision。
   - primary Pod 被替换后，新 primary 也不会触发 postProvision。
   - 这没有问题：后续 replica 通过 Sentinel 自动发现 primary；primary 替换通过 Sentinel failover 自动处理。
   - 只有当整个 Redis Server Component 删除重建时，postProvision 才会再次执行。

3. **failover 后 roleProbe 延迟**
   - Sentinel failover 后，新 primary 的 Redis 进程角色改变。
   - roleProbe 下一个周期检测到变化，Koda 更新 pod 标签，writer service 切换。
   - 延迟 ≈ roleProbe period + Koda reconcile 延迟。

4. **跨组件 env 解析失败**
   - 如果 Sentinel 组件尚未创建，Redis Server 的 `SENTINEL_POD_FQDN_LIST` 可能为空。
   - Koda 会让 Component 保持 Reconciling 直到依赖可用。
   - 启动脚本也需要能处理空列表（回退到默认 primary）。

### 4.9 与 kubeblocks-addons 的关键差异

| 项目 | kubeblocks-addons | 本 AppPack |
|---|---|---|
| roleProbe | 使用 dbctl | 使用 `INFO replication` 脚本 |
| availableProbe | `redis-ping.sh` | 类似 `redis-cli PING` |
| postProvision | 在 Redis primary 上注册 Sentinel | ✅ 一致 |
| master name | `REDIS_COMPONENT_NAME` | ✅ 一致 |
| Sentinel 参数 | 硬编码 20s/60s/1 | ✅ 一致 |

### 4.10 验收标准

| 层级 | 验收项 |
|---|---|
| 单元测试 | roleProbe / availableProbe 的 shellspec 测试通过 |
| 集成测试 | 在 kind 中部署 3+3 拓扑；`redis-cli INFO replication` 显示正确主从关系；`SENTINEL masters` 显示已监控 master |
| e2e | 通过 `writer` service 写入数据；通过 `redis-sentinel` service 查询当前 primary；删除 primary Pod 后 writer service 切换到新 primary |

---

## 5. 迭代 3：扩缩容

### 5.1 目标

- 支持 Redis Server 水平扩容/缩容，新副本自动通过 Sentinel 发现 primary 并加入复制。
- Redis Sentinel 副本数固定为 3，本期不支持扩缩容。
- 明确 Redis Server `memberJoin` / `memberLeave` 的实现范围，避免过度设计。

### 5.2 Redis Server 扩缩容

#### scale-out

新 Server Pod 启动流程与创建时相同：

1. 启动脚本查询 Sentinel：`sentinel get-master-addr-by-name <REDIS_COMPONENT_NAME>`。
2. Sentinel 返回当前 primary 地址。
3. 当前 Pod 配置 `replicaof <primary> <port>`，启动为 replica。
4. Redis 原生复制链路建立。
5. Sentinel 自动通过监控发现新 replica，加入 `known-replica`。

复制关系的建立完全由启动脚本 + Sentinel 完成，不需要 `memberJoin` 参与拓扑发现。

但新 replica 的 ACL 文件只包含启动脚本生成的**系统账号**（`default`、`replica`、`redis-sentinel`）。如果集群中已有节点存在通过 `accountProvision` 或手动 `ACL SETUSER` 创建的**非 init 账号**，这些账号不会自动出现在新 replica 上（Redis 不会通过复制传播 ACL 变更）。因此 Redis Server 需要 **`memberJoin` action** 来同步非 init 账号。

`memberJoin` 执行 `sync-acl.sh`：

```bash
#!/bin/sh
set -eu

service_port=${SERVICE_PORT:-6379}
auth_args=""
[ -n "$REDIS_DEFAULT_PASSWORD" ] && auth_args="-a $REDIS_DEFAULT_PASSWORD"

# 从其他 Server Pod 获取 ACL LIST，跳过自己
for peer in $(echo "$REDIS_POD_FQDN_LIST" | tr ',' '\n'); do
  [ "$peer" = "$KODA_MEMBER_JOIN_POD_FQDN" ] && continue
  acl_list=$(redis-cli $auth_args -h "$peer" -p "$service_port" ACL LIST 2>/dev/null) && break
done

if [ -z "$acl_list" ]; then
  echo "no ACL rules found in peers, skip synchronization"
  exit 0
fi

# 只同步非 init 账号（跳过 default / replica / redis-sentinel）
echo "$acl_list" | while read -r rule; do
  [ -z "$rule" ] && continue
  username=$(echo "$rule" | awk '{print $2}')
  case "$username" in
    default|replica|redis-sentinel) continue ;;
  esac
  rule_part=${rule#user $username }
  redis-cli $auth_args -h "$KODA_MEMBER_JOIN_POD_FQDN" -p "$service_port" \
    ACL SETUSER "$username" $rule_part || exit 1
done

redis-cli $auth_args -h "$KODA_MEMBER_JOIN_POD_FQDN" -p "$service_port" ACL SAVE
```

> **关键设计点**：只同步非 init 账号。系统账号由启动脚本根据当前 Pod 的环境变量重新生成，确保密码变更后新 replica 使用最新密码，不会被旧 primary 的 ACL 覆盖。

#### memberJoin 的执行 Pod 与时机

根据 Koda horizontal scaling 控制器行为：

- **执行 Pod**：`memberJoin` 只在新加入的 Pod 上执行。Koda 通过 `horizontalScalingMemberPodName(target.Instance)` 定位到加入实例的 Pod，并向该 Pod 的 koda-agent 发起 action。注入的实时环境变量为 `KODA_MEMBER_JOIN_POD_NAME` 与 `KODA_MEMBER_JOIN_POD_FQDN`，二者均指向**新 Pod 自身**。
- **执行时机**：scale-out opstask 先 patch Application 增加 replicas，StatefulGroup 创建新 Pod；Koda 等待新 Pod 的 StatefulInstance 状态变为 `Ready && Available`（即 `availableProbe` 已持续成功）后，才触发 `memberJoin`。因此 `sync-acl.sh` 执行时，Redis 进程已经启动并响应 PING。

```text
scale-out 时序：
  patch Application → 增加 replicas
        │
        ▼
  StatefulGroup 创建新 Pod
        │
        ▼
  等待 availableProbe 成功（Ready && Available）
        │
        ▼
  memberJoin 在新 Pod 执行 sync-acl.sh
        │
        ▼
  scale-out 完成
```

> **注意**：由于 memberJoin 在 Pod 已 Available 后才执行，新 replica 在 `sync-acl.sh` 完成前可能已接收客户端连接，但此时缺少非 init 账号。这是一个短暂窗口，KubeBlocks 也有同样行为。本期接受该窗口；如需严格避免，未来可考虑在 readinessProbe 中增加 ACL 同步完成标记。

#### scale-in

缩容时 Pod 被删除：

- 如果删除的是 replica：该 replica 会在 Sentinel 的 `known-replica` 列表中残留为 `disconnected` 状态。Redis 官方文档明确说明 *Sentinels never forget about replicas of a given master, even when they are unreachable for a long time*，唯一清理方式是执行 `SENTINEL RESET <master-name>`。然而 KubeBlocks Redis addon 的实践经验表明，残留的 disconnected replica 不影响功能，因此本期**不实现 Redis Server 的 `memberLeave` action**，接受视图中可能存在的残留项。
- 如果删除的是 primary：Sentinel 判定主观下线后触发自动 failover，提升新 primary。

> **已确认决策**：Redis Server 不实现 `memberLeave`。原因：
> 1. Redis Sentinel 不会自动遗忘 replica，需要显式 `SENTINEL RESET` 才能清理；
> 2. KubeBlocks Redis addon 同样未实现 Server memberLeave，实践证明功能正常；
> 3. 为保持最小实现，本期接受 scale-in 后 Sentinel 视图中可能残留的 disconnected replica，后续版本可视需求补充清理逻辑。

> **已确认决策**：Redis Server 需要 `memberJoin`，执行 `sync-acl.sh`。原因：
> 1. 用户可能通过 `accountProvision` 或手动 `ACL SETUSER` 创建非 init 账号；
> 2. Redis 不会通过主从复制传播 ACL 变更，因此新 replica 启动时缺少这些非 init 账号；
> 3. `sync-acl.sh` 从任意已有 peer 节点拉取 ACL LIST，仅同步非 init 账号到新 replica，然后 `ACL SAVE` 持久化。

### 5.3 Redis Sentinel 扩缩容（本期不支持）

**本期 Redis Sentinel 副本数固定为 3，不支持扩缩容。**

Redis Sentinel 是 Redis 官方推荐的 HA 方案，标准部署就是 3 个 Sentinel 实例。3 个 Sentinel 配合 quorum=2，既能容忍 1 个 Sentinel 故障，又不会因为节点过多增加协商开销。

#### 为什么不支持扩缩容？

Sentinel 集群节点数变化时，必须同步调整 `sentinel monitor` 中的 **quorum**。quorum 决定需要多少个 Sentinel 达成一致才能将 master 标记为 `odown`，它直接影响 failover 能否触发。

| 集群节点数 | 推荐 quorum |
|---|---|
| 3 | 2 |
| 5 | 3 |
| 2 | 1（不推荐） |

关键问题是：**quorum 是 Sentinel 的本地配置，不会通过 gossip 自动传播**。因此扩缩容时不能依赖 Sentinel 自发现，必须通过生命周期动作显式在所有 Sentinel 上设置新 quorum，并执行 `SENTINEL RESET` 让已知节点重新收敛。

这意味着支持 Sentinel 扩缩容需要：
- `memberJoin`：扩容时，新 Sentinel 计算新 quorum，对所有已知 Sentinel 执行 `SENTINEL SET <master> quorum <n>` + `SENTINEL RESET`。
- `memberLeave`：缩容时，被删除 Sentinel 对剩余 Sentinel 执行同样的操作。

这些动作涉及跨 Sentinel 的 quorum 协商和 majority 计算，实现和测试复杂度较高。而 3 Sentinel 已经满足绝大多数 Redis HA 场景，因此**本期将 Sentinel 副本固定为 3**（`minReplicas: 3, maxReplicas: 3`），quorum 固定为 2，不实现 `memberJoin` / `memberLeave`。

> **已确认决策**：Redis Sentinel 本期不支持扩缩容。副本数固定为 3，quorum 固定为 2。Sentinel 动态扩缩容（含 quorum 管理）作为后续版本能力。

### 5.4 memberJoin / memberLeave 决策汇总

| 组件 | memberJoin | memberLeave | 理由 |
|---|---|---|---|
| Redis Server | **实现 `sync-acl.sh`** | 不实现 | 拓扑自发现，但需同步非 init 账号；scale-in 残留 replica 可接受 |
| Redis Sentinel | 不实现 | 不实现 | 副本固定为 3，不支持扩缩容；gossip 自发现无需 memberJoin |

> **关于 Sentinel 扩缩容**：若未来支持，核心操作是调整 `quorum` 并执行 `SENTINEL RESET` 重新发现拓扑，而非单纯清理 `known-sentinel`。

### 5.5 运维任务

针对 Redis Server 水平扩缩容的运维任务示例：

- `redis/examples/opstasks/horizontal-scale-out.yaml`
- `redis/examples/opstasks/horizontal-scale-in.yaml`

> Redis Sentinel 本期副本数固定为 3，不提供扩缩容运维任务。

### 5.6 验收标准

| 层级 | 验收项 |
|---|---|
| 单元测试 | `sync-acl.sh` 的 shellspec 测试通过 |
| 集成测试 | kind 中 3+3 拓扑，Server 从 3 scale out 到 5，新 replica 自动加入复制且非 init 账号通过 `sync-acl.sh` 同步；Server scale in 到 3 后功能正常；Sentinel 副本数保持 3 |
| e2e | scale out 前后写入/读取数据一致；业务账号在 scale out 后仍可在新 replica 上认证 |

---

## 6. 迭代 4：配置变更

### 6.1 目标

- 支持 Redis Server 部分参数热更新（`CONFIG SET` + `CONFIG REWRITE`）。
- 明确热更新参数与需重启参数的边界。
- Sentinel 配置本期不开放用户变更。

### 6.2 参数分类

参考 Redis 7.2 `CONFIG SET` 能力与 KubeBlocks 实践经验，参数分为三类：

| 分类 | 示例 | 处理方式 |
|---|---|---|
| 热更新参数 | `maxmemory`、`timeout`、`maxclients`、`tcp-keepalive`、`slowlog-log-slower-than` | `CONFIG SET` + `CONFIG REWRITE` |
| 需重启参数 | `port`、`bind`、`appendonly`、`save`、`dbfilename`、`dir` | 模板更新后触发 Pod 重建 |
| 不支持参数 | `slaveof` / `replicaof`、`masterauth`、`aclfile` | 由启动脚本管理，用户不可变更 |

热更新参数清单（v1）：

```yaml
managedParams:
- maxmemory
- maxmemory-policy
- timeout
- maxclients
- tcp-keepalive
- slowlog-log-slower-than
- slowlog-max-len
- lazyfree-lazy-eviction
- lazyfree-lazy-expire
- lazyfree-lazy-server-del
```

> 初始清单保持精简，后续根据用户反馈扩展。

### 6.3 Server reconfigure 流程

`reconfigure` 动作由 Koda 在 ConfigFileDefinition 关联的 Runtime ConfigMap 更新后触发。

Koda 注入实时环境变量：

- `KODA_CONFIG_CHANGED_PARAMETERS`：变更参数列表，格式如 `maxmemory=1gb,timeout=300`
- `KODA_CONFIG_PARAM_<NAME>`：单个参数值

脚本逻辑：

```bash
#!/bin/sh
set -eu

service_port=${SERVICE_PORT:-6379}
auth_args=""
[ -n "$REDIS_DEFAULT_PASSWORD" ] && auth_args="-a $REDIS_DEFAULT_PASSWORD"

# KODA_CONFIG_CHANGED_PARAMETERS 格式: "key1=value1,key2=value2"
IFS=',' read -ra params <<< "$KODA_CONFIG_CHANGED_PARAMETERS"

for param in "${params[@]}"; do
  key="${param%%=*}"
  value="${param#*=}"

  # 跳过非热更新参数
  case "$key" in
    maxmemory|maxmemory-policy|timeout|maxclients|tcp-keepalive|slowlog-log-slower-than|slowlog-max-len|lazyfree-lazy-eviction|lazyfree-lazy-expire|lazyfree-lazy-server-del)
      ;;
    *)
      echo "skip non-hot parameter: $key"
      continue
      ;;
  esac

  redis-cli $auth_args -h localhost -p "$service_port" CONFIG SET "$key" "$value"
  redis-cli $auth_args -h localhost -p "$service_port" CONFIG REWRITE
done
```

设计要点：

- 只在 primary 上执行 `CONFIG SET` 和 `CONFIG REWRITE`（通过 `targetPodSelector: Role` + `matchingKey: primary`）。
- replica 通过主从复制自然获得参数变更？**不成立**。Redis `CONFIG SET` 不会自动同步到 replica。因此需要在所有 Server Pod 上执行 reconfigure。
- 方案：
  - `targetPodSelector: Any`，在每个 Server Pod 上独立执行 `CONFIG SET`。
  - 或者 `targetPodSelector: Role` + `matchingKey: primary`，在 primary 执行后，replica 通过读取新的只读模板？不现实。

**采用 `targetPodSelector: Any`**，在每个 Pod 上独立执行热更新。Koda 的 fan-out 机制会逐个调用。

### 6.4 Sentinel reconfigure

本期 Sentinel 配置不开放用户变更，因此 Sentinel ComponentDefinition 中不声明 `reconfigure` 动作。

如需后续扩展，可热更新的 Sentinel 参数包括：`down-after-milliseconds`、`failover-timeout`、`parallel-syncs` 等。

### 6.5 运维任务

- `redis/examples/opstasks/reconfigure.yaml`

示例：

```yaml
apiVersion: operations.koda.io/v1alpha1
kind: OpsTask
metadata:
  name: redis-reconfigure
spec:
  type: reconfigure
  target:
    applicationRef: redis-demo
    component: server
  parameters:
    maxmemory: 512mb
    timeout: 300
```

### 6.6 验收标准

| 层级 | 验收项 |
|---|---|
| 单元测试 | reconfigure 脚本 shellspec 测试通过 |
| 集成测试 | kind 中修改 `maxmemory`，所有 Server Pod 的 `CONFIG GET maxmemory` 返回新值；Pod 未重建 |
| e2e | 热更新后业务读写正常，Pod 重建后配置仍持久化 |

---

## 7. 迭代 5：主从切换与故障转移

### 7.1 目标

- 支持计划内主从切换（switchover OpsTask）。
- 验证自动故障转移能力。
- 不绕过 Sentinel 直接提升 replica。

### 7.2 switchover 设计

switchover 动作触发 `SENTINEL FAILOVER <master-name>`，由 Sentinel 自主选择并提升新 primary。

#### 无 candidate 切换

```bash
#!/bin/sh
set -eu

master_name="${REDIS_COMPONENT_NAME}"
sentinel_port=${SENTINEL_SERVICE_PORT:-26379}

for sentinel in $(echo "$SENTINEL_POD_FQDN_LIST" | tr ',' '\n'); do
  if redis-cli -h "$sentinel" -p "$sentinel_port" -a "$SENTINEL_PASSWORD" \
       SENTINEL FAILOVER "$master_name" | grep -q "OK"; then
    echo "failover triggered on $sentinel"
    break
  fi
done
```

#### 有 candidate 切换

如果 Koda 提供 `KODA_SWITCHOVER_CANDIDATE_FQDN`，则：

1. 检查 candidate 当前角色为 secondary。
2. 将所有 Server 节点的 `replica-priority` 设置为 100，candidate 设置为 1。
3. 触发 `SENTINEL FAILOVER`。
4. 等待新 primary 变成 candidate。
5. 恢复所有节点的 `replica-priority` 为原值（或默认值 100）。

```bash
# 设置 candidate 最高优先级
for pod in $(echo "$REDIS_POD_FQDN_LIST" | tr ',' '\n'); do
  if [ "$pod" = "$KODA_SWITCHOVER_CANDIDATE_FQDN" ]; then
    redis-cli -h "$pod" -p "$service_port" -a "$REDIS_DEFAULT_PASSWORD" CONFIG SET replica-priority 1
  else
    redis-cli -h "$pod" -p "$service_port" -a "$REDIS_DEFAULT_PASSWORD" CONFIG SET replica-priority 100
  fi
done

# 触发 failover
redis-cli -h "$sentinel" -p "$sentinel_port" -a "$SENTINEL_PASSWORD" SENTINEL FAILOVER "$master_name"

# 等待并恢复优先级
# ...
```

> **决策**：v1 同时支持有 candidate 和无 candidate。无 candidate 为最小可用实现，有 candidate 增强可控性。

### 7.3 自动故障转移

自动 failover 由 Sentinel 原生完成，平台不干预：

1. primary Pod 被删除或网络分区。
2. Sentinel 判定主观下线（SDOWN）→ 客观下线（ODOWN）。
3. Sentinel 选举 leader。
4. Leader Sentinel 向选中的 replica 发送 `SLAVEOF NO ONE`。
5. 其他 replica 重新配置 `replicaof` 指向新 primary。
6. Koda 的 `roleProbe` 在下一个周期检测到角色变化，更新 Pod 标签，`writer` service 切换。

### 7.4 运维任务

- `redis/examples/opstasks/switchover.yaml`

示例：

```yaml
apiVersion: operations.koda.io/v1alpha1
kind: OpsTask
metadata:
  name: redis-switchover
spec:
  type: switchover
  target:
    applicationRef: redis-demo
    component: server
  parameters:
    # candidate 可选
    candidate: redis-demo-server-1
```

### 7.5 验收标准

| 层级 | 验收项 |
|---|---|
| 单元测试 | switchover 脚本 shellspec 测试通过 |
| 集成测试 | kind 中执行 switchover OpsTask，primary 切换到新节点；删除 primary Pod 后自动 failover |
| e2e | 切换/故障转移期间业务写入最终一致，无数据丢失（异步复制下可能丢失未同步数据，需文档说明） |

---

## 8. 迭代 6：账号管理

### 8.1 说明

本期所有系统账号（`default`、`replica`、`redis-sentinel`）均在迭代 1 的启动脚本中初始化，不引入业务账号，因此不实现 `accountProvision`。

本迭代内容 deferred 到后续版本。

### 8.2 后续版本待补充

- 设计业务账号 `app` 的 statements（create/update/delete）。
- 实现 Server `accountProvision` 脚本（执行 `ACL SETUSER` / `ACL SAVE`）。
- 更新 `componentdefinition-redis-server.yaml` 的 credential 配置，添加业务账号声明。
- 配置 `accountProvision.targetPodSelector: All`（依赖 Koda fan-out 能力）。

---

## 9. 迭代 7：示例整理与文档编写

### 9.1 目标

- 提供完整的安装、使用、运维示例。
- 更新顶层 README 中的 Redis 状态。
- 端到端走查验证所有功能。

### 9.2 示例清单

- `redis/examples/application/namespace.yaml`
- `redis/examples/application/application.yaml`（3+3 拓扑）
- `redis/examples/application/kustomization.yaml`
- `redis/examples/opstasks/switchover.yaml`
- `redis/examples/opstasks/reconfigure.yaml`
- `redis/examples/opstasks/horizontal-scale-out.yaml`
- `redis/examples/opstasks/horizontal-scale-in.yaml`

### 9.3 README 内容

`redis/README.md` 至少包含：

- 安装 AppPack
- 创建 Application（3+3 拓扑示例）
- 连接示例（通过 `writer` service 写入，`redis` service 读取）
- 扩容/缩容示例
- 切换示例
- 参数热更新示例
- 故障转移说明与限制（异步复制可能丢数据）
- 拓扑约束（至少 3 Sentinel）

### 9.4 端到端走查

- 安装 AppPack
- 创建 Application
- 写入/读取数据
- 执行 switchover
- 执行 scale out/in
- 执行 reconfigure
- 验证故障转移

### 9.5 交付前检查

- [ ] chart 版本号确认
- [ ] 镜像引用确认
- [ ] 脚本可执行权限确认
- [ ] 空壳/占位内容清理
- [ ] 设计文档与实现一致性检查

---

## 10. 开放问题与风险

### 10.1 待确认决策

| 问题 | 当前倾向 | 需要确认 |
|---|---|---|
| 启动方式是否锁定 entrypoint？ | 是 | 是否与 Koda init container 支持有冲突 |
| `/data/redis.conf` 旧热更新参数是否保留？ | 保留（方案 A） | 启动脚本实现复杂度是否可接受 |
| Sentinel memberLeave 是否实现？ | 可选，最小版本不做 | 缩容后视图不一致是否可接受 |
| switchover 是否必须支持 candidate？ | v1 支持，但无 candidate 为最小可用 | Koda switchover 动作是否传 candidate |
| 热更新参数初始清单 | 10 个参数 | 是否满足常见运维需求 |

### 10.2 风险

| 风险 | 影响 | 缓解 |
|---|---|---|
| 跨组件 env 注入失败 | 高 | 迭代 2 集成测试中重点验证；启动脚本能处理空列表回退 |
| `/data/redis.conf` 残留参数冲突 | 中 | 启动脚本严格区分骨架参数与保留参数；单元测试覆盖 |
| roleProbe 更新延迟导致 writer service 短暂不一致 | 中 | 客户端应通过 Sentinel 发现 primary，writer service 作为兼容入口 |
| kind 环境无法模拟真实网络分区 | 中 | e2e 可在已有 k8s 集群补充验证 |
| 异步复制在 failover 下丢数据 | 低（已知限制） | 文档明确说明；重要场景建议应用层做幂等 |

### 10.3 下一步建议

按以下顺序推进风险最高、依赖最多的项：

1. 确认 entrypoint 启动方式与 Koda 的兼容性。
2. 在迭代 2 中验证跨组件 `credentialFieldRef` / `componentFieldRef` 注入。
3. 完成迭代 3-5 设计后，同步更新 `tasks.md` 中对应任务。
4. 迭代 4 开始前，确定热更新参数清单并同步到 `configfiledefinition-redis-server.yaml`。
