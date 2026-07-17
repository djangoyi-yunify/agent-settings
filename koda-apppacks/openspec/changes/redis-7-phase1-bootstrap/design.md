# Redis 7 AppPack Bootstrap / 创建设计

## 1. 设计原则

本 change 只处理**首次创建**场景：

- Redis Server 和 Sentinel 在首次创建时启动并建立拓扑。
- 启动脚本不需要处理 Pod 重启时的配置局部更新。
- 配置生成逻辑聚焦于创建时已知信息：Pod 身份、Service DNS、静态端口、密码。

> **关于"局部更新"**：指 Pod 重启时，启动脚本保留 `sentinel.conf` 中 Sentinel 运行时写入的 `sentinel monitor`、`known-replica`、`known-sentinel`、epoch 等状态，仅重新计算并更新 announce-ip、announce-port、port 等动态参数。本 change 不包含局部更新，后续涉及 Pod 重建的阶段再扩展。

## 2. 系统账号与密码

### 账号清单

| 用户名 | 所在组件 | 用途 | 密码来源 |
|---|---|---|---|
| `default` | Redis Server | 业务客户端访问 | Redis Server 自己的 initAccount |
| `replica` | Redis Server | 主从复制（`masteruser` / `masterauth`） | 复用 Redis Server `default` 密码 |
| `redis-sentinel` | Redis Server | 供 Sentinel 组件连接 Server | Redis Sentinel 的 initAccount `default` 密码 |
| `default` | Redis Sentinel | Sentinel 进程自身认证 | Redis Sentinel 自己的 initAccount |

### 跨组件密码引用

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

### ACL 初始化

启动脚本在首次创建时生成 `/data/users.acl`：

```bash
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
```

## 3. Redis Server 启动流程

```
启动脚本 redis-start.sh
│
├─ 1. 加载公共函数库
├─ 2. 解析当前 Pod 的 announce 地址（FQDN）
├─ 3. 生成 /data/redis.conf
│   ├─ include /etc/conf/redis.conf（只读模板）
│   ├─ port
│   ├─ replica-announce-ip / replica-announce-port
│   ├─ 确定主从关系
│   ├─ masteruser replica / masterauth
│   └─ aclfile /data/users.acl
├─ 4. 生成 /data/users.acl 系统账号
└─ 5. exec redis-server /data/redis.conf
```

### 主从关系确定

首次创建时，直接以序号为 0 的 Pod 作为 primary，其他 Pod 设置 `replicaof` 指向 Pod-0 的 `replica-announce-ip` 和 `replica-announce-port`。

```bash
# 假设 REDIS_POD_FQDN_LIST 包含所有 Pod FQDN，按序号排序
primary_host=$(echo "$REDIS_POD_FQDN_LIST" | cut -d',' -f1)
primary_announce_ip=$(get_announce_ip "$primary_host")
primary_announce_port=${REDIS_SERVICE_PORT:-6379}

current_pod_index=$(get_current_pod_index)

if [ "$current_pod_index" -eq 0 ]; then
  # primary 不设置 replicaof
  :
else
  echo "replicaof $primary_announce_ip $primary_announce_port" >> /data/redis.conf
fi
```

说明：
- 不查询 Sentinel 确定 primary。
- 使用 Pod-0 的 `replica-announce-ip` 和 `replica-announce-port` 作为 `replicaof` 目标。
- 后续 failover 或 switchover 由 Sentinel 负责调整拓扑。

## 4. Redis Sentinel 启动流程

```
启动脚本 redis-sentinel-start.sh
│
├─ 1. 加载公共函数库
├─ 2. 解析当前 Pod 的 announce 地址（FQDN）
├─ 3. 创建 /data/sentinel/redis-sentinel.conf
│   ├─ port
│   ├─ announce-ip / announce-port
│   ├─ sentinel sentinel-user redis-sentinel
│   └─ sentinel sentinel-pass $SENTINEL_PASSWORD
└─ 4. exec redis-server /data/sentinel/redis-sentinel.conf --sentinel
```

