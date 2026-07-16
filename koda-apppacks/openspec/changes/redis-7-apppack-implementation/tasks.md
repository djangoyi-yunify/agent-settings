# Redis 7 AppPack 实施任务清单

## 阶段 0：基础骨架

### 目录与 chart 结构

- [ ] 创建 `redis/apppack/chart/` 目录结构（与 kubeblocks-addons 内部结构一致）
- [ ] 创建 `redis/apppack/chart/config/` 目录
- [ ] 创建 `redis/apppack/chart/scripts/` 目录
- [ ] 创建 `redis/apppack/chart/scripts-ut-spec/` 目录
- [ ] 创建 `redis/apppack/chart/templates/` 目录
- [ ] 创建 `redis/examples/application/` 目录结构
- [ ] 创建 `redis/examples/opstasks/` 目录结构
- [ ] 创建 `redis/test/` 目录结构

### Helm chart 基础

- [ ] 编写 `redis/apppack/chart/Chart.yaml`
- [ ] 编写 `redis/apppack/chart/values.yaml`
- [ ] 编写 `redis/apppack/chart/README.md`
- [ ] 编写 `redis/apppack/chart/templates/_helpers.tpl`

### AppPack 与定义层 CRD

- [ ] 编写 `redis/apppack/apppack.yaml`
- [ ] 编写 `redis/apppack/chart/templates/applicationdefinition.yaml`
- [ ] 编写 `redis/apppack/chart/templates/componentmatrix.yaml`
- [ ] 编写 `redis/apppack/chart/templates/componentdefinition-redis-server.yaml`（骨架）
- [ ] 编写 `redis/apppack/chart/templates/componentdefinition-redis-sentinel.yaml`（骨架）
- [ ] 编写 `redis/apppack/chart/templates/configfiledefinition-redis-server.yaml`（骨架）
- [ ] Sentinel 配置不通过 ConfigFileDefinition 管理，由启动脚本生成在持久化卷中

### 配置文件

- [ ] 编写 `redis/apppack/chart/config/redis7-config.tpl`
- [ ] 编写 `redis/apppack/chart/config/redis7-config-constraint.cue`
- [ ] 编写 `redis/apppack/chart/config/redis-config-effect-scope.yaml`
- [ ] Sentinel 配置不放在 `config/` 或 `templates/`，由启动脚本生成

### 配置模板 ConfigMap

- [ ] 编写 `redis/apppack/chart/templates/redis-config-template.yaml`（对应 ConfigFileDefinition）
- [ ] 不编写 Sentinel 配置模板，Sentinel 配置由启动脚本生成

### 脚本 ConfigMap 占位

- [ ] 编写 Server 脚本 ConfigMap 模板（文件名后续细化）
- [ ] 编写 Sentinel 脚本 ConfigMap 模板（文件名后续细化）

### 本地单元测试工具链

- [ ] 配置 shellspec（与 kubeblocks-addons 一致）
- [ ] 配置 shellcheck
- [ ] 配置 helm lint / helm template 检查脚本
- [ ] 编写 `redis/apppack/chart/scripts-ut-spec/utils.sh`
- [ ] 编写 `redis/apppack/chart/scripts-ut-spec/` 最小示例（文件名后续细化）

### 集成测试环境抽象层

- [ ] 编写 `redis/test/hack/setup-cluster.sh`
- [ ] 编写 `redis/test/hack/install-koda.sh`
- [ ] 编写 `redis/test/hack/teardown-cluster.sh`
- [ ] 编写 `redis/test/README.md`（测试环境说明）

### 入口脚本

- [ ] 编写 `redis/Makefile`（`test-unit`、`test-integration`、`test-e2e` 目标）

### 阶段 0 验收

- [ ] `make test-unit` 通过
- [ ] 空壳 CRD 可在 kind / 已有 k8s 集群中 apply

---

## 迭代 1：Bootstrap / 创建

### Redis Server

- [ ] 设计 `redis.conf` 模板内容
- [ ] 实现 Server 启动脚本（文件名后续细化），采用主容器 entrypoint 方式
- [ ] 在启动脚本中生成 `/data/redis.conf`（include `/etc/conf/redis.conf`）
- [ ] 在启动脚本中实现 initAccount（default）密码初始化（通过 ACL 文件方式）
- [ ] 在启动脚本中创建系统用户 `replica`（主从复制）并写入 `masteruser` / `masterauth`
- [ ] 在启动脚本中创建系统用户 `redis-sentinel`（供 Sentinel 组件连接 Server）
- [ ] 实现 ACL 文件 `/data/users.acl` 初始化逻辑（幂等：清理旧系统账号行后重新写入）
- [ ] 在 `componentdefinition-redis-server.yaml` 中声明跨组件引用 `REDIS_SENTINEL_PASSWORD`（引用 Sentinel 组件 default 密码）
- [ ] 更新 `componentdefinition-redis-server.yaml` 的 runtime 与 volumes
  - `/data`：data PVC
  - `/etc/conf`：redis 配置模板 ConfigMap
  - `/scripts`：redis 脚本 ConfigMap

