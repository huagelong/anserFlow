---
name: flowcode_design
description: 使用 Pencil MCP 在画布上进行 UI 设计、原型创建和设计系统管理。Use when the user asks to create UI designs, mockups, dashboards, landing pages, wireframes, design systems, or any visual design task using Pencil. Triggers on mentions of "设计"、"UI"、"原型"、"画布"、"dashboard"、"landing page"、"wireframe"、"设计系统"、"Pencil"。
---

# Pencil MCP 设计技能

通过 Pencil MCP 服务器在画布上创建和修改 UI 设计。所有操作通过 `CallMcpTool` 调用 Pencil MCP 的工具完成。

## 工具调用方式

所有 Pencil MCP 工具通过 `CallMcpTool` 调用，`server_name` 固定为 `"pencil"`，`tool_name` 为具体工具名。

## 核心工作流

### 第一步：获取编辑器状态（必须）

每次设计任务开始前，调用 `get_editor_state` 获取当前画布上下文：

```
server_name: "pencil"
tool_name: "get_editor_state"
arguments: { "include_schema": true }
```

首次调用必须 `include_schema: true`，获取 .pen 文件 schema。后续调用可设为 `false`。

返回信息包含：
- 当前活跃的 .pen 文件路径（`filePath`）
- 用户选中的节点
- 可用的设计系统组件

**重要**：后续所有工具调用都需要传入 `filePath` 参数。

### 第二步：了解可用组件

如果是新文档或有设计系统，调用 `batch_get` 获取可用组件：

```
arguments: {
  "filePath": "<从 get_editor_state 获取>",
  "patterns": [{ "reusable": true }],
  "readDepth": 2,
  "searchDepth": 3
}
```

### 第三步：获取设计指南（可选）

调用 `get_guidelines` 获取设计指南或视觉风格：

```
// 列出所有可用指南和风格
arguments: {}

// 加载具体指南
arguments: { "category": "guide", "name": "<指南名>" }

// 加载视觉风格
arguments: { "category": "style", "name": "<风格名>", "params": { ... } }
```

### 第四步：执行设计

使用 `batch_design` 创建和修改设计元素。

## batch_design 操作语法

`batch_design` 是核心设计工具，`input` 参数是一个 JavaScript 风格的操作字符串。

### 支持的操作

| 操作 | 语法 | 说明 |
|------|------|------|
| Insert | `id=I(parent, nodeData)` | 插入新节点 |
| Copy | `id=C(path, parent, copyData)` | 复制节点 |
| Update | `U(path, updateData)` | 更新节点属性 |
| Replace | `id=R(path, nodeData)` | 替换节点 |
| Move | `M(nodeId, parent, index?)` | 移动节点 |
| Delete | `D(nodeId)` | 删除节点 |
| Get Image | `G(nodeId, type, prompt)` | 生成/获取图片 |

### 关键规则

1. **I/C/R 操作必须有绑定名**：`sidebar=I("parentId",{...})`
2. **不要手写 id**：节点 id 由系统自动生成
3. **操作按顺序执行**：失败时整个批次回滚
4. **大型设计分批处理**：先创建结构，再填充内容
5. **`document` 是预定义绑定**：指向文档根节点，仅用于创建顶级 screen/frame
6. **图片没有独立节点类型**：先用 I 创建 frame/rectangle，再用 G 应用图片填充

### 节点类型与属性兼容性

- `children`、`layout`、`gap`、`padding`、`justifyContent`、`alignItems`：仅 `frame` 和 `group`
- `fill`、`stroke`、`cornerRadius`：不能用于 `group`
- `cornerRadius`：仅 `frame`、`rectangle`、`polygon`
- `content`、`fontSize`、`fontFamily` 等：仅 `text`
- `iconFontName`、`iconFontFamily`：仅 `icon_font`

### 组件实例（ref）

使用组件时插入 ref 实例，通过 `descendants` 或后续 U/R 操作自定义：

```
card=I("body",{type:"ref",ref:"CardComponentId"})
U(card+"/titleText",{content:"New Title"})
```

路径格式：`instanceId/childId`，支持多级嵌套。

## 常用设计模式

### 创建新页面/画板

