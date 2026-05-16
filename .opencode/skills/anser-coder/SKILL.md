---
name: anser-coder
description: AnserFlow 沙箱编码执行规范。opencode 在 Docker 沙箱中执行 Issue 编码任务时必须遵守的代码风格、提交规范、PR 格式和质量标准。
is_builtin: true
---

# AnserFlow 沙箱编码执行规范

## 目标

作为 AnserFlow 默认内置 Skill，定义 opencode 在 Docker 沙箱中执行 Issue 编码任务时的行为规范。确保产出代码风格一致、提交历史清晰、PR 格式标准。

## 编码原则

### 最小变更

- 只修改完成当前 Issue 必需的文件，不触碰无关代码。
- 不重构、不格式化、不"顺便改进"Issue 范围外的代码。
- 不新增未请求的抽象、配置项或扩展点。

### 沿用现有风格

- 优先沿用项目已有的架构模式、命名约定、错误处理和测试风格。
- 如果项目有 linter/formatter 配置（如 `.golangci.yml`、`.eslintrc`、`prettier.config`），严格遵守。
- 如果项目有现有测试，新增代码必须补充对应测试。

### 防御性编码

- 所有外部输入必须校验。
- 错误信息应具体、可操作，不暴露内部实现细节。
- 资源泄漏必须处理（关闭文件、释放连接等）。

## Git 提交规范

### Commit Message

格式：

```text
<type>(<scope>): <description>

[可选 body：说明原因和影响]
```

type 枚举：

| type | 含义 |
|------|------|
| feat | 新功能 |
| fix | 修复 bug |
| refactor | 重构（不改行为） |
| test | 补充测试 |
| docs | 文档变更 |
| chore | 构建/工具/依赖 |

### 提交粒度

- 按**逻辑单元**提交，不按文件数量提交。
- 一个提交只做一件事；如果 Issue 涉及多个独立改动，拆分为多个提交。
- 不提交调试代码、注释掉的代码、`console.log`/`fmt.Println` 等临时代码。

## PR 格式

### 标题

```text
[#<issue_id>] <简短描述>
```

示例：`[#12] 实现 Issue 状态变更群聊通知`

### 描述模板

```markdown
## 关联 Issue

Closes #<issue_id>

## 变更内容

- <变更点 1>
- <变更点 2>

## 验证方式

- [ ] <验证步骤 1>
- [ ] <验证步骤 2>

## 截图/日志（如适用）

<粘贴关键截图或终端输出>
```

### PR 要求

- PR 描述必须关联 Issue 编号（`Closes #xx`）。
- 列出所有变更点和验证步骤。
- 不含合并冲突、不包含无关文件变更。
- 如果有数据库迁移，在描述中单独说明。

## 代码风格

### Go（后端）

- 遵循 [Effective Go](https://go.dev/doc/effective_go) 和项目 `.golangci.yml` 配置。
- 包注释、导出函数注释、错误变量命名 `ErrXxx`。
- 错误处理：不吞错误，使用 `fmt.Errorf("context: %w", err)` 包装。

### TypeScript/React（前端）

- 遵循项目 ESLint + Prettier 配置。
- 组件使用函数式组件 + Hooks。
- 类型优先使用 `interface`，复杂联合类型使用 `type`。
- 不使用 `any`，如确需使用须加注释说明原因。

## 质量门禁

编码完成后必须自检：

1. **编译通过**：Go `go build` / 前端 `next build` 无错误。
2. **Lint 通过**：Go `golangci-lint run` / 前端 `eslint` 无新增告警。
3. **测试通过**：运行受影响的测试套件，不引入新的失败用例。
4. **无遗留调试代码**：grep 确认无 `fmt.Println`、`console.log`、`debugger` 等临时代码。

以上任一不通过，不提交 PR，在 Issue 时间线中记录失败原因。

## 执行边界

- 不尝试修改项目外的文件或系统配置。
- 不安装未在项目依赖中声明的新包（除非 Issue 明确要求）。
- 不删除或重命名现有文件（除非 Issue 明确要求）。
- 不修改数据库迁移文件（由平台管理，不由沙箱产生）。
