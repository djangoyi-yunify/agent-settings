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

### 注意事项
- 标题/内容**尽量**使用简体中文
- 专业词汇、专有名词等，**必须**使用英语
- 如果使用简体中文表达时会产生歧义，则使用英语