### Sentinel

- [ ] 设计 `sentinel.conf` 生成逻辑（不经过模板渲染）
- [ ] 实现 Sentinel 启动脚本（文件名后续细化），采用主容器 entrypoint 方式
- [ ] 在启动脚本中实现 `/data/sentinel/redis-sentinel.conf` 的 surgical update 逻辑
- [ ] 在启动脚本中配置 `sentinel sentinel-user redis-sentinel` / `sentinel sentinel-pass $SENTINEL_PASSWORD`
- [ ] 在 `componentdefinition-redis-sentinel.yaml` 中声明 `SENTINEL_USER` 环境变量（硬编码为 `redis-sentinel`）和 `SENTINEL_PASSWORD`
- [ ] 更新 `componentdefinition-redis-sentinel.yaml` 的 runtime 与 volumes
  - `/data`：data PVC
  - `/scripts`：sentinel 脚本 ConfigMap

### 测试

- [ ] 编写 Server 启动脚本的 shellspec 单元测试
- [ ] 编写 Sentinel 启动脚本的 shellspec 单元测试
- [ ] 在 kind 中验证单组件 Pod Ready
- [ ] 验证 `redis-cli PING` / `redis-cli -p 26379 PING sentinel`

---

## 迭代 2：角色感知与服务发现

### Redis Server

- [ ] 实现 roleProbe 脚本
- [ ] 实现 availableProbe 脚本
- [ ] 实现通过 Sentinel 发现当前主节点的逻辑
- [ ] 实现 postProvision 脚本：在 primary Pod 上向所有 Sentinel 执行 `SENTINEL monitor`
- [ ] 更新 `componentdefinition-redis-server.yaml` 的 lifecycle actions

### Sentinel

- [ ] 实现 Sentinel 可用探测脚本
- [ ] 更新 `componentdefinition-redis-sentinel.yaml` 的 lifecycle actions

### 跨组件集成

- [ ] 在 `componentdefinition-redis-server.yaml` 中通过 `credentialFieldRef` 引用 Sentinel 组件 default 账号密码
- [ ] 在 `componentdefinition-redis-server.yaml` 中声明 sentinel dependency（用于注入 `SENTINEL_POD_FQDN_LIST` 等）
- [ ] 在 `componentdefinition-redis-sentinel.yaml` 中声明 redis dependency（用于注入 Redis 组件信息）
- [ ] 确定 master name 命名规则（建议使用 `REDIS_COMPONENT_NAME`）

### 测试

- [ ] 编写 roleProbe / availableProbe 的 shellspec 测试
- [ ] 在 kind 中部署 3+3 拓扑
- [ ] 验证 Server 角色正确
- [ ] 验证 Sentinel 已监控 master

---

## 迭代 3：扩缩容

### Redis Server

- [ ] 更新 `componentdefinition-redis-server.yaml` 的 scaling 限制
- [ ] 实现 `sync-acl.sh` 并在 `componentdefinition-redis-server.yaml` 中声明 `memberJoin` action
- [ ] 验证 scale-out 后新 replica 自动通过 Sentinel 发现 primary 并加入复制
- [ ] 验证 scale-out 后 `sync-acl.sh` 把非 init 账号同步到新 replica
- [ ] 不在 Server 实现 memberLeave（接受 Sentinel 中残留的 known-replica）

### Sentinel

- [ ] 在 `componentdefinition-redis-sentinel.yaml` 中设置 `scaling.replicasLimit.minReplicas: 3, maxReplicas: 3`，固定 Sentinel 副本数
- [ ] 验证 3 Sentinel 正常组成集群（无需 memberJoin）
- [ ] 不实现 Sentinel memberLeave（本期不支持 Sentinel 扩缩容）

### 运维任务

- [ ] 编写 `redis/examples/opstasks/horizontal-scale-out.yaml`
- [ ] 编写 `redis/examples/opstasks/horizontal-scale-in.yaml`

### 测试