> 注意：本 change 不处理 Sentinel 重启时的配置局部更新。后续涉及 Pod 重建的阶段再扩展。
> 局部更新的对象是 `sentinel.conf`：保留 `sentinel monitor`、`known-replica`、`known-sentinel`、epoch 等运行时状态，仅更新 announce-ip、announce-port、port 等动态参数。

## 5. 脚本层面的 TLS 支持

本 change 在脚本层面为后续 TLS 能力做准备，但**不实际启用 TLS**，也**不测试 TLS 连接能力**。

### TLS 开关环境变量

```yaml
# Redis Server ComponentDefinition
env:
- name: REDIS_TLS_ENABLED
  value: "false"

# Redis Sentinel ComponentDefinition
env:
- name: REDIS_TLS_ENABLED
  value: "false"
```

> 本期 `REDIS_TLS_ENABLED` 始终为 `"false"`，由 Koda 平台根据配置模板渲染。

### redis-cli TLS 参数适配

公共函数库提供一个 `build_redis_cli_args` 函数，根据 `REDIS_TLS_ENABLED` 返回正确的 `redis-cli` 参数：

```bash
build_redis_cli_args() {
  local base_args="$1"
  if [ "${REDIS_TLS_ENABLED:-false}" = "true" ]; then
    echo "$base_args --tls --cert ${REDIS_TLS_CERT_FILE:-/etc/tls/server.crt} --key ${REDIS_TLS_KEY_FILE:-/etc/tls/server.key} --cacert ${REDIS_TLS_CA_CERT_FILE:-/etc/tls/ca.crt}"
  else
    echo "$base_args"
  fi
}
```

所有脚本中使用 `redis-cli` 的地方都通过该函数构造参数。

### TLS 配置参数的来源

- `tls-port`、`tls-cert-file`、`tls-key-file`、`tls-ca-cert-file`、`tls-replication` 等参数由 Koda 配置模板渲染到 `/etc/conf/redis.conf`
- 启动脚本通过 `include /etc/conf/redis.conf` 引入这些参数
- 启动脚本自身**不写入**任何 TLS 相关参数

## 6. ComponentDefinition 关键配置

### Redis Server

- **podManagementPolicy**: `OrderedReady`
- **roles**: `primary` / `secondary`
- **volumes**: `data` PVC 挂载到 `/data`
- **configs**: `redis-config` ConfigMap 挂载到 `/etc/conf`
- **env**: `REDIS_DEFAULT_PASSWORD`、`REDIS_REPL_PASSWORD`、`REDIS_SENTINEL_PASSWORD`、`REDIS_TLS_ENABLED`、`REDIS_POD_FQDN_LIST`、`POD_NAME`
- **lifecycle.actions**: 本期不声明具体 action（留到 phase2）

### Redis Sentinel

- **podManagementPolicy**: `Parallel`
- **volumes**: `data` PVC 挂载到 `/data`
- **env**: `SENTINEL_USER`、`SENTINEL_PASSWORD`、`REDIS_TLS_ENABLED`
- **lifecycle.actions**: 本期不声明具体 action

## 7. 测试策略

### 单元测试

- `redis-start.sh` shellspec：验证 `/data/redis.conf` 生成、ACL 生成、角色确定逻辑
- `redis-sentinel-start.sh` shellspec：验证 `/data/sentinel/redis-sentinel.conf` 生成
- `utils.sh` shellspec：验证 `build_redis_cli_args` 在 TLS 开启/关闭时返回正确参数
- `shellcheck` 无 error

### 集成测试

- 在 kind 中创建 Application（TLS 开关关闭）
- Redis Server 与 Sentinel 均 Ready
- `redis-cli PING` / `redis-cli -p 26379 PING sentinel` 通过
- `redis-cli INFO replication` 显示正确主从关系

### e2e

- 通过 primary service 写入数据
- 在 replica 上读取
- Sentinel 返回当前 primary 地址

### TLS 测试边界

- 不验证实际的 TLS 连接能力
- 不验证 TLS 证书挂载
- 仅验证脚本能根据 `REDIS_TLS_ENABLED` 正确构造 `redis-cli` 参数
