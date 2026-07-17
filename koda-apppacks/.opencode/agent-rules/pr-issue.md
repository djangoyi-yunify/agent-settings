## PR和Issue规则
### 远端代码仓库
- 远端代码仓库提供商：github
- 远端仓库：djangoyi-yunify/koda-apppacks（对应本地仓库）
- fork的上游仓库：kubesphere-extensions/koda-apppacks

> 当前已通过浏览器完成fork

### PR 说明
- PR的目标分支：main
- 候选的PR的目标仓库
  - 私人仓库：djangoyi-yunify/koda-apppacks
  - 公共仓库：kubesphere-extensions/koda-apppacks
- 创建/更新PR的要求
  - 当用户没有明确说目标仓库时，PR的目标仓库是**公共仓库**
  - 当用户指定目标仓库时，根据用户提示选择正确的目标仓库

### Issue
- Issue的目标仓库
  - kubesphere-extensions/koda-apppacks
  - kubesphere-extensions/koda
- 创建/更新Issue的要求
  - 当用户没有明确说目标仓库时，Issue的目标仓库是kubesphere-extensions/koda-apppacks
  - 当用户指定目标仓库时，根据用户提示选择正确的目标仓库

### 撰写PR/Issue
#### PR标题
- 格式：`类型(功能模块): 动词+具体内容概述`
  - 标题中如果包含文件名、CRD名或字段名，则**必须**使用确切的名称
  - 标题中，如果单个词不能表达完整意图，则需要使用词组来进行表达，例如
    - ConfigItems skeleton building
    - merge strategy
    - reconcile integration
- 举例
  - docs(parameter, superpowers spec): design ComponentParameter Controller, step1
  - docs(parameter, koda design): design ConfigFileDefinition CRD
  - codes(paramter): impl ComponentParameter controller Step 1 - reconciler skeleton, status aggregation, and phase management

#### Issue标题
- 格式: `类型(功能模块): Bug/需求/建议概述`

#### PR/Issue内容
- MarkDown格式
- 保持内容简洁，概括PR/Issue所包含内容的要点
- 新建/更新内容：把内容保存到临时目录，再使用 `gh -F` 或 `gh --body-file` 指定内容路径

### 处理PR/Issue中的修改意见
#### 修改意见来源
- PR中的comment
- issue或issue中的comment

#### 原则
针对修改意见，必须根据实际情况进行分析和判断，不要盲目全盘接受或否定。在此过程中，需要对涉及到的本项目代码、依赖项目的代码、参考项目的代码、各类设计文档、调研文档进行深入理解，避免误判。

#### 步骤1：记录修改意见
使用临时文件记录修改意见和处理情况

##### 路径和命名
临时文件的存放路径 /tmp

命名规则
- PR：PR{number}-{date}，例如：PR311-20260617
- issue：ISSUE{number}-{date}，例如：ISSUE130-20260617

##### 记录内容
记录内容包含1个或多个**内容单元**和1个**阶段总结**

**内容单元**是PR、issue中提及的修改意见，按照意图和内容拆分的最小处理单位

格式举例：
```
## Comment 1: Namespace default normalization in buildConfigItemsSkeleton

**Location**: `internal/controller/parameters/skeleton.go`, line 53
**Source**: gemini-code-assist
**Priority**: high

**Original comment**:
If `tpl.Namespace` is empty, the API server will default `ConfigTemplateSpec.Namespace` to `"default"` due to the `+kubebuilder:default="default"` marker in the CRD definition. If the skeleton is built with an empty namespace, comparing it with the defaulted spec from the API server will result in a mismatch on every reconciliation, causing an infinite reconciliation loop. We should normalize the namespace to `"default"` when it is empty.

**Upstream investigation**:
- Upstream `ComponentFileTemplate.Namespace` has `+kubebuilder:default="default"` marker (`apis/apps/v1/componentdefinition_types.go:1080`).
- Upstream `generateConfigTemplateItem` in `pkg/parameters/parameter_utils.go:75-84` uses `template.DeepCopy()` directly for `ConfigSpec`. It does NOT explicitly normalize empty namespace to "default".
- However, upstream flow reads `templates` from ComponentDefinition (not from in-memory skeleton comparison). The comparison happens via `reflect.DeepEqual(compParam.Spec.ConfigItemDetails, merged.Spec.ConfigItemDetails)` after both sides have been through the API server defaulting.
- In koda, `buildConfigItemsSkeleton` builds skeleton in-memory and compares against `cp.Spec.ConfigItems` which may have been defaulted by the API server. If the CRD default is applied to existing spec items, skeleton with empty namespace will mismatch.
- The CRD `config/crd/bases/parameters.koda.io_componentparameters.yaml` does have `default: default` for `namespace` field.

**判定结果**: 接受
**理由**: 这是 Koda 自身代码路径的问题。Upstream 的 comparison 发生在两者都经过 API server defaulting 之后，而 Koda 的 skeleton 是本地构建的，可能携带空 namespace，与 API server 已 default 的 existing spec 比较会产生 infinite loop。Normalize 到 "default" 是必要的防御性处理。

**处理结果**: 处理中
```
**阶段总结**记录当前已知的修改意见处理状态的汇总情况

格式举例：
```
| # | Comment | Decision | Status |
|---|---------|----------|--------|
| 1 | Namespace default normalization | 接受 | 待处理 |
| 2 | Nil/empty slice equivalence | 部分接受 | 待处理 |
| 3 | Deep copy ConfigSpec | 拒绝 | 无需处理 |
| 4 | Return patch error on Component NotFound | 接受 | 待处理 |
| 5 | Return patch error on ComponentDefinition NotFound | 接受 | 处理完毕 |
```
#### 步骤2：收尾工作
时机：当前已知的所有修改意见都处理完毕

动作
- 执行`git commit` 和 `git push`
- 更新临时文件中**内容单元**的判定结果和处理结果，以及**阶段总结**中处理状态的汇总情况
- 在对应的PR或issue中追加comment，简要总结**本次**针对修改意见的处理情况

### 注意事项
- 标题/内容**尽量**使用简体中文
- 专业词汇、专有名词等，**必须**使用英语
- 如果使用简体中文表达时会产生歧义，则使用英语
- 处理修改意见时，应检查**内容单元**和**阶段总结**中的处理状态，不要重复处理已处理过的修改意见

