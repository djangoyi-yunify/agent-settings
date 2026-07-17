# Redis 7 AppPack 业务账号管理任务清单

## 业务账号设计

- [ ] 设计业务账号 `app` 的默认权限
- [ ] 设计 create/update/delete statements
- [ ] 确定账号密码生成与轮换策略

## accountProvision 脚本

- [ ] 实现 `redis/apppack/chart/scripts/redis-account-provision.sh`
- [ ] 解析 `KODA_ACCOUNT_ACTION`、`KODA_ACCOUNT_NAME`、`KODA_ACCOUNT_PASSWORD`
- [ ] 实现 create 操作：`ACL SETUSER`
- [ ] 实现 update 操作：`ACL SETUSER`
- [ ] 实现 delete 操作：`ACL DELUSER`
- [ ] 执行 `ACL SAVE` 持久化

## ComponentDefinition 更新

- [ ] 更新 `componentdefinition-redis-server.yaml`：
  - [ ] 添加 `app` 业务账号 credential 声明
  - [ ] 声明 `accountProvision` lifecycle action
  - [ ] 配置 `targetPodSelector: All`

## 启动脚本协调

- [ ] 确保 `redis-start.sh` 在首次创建时不覆盖业务账号
- [ ] 验证 `/data/users.acl` 中系统账号和业务账号共存

## 测试

- [ ] 编写 `redis-account-provision.sh` shellspec 测试
- [ ] `shellcheck` 无 error
- [ ] `make test-unit` 通过
- [ ] 在 kind 中验证 accountProvision 创建账号
- [ ] 在 kind 中验证所有 Server Pod 的 `ACL LIST` 一致

## e2e

- [ ] 使用业务账号连接 Redis 写入/读取
- [ ] scale-out 后在新 replica 上业务账号可用
- [ ] 更新业务账号密码后生效
