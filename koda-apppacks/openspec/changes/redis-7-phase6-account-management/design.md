# Redis 7 AppPack 业务账号管理设计

## 1. 设计原则

- 系统账号（`default`、`replica`、`redis-sentinel`）在首次创建时由启动脚本初始化。
- 业务账号通过 `accountProvision` 动作管理，支持 create/update/delete。
- `accountProvision` 需要在所有 Redis Server Pod 上执行，因为 ACL 变更不会通过复制传播。
- Sentinel 本期不实现业务账号管理。

## 2. 业务账号设计

### 默认业务账号：`app`

| 属性 | 值 |
|---|---|
| 用户名 | `app` |
| 权限 | `~* &* +@all`（与 `default` 类似，但用于业务访问） |
| 密码来源 | Koda 为 `app` 账号生成的 credential |

### 账号 statements

```yaml
create:
  - "ACL SETUSER app on #%s ~* &* +@all"
update:
  - "ACL SETUSER app on #%s ~* &* +@all"
delete:
  - "ACL DELUSER app"
```

> 实际 statements 由 Koda 根据 credential 生成，脚本负责执行。

## 3. accountProvision 脚本

```bash
#!/bin/sh
set -eu

service_port=${SERVICE_PORT:-6379}
auth_args=""
[ -n "$REDIS_DEFAULT_PASSWORD" ] && auth_args="-a $REDIS_DEFAULT_PASSWORD"

# Koda 注入的账号操作信息
# KODA_ACCOUNT_ACTION: create/update/delete
# KODA_ACCOUNT_NAME: 账号名
# KODA_ACCOUNT_PASSWORD: 密码（hash 前）

action="${KODA_ACCOUNT_ACTION:-create}"
username="${KODA_ACCOUNT_NAME:-}"
password="${KODA_ACCOUNT_PASSWORD:-}"

if [ -z "$username" ] || [ -z "$password" ]; then
  echo "missing account name or password"
  exit 1
fi

password_hash=$(echo -n "$password" | sha256sum | cut -d' ' -f1)

case "$action" in
  create|update)
    redis-cli $auth_args -h localhost -p "$service_port" \
      ACL SETUSER "$username" on "#$password_hash" '~*' '&*' '+@all'
    ;;
  delete)
    redis-cli $auth_args -h localhost -p "$service_port" \
      ACL DELUSER "$username"
    ;;
  *)
    echo "unsupported account action: $action"
    exit 1
    ;;
esac

redis-cli $auth_args -h localhost -p "$service_port" ACL SAVE
```

## 4. ComponentDefinition 更新

```yaml
credentials:
- name: default
  # ... 已存在
- name: app
  username:
    value: app
  password:
    # Koda 自动生成
lifecycle:
  actions:
    accountProvision:
      exec:
        container: redis
        command:
        - /bin/sh
        - /scripts/redis-account-provision.sh
      targetPodSelector: All
```

> `targetPodSelector: All` 依赖 Koda fan-out 能力。如果本期 Koda 不支持，
> 需要与平台团队确认替代方案。

## 5. 与 memberJoin 的集成

phase3-scaling 的 `memberJoin` 通过 `sync-acl.sh` 从 peer 同步非 init 账号。
业务账号创建后，scale-out 时新 replica 会自动通过 `sync-acl.sh` 获取这些账号。

## 6. 测试策略

### 单元测试

- `redis-account-provision.sh` shellspec：
  - 测试 create 操作
  - 测试 update 操作
  - 测试 delete 操作
  - 测试缺少参数时的错误处理

### 集成测试

- 在 kind 中通过 accountProvision 创建 `app` 账号
- 验证所有 Server Pod 的 `ACL LIST` 包含 `app`
- 使用 `app` 账号连接 Redis 执行 PING/SET/GET

### e2e

- 写入业务数据
- scale-out 后在新 replica 上用 `app` 账号读取
- 更新 `app` 密码后使用新密码连接
