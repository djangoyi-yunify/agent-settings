# Redis 7 AppPack Bootstrap / 创建任务清单

## Redis Server 启动脚本

- [ ] 实现 `redis/apppack/chart/scripts/redis-start.sh`
- [ ] 加载公共函数库
- [ ] 解析当前 Pod FQDN 作为 `replica-announce-ip`
- [ ] 生成 `/data/redis.conf`：
  - [ ] 写入 `include /etc/conf/redis.conf`
  - [ ] 写入 `port`
  - [ ] 写入 `replica-announce-ip` 和 `replica-announce-port`
  - [ ] 根据当前 Pod 序号确定是否写入 `replicaof`（Pod-0 不写入，其他 Pod 写入 Pod-0 的 replica-announce-ip/port）
  - [ ] 写入 `masteruser replica` 和 `masterauth`
  - [ ] 写入 `aclfile /data/users.acl`
- [ ] 生成 `/data/users.acl`：
  - [ ] `default` 用户
  - [ ] `replica` 用户
  - [ ] `redis-sentinel` 用户
- [ ] 最后 `exec redis-server /data/redis.conf`

## Redis Sentinel 启动脚本

- [ ] 实现 `redis/apppack/chart/scripts/redis-sentinel-start.sh`
- [ ] 加载公共函数库
- [ ] 解析当前 Pod FQDN 作为 `announce-ip`
- [ ] 创建 `/data/sentinel/redis-sentinel.conf`：
  - [ ] 写入 `port`
  - [ ] 写入 `announce-ip` 和 `announce-port`
  - [ ] 写入 `sentinel sentinel-user redis-sentinel`
  - [ ] 写入 `sentinel sentinel-pass $SENTINEL_PASSWORD`
- [ ] 最后 `exec redis-server /data/sentinel/redis-sentinel.conf --sentinel`

## TLS 开关与 redis-cli 参数适配

- [ ] 实现公共函数 `build_redis_cli_args`：
  - [ ] 读取 `REDIS_TLS_ENABLED` 环境变量
  - [ ] TLS 开启时追加 `--tls --cert <cert> --key <key> --cacert <ca>`
  - [ ] TLS 关闭时不追加 TLS 参数
- [ ] 所有使用 `redis-cli` 的脚本通过 `build_redis_cli_args` 构造参数
- [ ] 在 `componentdefinition-redis-server.yaml` 中声明 `REDIS_TLS_ENABLED` 环境变量（默认 `"false"`）
- [ ] 在 `componentdefinition-redis-sentinel.yaml` 中声明 `REDIS_TLS_ENABLED` 环境变量（默认 `"false"`）

## ComponentDefinition 更新

- [ ] 更新 `componentdefinition-redis-server.yaml`：
  - [ ] 声明 `REDIS_DEFAULT_PASSWORD` 环境变量
  - [ ] 声明 `REDIS_REPL_PASSWORD` 环境变量
  - [ ] 声明 `REDIS_SENTINEL_PASSWORD` 环境变量（跨组件引用 Sentinel）
  - [ ] 声明 `SENTINEL_COMPONENT_NAME` 和 `SENTINEL_POD_FQDN_LIST`（用于 Sentinel 查询）
  - [ ] 声明 `REDIS_TLS_ENABLED` 环境变量
  - [ ] 声明 volumes、configs、scripts 挂载
- [ ] 更新 `componentdefinition-redis-sentinel.yaml`：
  - [ ] 声明 `SENTINEL_USER` 环境变量
  - [ ] 声明 `SENTINEL_PASSWORD` 环境变量
  - [ ] 声明 `REDIS_TLS_ENABLED` 环境变量
  - [ ] 声明 volumes、scripts 挂载

## 配置模板

- [ ] 完善 `redis7-config.tpl`（最小有效配置，不强制包含 TLS 参数）
- [ ] 完善 `redis7-config-constraint.cue`（基础约束）
- [ ] 完善 `redis-config-effect-scope.yaml`（基础 scope）

## 测试

- [ ] 编写 `redis-start.sh` shellspec 测试
- [ ] 编写 `redis-sentinel-start.sh` shellspec 测试
- [ ] 编写 `utils.sh` shellspec 测试（验证 `build_redis_cli_args` 在 TLS 开启/关闭时行为正确）
- [ ] `shellcheck` 无 error
- [ ] `make test-unit` 通过
- [ ] 在 kind 中验证单组件 Pod Ready（TLS 关闭）
- [ ] 在 kind 中验证 `redis-cli PING` / `redis-cli -p 26379 PING sentinel`（非 TLS）
- [ ] 在 kind 中验证 `INFO replication` 主从关系正确（非 TLS）

## e2e

- [ ] 通过 primary service 写入数据（非 TLS）
- [ ] 在 replica 上读取数据（非 TLS）
- [ ] 通过 Sentinel 查询当前 primary 地址（非 TLS）

## 明确不做

- [ ] 不生成 TLS 配置参数（由 Koda 模板负责）
- [ ] 不挂载 TLS Secret
- [ ] 不测试实际 TLS 连接能力
- [ ] 不实现业务账号管理和 `accountProvision`
