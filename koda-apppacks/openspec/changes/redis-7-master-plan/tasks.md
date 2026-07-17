# Redis 7 AppPack 实施总控任务清单

## 1. 阶段 change 初始化

- [ ] 1.1 确认阶段 0 至阶段 7 的 change 名称与目标一致
- [ ] 1.2 为尚未创建的阶段 change 创建 OpenSpec change 目录
- [ ] 1.3 在 design.md 中更新各阶段 change 的当前状态

## 2. 依赖关系维护

- [ ] 2.1 校验阶段依赖图无循环依赖
- [ ] 2.2 确认 phase3-scaling 对 phase6-account-management 的依赖是否必要
- [ ] 2.3 在 design.md 中同步调整依赖关系

## 3. 编号与规格一致性

- [ ] 3.1 修正 proposal.md 中的迭代编号（补充 iteration 6 说明）
- [ ] 3.2 统一 proposal.md 与 design.md 中的阶段数量描述
- [ ] 3.3 修正 design.md 中 delta spec 路径描述（`specs/<capability>/spec.md`）

## 4. GitHub Issue 同步

- [ ] 4.1 确认 issue #223 及 #329-#339 均存在于 `kubesphere-extensions/koda`
- [ ] 4.2 在顶层 issue #223 中汇总各阶段 issue 链接
- [ ] 4.3 当阶段 change 状态变化时，在对应 issue 中评论更新

## 5. 状态追踪

- [ ] 5.1 每周检查一次 design.md 中的阶段状态表
- [ ] 5.2 当阶段 change 归档时，更新状态表为「已归档」
- [ ] 5.3 当新增风险或阻塞时，更新风险与缓解表格