```
page=I("document",{type:"frame",name:"Landing Page",width:1440,height:900,layout:"vertical",fill:"#FFFFFF"})
```

### 创建带布局的区域

```
header=I(page,{type:"frame",name:"Header",width:"fill_container",height:64,layout:"horizontal",padding:{left:24,right:24},alignItems:"center",justifyContent:"space_between"})
```

### 插入文本

```
title=I(header,{type:"text",content:"Welcome",fontSize:24,fontWeight:"bold",fontFamily:"Inter"})
```

### 使用设计系统组件

```
btn=I(section,{type:"ref",ref:"<组件ID>",width:200,height:48})
U(btn+"/labelNode",{content:"Click Me"})
```

### 复制并修改

```
card2=C("card1Id",container,{name:"Card 2",positionDirection:"right",positionPadding:24,descendants:{"titleId":{content:"New Title"}}})
```

### 应用图片

```
heroImg=I("parentId",{type:"frame",name:"Hero Image",width:400,height:300})
G(heroImg,"ai","modern office workspace bright natural light")
```

### 查找空位放置新画板

先调用 `find_empty_space_on_canvas`，获取坐标后插入。

## 辅助工具

### 验证设计

设计完成后用 `snapshot_layout` 检查布局问题：

```
arguments: { "filePath": "<path>", "problemsOnly": true }
```

### 截图验证

用 `get_screenshot` 查看视觉效果（节制使用）：

```
arguments: { "filePath": "<path>", "nodeId": "<要截图的节点ID>" }
```

### 读取设计结构

用 `batch_get` 读取现有节点结构和子元素。

### 导出设计

用 `export_nodes` 导出为图片：

```
arguments: {
  "filePath": "<path>",
  "nodeIds": ["<节点ID>"],
  "outputDir": "<输出目录>",
  "format": "png",
  "scale": 2
}
```

### 变量与主题

- `get_variables`：读取设计变量（颜色、间距等）
- `set_variables`：设置设计变量，支持明暗主题

### 批量替换属性

用 `replace_all_matching_properties` 全局替换样式属性（如统一修改颜色）：

```
arguments: {
  "filePath": "<path>",
  "parents": ["<父节点ID>"],
  "properties": {
    "fillColor": [{ "from": "#3B82F6", "to": "#2563EB" }]
  }
}
```

### 搜索属性

用 `search_all_unique_properties` 查看文档中使用的所有属性值。

## 设计最佳实践

1. **先获取状态**：每次任务开始调用 `get_editor_state`
2. **了解组件库**：先 `batch_get` 查看可用组件，避免从零构建
3. **分批操作**：大设计拆分为多个 `batch_design` 调用（结构 -> 内容 -> 细节）
4. **先放后调**：先创建整体结构，再逐步调整样式
5. **用布局而非坐标**：优先使用 `layout: "horizontal"/"vertical"` + `gap`/`padding`，避免手动 `x`/`y` 定位
6. **截图验证**：完成一个完整区域后再截图确认，不要每步都截
7. **引用设计系统**：尽量使用已有的 ref 组件，保持一致性

## 完整示例：创建简单 Dashboard

```
// 1. 获取编辑器状态
get_editor_state({ include_schema: true })

// 2. 获取可用组件
batch_get({ filePath: "<path>", patterns: [{ reusable: true }], readDepth: 2, searchDepth: 3 })

// 3. 创建页面结构
batch_design({
  filePath: "<path>",
  input: `page=I("document",{type:"frame",name:"Dashboard",width:1440,height:900,layout:"horizontal",fill:"#F8FAFC"})
sidebar=I(page,{type:"frame",name:"Sidebar",width:240,height:"fill_container",layout:"vertical",padding:16,gap:8,fill:"#1E293B"})
main=I(page,{type:"frame",name:"Main",width:"fill_container",height:"fill_container",layout:"vertical",padding:32,gap:24})`
})

// 4. 填充内容（下一个 batch_design）
// ... 逐步添加 sidebar 菜单项、main 区域内容

// 5. 验证
snapshot_layout({ filePath: "<path>", problemsOnly: true })
get_screenshot({ filePath: "<path>", nodeId: "<pageNodeId>" })
```
