

## Git commit提交规范
提交信息使用 Conventional Commits，确保日志可读、便于生成变更记录。

### 提交粒度

- 单次提交只做一类变更（功能/修复/文档）
- 提交前先整理改动，避免混入无关格式化或临时调试
- 每个提交应可构建、可运行，方便回滚
- 大改动拆分为多个可审查的小提交

### 提交信息格式

`<type>[(scope)]: <summary>  [body]  [footer]`

- type 建议：feat、fix、docs、refactor、test、chore、build（其他场景按需）
- scope 使用模块/目录（如 app、data、scripts），无明确范围可省略
- summary 使用中文、动词开头，长度 ≤ 50 字，不加句号
- 需要时在正文补充动机、影响或迁移方式

### 提交类型

|类型|说明|
|---|---|
|`🎉 init`|项目初始化|
|`✨ feat`|新功能|
|`🐞 fix`|错误修复|
|`📃 docs`|文档变更|
|`🌈 style`|代码格式化（不影响代码逻辑）|
|`🦄 refactor`|代码重构（不新增功能或修复错误）|
|`🎈 perf`|性能优化|
|`🧪 test`|测试相关|
|`🔧 build`|构建系统或外部依赖|
|`🐎 ci`|CI配置相关|
|`🐳 chore`|构建过程或辅助工具的变动|
|`↩ revert`|撤销提交|

### 破坏性变更

- 在 type 后添加 `!`，或在正文写明 `BREAKING CHANGE: ...`
- 明确写出受影响范围与升级指引

###  提交流程

- `git status` 确认改动范围
- `git add <files>` 仅添加相关文件
- `npm run lint` 通过后再提交
- `git commit -m "..."` 完成提交
- `git push` 后发起 PR（如需）