- [ ] 编写 `sync-acl.sh` 的 shellspec 测试
- [ ] 在 kind 中执行 scale out，验证新 replica 自动加入复制且非 init 账号存在
- [ ] 在 kind 中执行 scale in，接受 Sentinel 中残留的 known-replica
- [ ] 验证 Sentinel 副本数固定为 3，不支持扩缩容
- [ ] e2e：写入数据后 scale out，读取一致；业务账号在 scale out 后仍可在新 replica 上认证

---

## 迭代 4：配置变更

### Redis Server

- [ ] 实现 Server reconfigure 脚本
- [ ] 解析 `KODA_CONFIG_CHANGED_PARAMETERS` 并执行 `CONFIG SET` + `CONFIG REWRITE`
- [ ] 确定热更新参数清单
- [ ] 更新 `configfiledefinition-redis-server.yaml` 的 `managedParams`

### Sentinel

- [ ] 确定 Sentinel 是否需要 reconfigure
- [ ] 如需要，实现 Sentinel reconfigure 脚本

### 运维任务

- [ ] 编写 `redis/examples/opstasks/reconfigure.yaml`

### 测试

- [ ] 编写 reconfigure 的 shellspec 测试
- [ ] 在 kind 中修改 config variables，验证热更新生效
- [ ] 验证 Pod 未重建
- [ ] e2e：热更新 `maxmemory` 后业务正常

---

## 迭代 5：主从切换与故障转移

### Redis Server

- [ ] 实现 Server switchover 脚本
- [ ] 调用 `SENTINEL FAILOVER <master-name>`
- [ ] 更新 `componentdefinition-redis-server.yaml` 的 switchover action

### Sentinel

- [ ] 验证 Sentinel 原生 failover 行为符合预期
- [ ] 必要时调整 Sentinel 配置参数（down-after-milliseconds 等）

### 运维任务

- [ ] 编写 `redis/examples/opstasks/switchover.yaml`

### 测试

- [ ] 编写 switchover 的 shellspec 测试
- [ ] 在 kind 中执行 switchover opstask，验证业务切换
- [ ] 在 kind 中删除 primary Pod，验证自动 failover
- [ ] e2e：故障转移期间业务恢复

---

## 迭代 6：账号管理（本期 deferred）

> 说明：本期所有系统账号（`default`、`replica`、`redis-sentinel`）均在迭代 1 的启动脚本中初始化，不引入业务账号，因此不实现 `accountProvision`。本迭代内容移至后续版本。

### Redis Server

- [ ] 设计业务账号 `app` 的 statements（create/update/delete）
- [ ] 实现 Server accountProvision 脚本（执行 `ACL SETUSER` / `ACL SAVE`）
- [ ] 更新 `componentdefinition-redis-server.yaml` 的 credential 配置，添加业务账号声明
- [ ] 配置 `accountProvision.targetPodSelector: All`（依赖 #259 fan-out）

### Sentinel

- [ ] 确定 Sentinel 是否需要独立账号管理
- [ ] 如需要，实现 Sentinel accountProvision 脚本

### 测试

- [ ] 编写 accountProvision 的 shellspec 测试
- [ ] 在 kind 中验证 `ACL LIST` 包含业务账号
- [ ] 使用业务账号连接 Redis 写入/读取

---

## 迭代 7：示例整理与文档编写

### 示例

- [ ] 编写 `redis/examples/application/namespace.yaml`
- [ ] 编写 `redis/examples/application/application.yaml`（3+3 拓扑）
- [ ] 编写 `redis/examples/application/kustomization.yaml`
- [ ] 整理所有 opstasks，统一命名与注释

### 文档

- [ ] 编写 `redis/README.md`
  - 安装说明
  - 拓扑说明
  - 连接示例
  - 扩容/缩容示例
  - 切换示例
  - 参数热更新示例
- [ ] 更新顶层 `README.md` 中的 Redis 状态

### 端到端走查

- [ ] 按 README 完整操作一遍
- [ ] 安装 AppPack
- [ ] 创建 Application
- [ ] 写入/读取数据
- [ ] 执行 switchover
- [ ] 执行 scale out/in
- [ ] 执行 reconfigure
- [ ] 验证故障转移

### 交付前检查

- [ ] chart 版本号确认
- [ ] 镜像引用确认
- [ ] 脚本可执行权限确认
- [ ] 空壳/占位内容清理
- [ ] 设计文档与实现一致性检查

---

## 可选后续

- [ ] 编写 `.github/workflows/redis-test.yaml`（CI 标准化）
- [ ] 补充 shellspec 覆盖率
- [ ] 补充 Grafana dashboard
- [ ] 补充备份恢复设计（下个大版本）
