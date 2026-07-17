# Redis 7 AppPack 实施路线图规格

## ADDED Requirements

### Requirement: Redis 7 AppPack 按阶段拆分实施
Redis 7 AppPack 的实施 SHALL 拆分为一个总控 change 和多个阶段 change，每个阶段 change 对应一个可独立验收的里程碑。

#### Scenario: 阶段 change 存在
- **WHEN** 查看 `openspec/changes/` 目录
- **THEN** 存在 `redis-7-master-plan` 总控 change
- **AND** 存在 `redis-7-phase0-skeleton` 阶段 change
- **AND** 存在 `redis-7-phase1-bootstrap` 阶段 change

### Requirement: 每个阶段 change 具有明确的验收标准
每个阶段 change SHALL 在 `proposal.md` 中定义验收标准，且验收标准可被独立验证。

#### Scenario: 阶段 change 包含验收标准
- **WHEN** 查看任意阶段 change 的 `proposal.md`
- **THEN** 存在「验收标准」章节
- **AND** 验收标准包含可验证的条目

### Requirement: 阶段 change 之间存在明确的依赖关系
每个阶段 change SHALL 声明其所依赖的前置阶段 change，且依赖关系不得形成循环。

#### Scenario: 依赖关系可被读取
- **WHEN** 查看 `redis-7-master-plan/design.md` 中的阶段清单
- **THEN** 每个阶段都有明确的「依赖」列
- **AND** 依赖图中不存在循环依赖

### Requirement: 阶段变更通过 GitHub issue 跟踪
每个阶段 change SHALL 对应一个 GitHub issue，用于跟踪实施状态和概要信息。

#### Scenario: Issue 映射完整
- **WHEN** 查看 `redis-7-master-plan/design.md` 中的 issue 映射表
- **THEN** 每个阶段 change 都对应一个 issue 编号
- **AND** 所有 issue 位于同一仓库

### Requirement: 能力规格随阶段 change 归档逐步填充
Redis 7 AppPack 的主规格 SHALL 通过各阶段 change 的 delta spec 归档逐步生成，而不是在项目实施前一次性写完整。

#### Scenario: 主规格目录初始为空
- **WHEN** 查看 `openspec/specs/redis-7/` 目录
- **THEN** 在第一个阶段 change 归档前，该目录为空或仅包含占位文件

#### Scenario: 阶段归档后生成主规格
- **WHEN** 一个阶段 change 完成并执行 `openspec archive`
- **THEN** 该阶段贡献的 capability delta 被合并到 `openspec/specs/redis-7/`
