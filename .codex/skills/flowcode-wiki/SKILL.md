---
name: flowcode_wiki
description: Build a wiki-style knowledge base from a project's existing Markdown documentation files. Use when Codex is asked to analyze documentation, split long docs into smaller Markdown pages, create a central wiki index, add retrieval paths, relationship graphs, and cross-links, while using source docs as the only content source and avoiding unrelated file changes. Source and output directories are configurable, not hardcoded.
---

# Docs Wiki Builder

## 使用目标

把项目已有 Markdown 文档整理成可检索的 wiki 知识库，方便 Codex 或其他大模型按总目录逐级检索。

必须遵守：

- 只从源 Markdown 文档目录获取内容，不修改源文档原文。
- 不改动代码、配置和无关目录。
- 不补写源文档中没有的设计结论。
- 不创建临时文件或额外总结文档。
- 所有生成的链接必须指向真实存在的目标文件；写入后要校验。

## 路径策略

不要把目录写死。把路径视为可配置参数：

```text
sourceDocsDir   源 Markdown 文档目录
wikiDir         wiki 输出目录
wikiIndex       wiki 总目录文件（默认 wikiDir 下的 README.md）
```

推断顺序：

1. 优先使用用户明确给出的路径。
2. 如果用户只说"生成 wiki"或"整理文档"，优先查找当前项目中明显的 Markdown 文档目录（如 `docs/`、`doc/`、`documentation/`）；如果存在多个候选且无法判断，先询问用户。
3. 如果用户指定了输出位置，`wikiDir` 取该位置；否则默认使用项目根目录下的 `wiki/`。
4. `wikiIndex` 默认为 `wikiDir/README.md`，除非用户指定了其他索引文件名。
5. 如果本 Skill 的副本也在 `wikiDir` 下，生成业务 wiki 时不要覆盖或移动该 Skill 目录。

在汇报中说明实际采用的路径。

## 输出结构

在 `wikiDir` 下生成或更新：

```text
<wikiDir>/
  README.md
  <源文档名>/
    README.md
    <章节>.md
    <章节>/
      README.md
      <子章节>.md
```

## 拆分规则

按源文档文件名顺序处理 `sourceDocsDir` 下的 Markdown 文件。

解析标题时必须忽略代码块中的 `#`、`##`、`###`。推荐粒度：

- 每个源文档的一级标题成为 `wikiDir/<源文档名>/README.md` 的主题入口。
- 二级标题没有三级标题时，二级章节拆成一个小文档。
- 二级标题下面有三级标题时，二级章节生成一个目录入口，三级标题各自拆成小文档。
- 不要把表格、代码块、流程图、连续枚举步骤拆断。
- 章节正文尽量原样迁移，只允许添加导航元信息。

## 总目录要求

`wikiIndex`（即 `wikiDir/README.md`）必须包含：

- 快速入口：所有源文档主题入口。
- 推荐检索路径：面向常见问题的阅读路线（从实际文档主题和编号中推断，不要创造不存在的主题）。
- 关系图谱：Mermaid 图或关系表。
- 细分文档树：所有小文档链接。

推荐检索路径应基于实际文档内容和语义关系构建，以下仅为示例格式：

```text
示例链路1：项目概述 -> 系统架构 -> 数据模型 -> API 设计
示例链路2：核心引擎 -> 外部集成 -> 验证与部署
```

不要套用不相关的链路，也不要编造源文档中不存在的主题。

## 关系规则

优先使用源文档的实际编号和主题建立关联。关系必须从实际文档标题、编号、链接和语义中推断。

如果源文档有编号，按编号顺序建立相邻关系；如果没有编号，按标题语义推断。不要创造源文档中不存在的主题或关联。

## 小文档导航

每个小文档开头添加简短导航块：

```md
> 来源：`<sourceDocsDir>/<源文件>` 第 N 行
> 位置：[总目录](../README.md) -> [源文档](README.md) -> 当前章节
> 相邻：[上一篇](...) · [下一篇](...)
> 相关主题：[...](...) · [...]
```

如果用户要求完全保留原文，不要在小文档内加入导航块，把所有关系集中放到 `wikiIndex`。

## 校验

完成后检查：

- 源文档目录没有被修改。
- `git diff --stat` 只出现 `wikiDir` 下的文件，或只出现用户明确允许的位置。
- `wikiIndex` 能到达所有主题入口和小文档。
- 小文档相对链接能指向已生成文件。
- 代码块没有被标题解析误拆。

## 汇报格式

完成后简短说明：

- 使用的源文档目录和 wiki 输出目录。
- 创建或更新了哪些文件。
- 拆分粒度和校验结果。
