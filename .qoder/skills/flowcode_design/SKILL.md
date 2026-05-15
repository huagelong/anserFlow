---
name: flowcode_design
description: 使用 Pencil MCP 在画布上进行 UI 设计、原型创建和设计系统管理。Use when the user asks to create UI designs, mockups, dashboards, landing pages, wireframes, design systems, or any visual design task using Pencil. Triggers on mentions of "设计"、"UI"、"原型"、"画布"、"dashboard"、"landing page"、"wireframe"、"设计系统"、"Pencil"。
---

# Pencil MCP 设计技能

通过 Pencil MCP 服务器在画布上创建和修改 UI 设计。所有操作通过 `CallMcpTool` 调用 Pencil MCP 的工具完成。

核心原则：**自动运行，自主找方案，实在不能运行就停止**。遇到问题时先自行分析并尝试解决，不等待用户介入；多个方案可选时，选择最优方案直接执行；只有确认无法继续时才停止任务并说明原因。

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

### 第二步：读取项目设计规范（必须）

在设计之前，必须先读取项目的设计规范文档，提取颜色、字体、间距、布局等 Token 值。这些文档是所有设计决策的唯一来源。

查找项目中包含以下内容的规范文档（文件名和路径因项目而异）：
- **视觉规范**：颜色体系、字体、字号阶梯、间距阶梯、圆角规范、组件样式
- **布局规范**：顶栏、侧边栏、内容区的尺寸、背景色、边框等布局参数
- **设计总览**：页面清单、体验原则、交互规范

读取后提取关键值，用于后续的 `set_variables` 和所有 `batch_design` 操作。

### 第三步：初始化设计变量（新文档必须）

根据上一步读取的 Token 值，调用 `set_variables` 创建设计变量。以下为示例格式，实际值必须来自项目设计文档：

```
server_name: "pencil"
tool_name: "set_variables"
arguments: {
  "filePath": "<path>",
  "variables": {
    "color-bg": "<从设计文档提取>",
    "color-surface": "<从设计文档提取>",
    "color-primary": "<从设计文档提取>",
    "color-text-primary": "<从设计文档提取>",
    "color-border": "<从设计文档提取>"
    // ... 根据文档补充更多变量
  }
}
```

已有文档可跳过此步，用 `get_variables` 确认变量已存在。

### 第四步：了解可用组件

如果是新文档或有设计系统，调用 `batch_get` 获取可用组件：

```
arguments: {
  "filePath": "<从 get_editor_state 获取>",
  "patterns": [{ "reusable": true }],
  "readDepth": 2,
  "searchDepth": 3
}
```

### 第五步：获取设计指南（可选）

调用 `get_guidelines` 获取设计指南或视觉风格：

```
// 列出所有可用指南和风格
arguments: {}

// 加载具体指南
arguments: { "category": "guide", "name": "<指南名>" }

// 加载视觉风格
arguments: { "category": "style", "name": "<风格名>", "params": { ... } }
```

### 第六步：创建全局布局并填充内容

使用 `batch_design` 创建和修改设计元素。

**强制要求**：所有项目内页面必须基于「全局布局模板」（见下方章节）创建，不能从零自由构建。先创建页面框架（顶栏 + 侧边栏 + 内容区），再在内容区内填充页面特定内容。所有色值、字号、间距必须使用从设计文档提取的 Token 值。

### 第七步：一致性校验

设计完成后必须执行一致性校验（见「一致性校验」章节）。

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
7. **padding 必须使用数组格式**：`[all]`、`[h, v]`、`[top, right, bottom, left]`，不接受对象格式

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

## 设计 Token 体系

所有设计操作的色值、字体、间距、圆角必须来自项目的设计规范文档，禁止自由编造值。以下展示一个完整的 Token 体系示例，供理解结构参考。实际值必须从项目的设计文档中读取。

### Token 体系结构示例

一个完整的设计 Token 体系通常包含以下分类，具体值从设计文档提取：

#### 表面与背景色

需要提取的 Token：页面背景色、一级表面色（白色卡片/表格）、二级表面色（工具栏）、深色表面（终端/日志面板）。

#### 文字色

需要提取的 Token：主文字色、次级文字色、弱文字色。

#### 品牌与功能色

需要提取的 Token：主色及浅底、成功色及浅底、警告色及浅底、危险色及浅底、辅助色、紫色识别色。

#### 边框与分割色

需要提取的 Token：默认边框色、强调边框色。

#### 状态色映射

如果项目有状态机（如 Issue 状态），需要从设计文档提取每种状态对应的文字色和背景色。

#### 字体

需要提取：UI 文本字体、代码字体。

#### 字号阶梯

需要提取的 Token：从 xs（12px）到 3xl（32px）的完整字号阶梯，每个包含大小、字重、行高。

#### 间距阶梯

需要提取的 Token：基于基础单位（如 4px）的间距阶梯，从 space-1 到 space-8。

#### 圆角

需要提取的 Token：小组件圆角、卡片圆角、全圆角。

### 设计一致性原则

1. **Token 唯一来源**：所有 fill、stroke、fontColor 必须来自设计文档中定义的 Token，禁止自由编造色值
2. **字号阶梯化**：使用设计文档定义的字号阶梯，不要随意设定字号
3. **间距统一倍数**：使用设计文档定义的间距阶梯（如 4px 基础单位的倍数）
4. **圆角克制**：按照设计文档的圆角规范使用，不要随意设定
5. **默认不用阴影**：依靠背景色和边框分层，仅弹窗和拖拽卡片使用阴影

## 常用设计模式

### 创建新页面/画板

```
page=I("document",{type:"frame",name:"Landing Page",width:1440,height:900,layout:"vertical",fill:"<页面背景色>"})
```

### 创建带布局的区域

```
header=I(page,{type:"frame",name:"Header",width:"fill_container",height:64,layout:"horizontal",padding:[0,24],alignItems:"center",justifyContent:"space_between"})
```

### 插入文本

```
title=I(header,{type:"text",content:"Welcome",fontSize:24,fontWeight:"bold",fontFamily:"<UI字体>"})
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

## 全局布局模板

所有项目内页面必须使用从设计文档提取的统一布局框架。以下为典型布局结构示例，具体尺寸和色值从项目的布局文档中读取。

### 布局结构示例

```
┌──────────────────────────────────────────────────────────────┐
│ Topbar       bg=一级表面色  borderBottom=边框色             │
├──────────────┬───────────────────────────────────────────────┤
│ Sidebar      │ Content   bg=页面背景色  padding=按文档规范  │
│ bg=一级表面色│                                               │
│ borderRight  │  面包屑 > 页面标题栏 > 工具栏 > 主内容        │
│ =边框色     │                                               │
└──────────────┴───────────────────────────────────────────────┘
```

### 标准页面模板代码示例

以下为代码结构示例，其中 `<Token>` 占位符表示从设计文档提取的实际值：

```
page=I("document",{type:"frame",name:"<页面名>",width:1440,height:900,layout:"vertical",fill:"<页面背景色>"})

// 顶栏
opbar=I(page,{type:"frame",name:"Topbar",width:"fill_container",height:<顶栏高度>,layout:"horizontal",fill:"<一级表面色>",padding:[0,<水平间距>],alignItems:"center",justifyContent:"space_between",stroke:"<边框色>",strokeWidth:[0,0,1,0]})
topbarLeft=I(topbar,{type:"frame",name:"TopbarLeft",layout:"horizontal",gap:16,alignItems:"center"})
logo=I(topbarLeft,{type:"text",content:"<产品名>",fontSize:<产品名字号>,fontWeight:700,fontFamily:"<UI字体>",fill:"<主文字色>"})
topbarRight=I(topbar,{type:"frame",name:"TopbarRight",layout:"horizontal",gap:12,alignItems:"center"})
userAvatar=I(topbarRight,{type:"frame",name:"UserAvatar",width:32,height:32,fill:"<主色>",cornerRadius:999})

// 主体区域
body=I(page,{type:"frame",name:"Body",width:"fill_container",height:"fill_container",layout:"horizontal"})

// 侧边栏
sidebar=I(body,{type:"frame",name:"Sidebar",width:<侧边栏宽度>,height:"fill_container",layout:"vertical",fill:"<一级表面色>",padding:[12,0],stroke:"<边框色>",strokeWidth:[0,1,0,0]})

// 内容区
content=I(body,{type:"frame",name:"Content",width:"fill_container",height:"fill_container",layout:"vertical",fill:"<页面背景色>",padding:[<内容区垂直padding>,<内容区水平padding>],gap:20})
```

### 登录页模板示例

登录页不使用顶栏+侧边栏布局，使用居中面板：

```
page=I("document",{type:"frame",name:"Login",width:1440,height:900,fill:"<页面背景色>",layout:"horizontal",justifyContent:"center",alignItems:"center"})
panel=I(page,{type:"frame",name:"LoginPanel",width:<登录面板宽度>,fill:"<一级表面色>",cornerRadius:<卡片圆角>,padding:32,layout:"vertical",gap:24,stroke:"<边框色>",strokeWidth:1})
productName=I(panel,{type:"text",content:"<产品名>",fontSize:<产品名字号>,fontWeight:700,fontFamily:"<UI字体>",fill:"<主文字色>"})
```

### 布局一致性要求

1. **所有项目内页面**必须使用标准布局框架（顶栏 + 侧边栏 + 内容区）
2. 顶栏、侧边栏、内容区的尺寸和色值必须与设计文档一致
3. 只有特殊页面（如登录页）可使用独立模板
4. 内容区内按顺序：面包屑 -> 页面标题栏 -> 工具栏（可选）-> 主内容

## 公共组件模板

以下为常用组件的代码结构示例，所有色值、字号、间距使用从设计文档提取的 Token 值。

### 侧边栏导航项

普通项：
```
navItem=I(sidebar,{type:"frame",name:"NavItem-<名称>",width:"fill_container",height:40,layout:"horizontal",padding:[0,16],gap:8,alignItems:"center",cornerRadius:<小组件圆角>})
navIcon=I(navItem,{type:"text",content:"<图标>",fontSize:16,fontFamily:"<UI字体>",fill:"<次级文字色>"})
navLabel=I(navItem,{type:"text",content:"<导航名>",fontSize:<正文号>,fontFamily:"<UI字体>",fill:"<次级文字色>"})
```

选中项（当前路由）：
```
navItemActive=I(sidebar,{type:"frame",name:"NavItem-<名称>-Active",width:"fill_container",height:40,layout:"horizontal",padding:[0,16],gap:8,alignItems:"center",fill:"<主色浅底>",cornerRadius:<小组件圆角>,stroke:"<主色>",strokeWidth:[0,0,0,3]})
navIconActive=I(navItemActive,{type:"text",content:"<图标>",fontSize:16,fontFamily:"<UI字体>",fill:"<主色>"})
navLabelActive=I(navItemActive,{type:"text",content:"<导航名>",fontSize:<正文号>,fontFamily:"<UI字体>",fill:"<主色>",fontWeight:500})
```

导航分组标题：
```
navGroup=I(sidebar,{type:"text",content:"<分组名>",fontSize:<辅助号>,fontFamily:"<UI字体>",fill:"<弱文字色>",fontWeight:500,padding:[12,16,4,16]})
```

### 页面标题栏

```
titleBar=I(content,{type:"frame",name:"TitleBar",width:"fill_container",layout:"horizontal",justifyContent:"space_between",alignItems:"center"})
titleLeft=I(titleBar,{type:"frame",name:"TitleLeft",layout:"vertical",gap:4})
titleText=I(titleLeft,{type:"text",content:"<页面标题>",fontSize:<页面标题号>,fontWeight:600,fontFamily:"<UI字体>",fill:"<主文字色>"})
titleSubtitle=I(titleLeft,{type:"text",content:"<副标题>",fontSize:<辅助号>,fontFamily:"<UI字体>",fill:"<次级文字色>"})
titleActions=I(titleBar,{type:"frame",name:"TitleActions",layout:"horizontal",gap:8,alignItems:"center"})
primaryBtn=I(titleActions,{type:"frame",name:"PrimaryBtn",height:32,padding:[0,16],fill:"<主色>",cornerRadius:<小组件圆角>,layout:"horizontal",alignItems:"center",justifyContent:"center"})
primaryBtnText=I(primaryBtn,{type:"text",content:"<主操作>",fontSize:<正文号>,fontWeight:500,fontFamily:"<UI字体>",fill:"<一级表面色>"})
```

### 工具栏

```
toolbar=I(content,{type:"frame",name:"Toolbar",width:"fill_container",height:<工具栏高度>,layout:"horizontal",fill:"<一级表面色>",padding:[0,16],alignItems:"center",gap:12,cornerRadius:<卡片圆角>,stroke:"<边框色>",strokeWidth:1})
searchInput=I(toolbar,{type:"frame",name:"SearchInput",width:240,height:32,fill:"<二级表面色>",cornerRadius:<小组件圆角>,padding:[0,12],layout:"horizontal",alignItems:"center"})
searchPlaceholder=I(searchInput,{type:"text",content:"搜索...",fontSize:<辅助号>,fontFamily:"<UI字体>",fill:"<弱文字色>"})
filterDropdown=I(toolbar,{type:"frame",name:"Filter",height:32,padding:[0,12],fill:"<一级表面色>",cornerRadius:<小组件圆角>,stroke:"<边框色>",strokeWidth:1,layout:"horizontal",alignItems:"center",gap:4})
filterText=I(filterDropdown,{type:"text",content:"<筛选>",fontSize:<辅助号>,fontFamily:"<UI字体>",fill:"<次级文字色>"})
```

### 状态徽章

高度 22px，圆角 999px，字号 12px，字重 500。每种状态使用设计文档中对应状态色的文字色和背景色：

```
badge=I(parent,{type:"frame",name:"Badge-<状态>",height:22,padding:[0,8],cornerRadius:999,layout:"horizontal",alignItems:"center",justifyContent:"center",fill:"<状态背景色>"})
badgeText=I(badge,{type:"text",content:"<状态名>",fontSize:12,fontWeight:500,fontFamily:"<UI字体>",fill:"<状态文字色>"})
```

### 卡片

```
card=I(parent,{type:"frame",name:"Card",width:"fill_container",padding:<卡片padding>,fill:"<一级表面色>",cornerRadius:<卡片圆角>,stroke:"<边框色>",strokeWidth:1,layout:"vertical",gap:12})
cardTitle=I(card,{type:"text",content:"<卡片标题>",fontSize:<小标题号>,fontWeight:600,fontFamily:"<UI字体>",fill:"<主文字色>"})
cardDesc=I(card,{type:"text",content:"<卡片描述>",fontSize:<辅助号>,fontFamily:"<UI字体>",fill:"<次级文字色>"})
```

选中态卡片：`fill:"<主色浅底>", stroke:"<主色>", strokeWidth:1`

### 表格行

```
tableRow=I(tableBody,{type:"frame",name:"Row",width:"fill_container",height:<表格行高>,layout:"horizontal",padding:[0,16],alignItems:"center",fill:"<一级表面色>",stroke:"<边框色>",strokeWidth:[0,0,1,0]})
```

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
    "fillColor": [{ "from": "<非Token色值>", "to": "<Token色值>" }]
  }
}
```

### 搜索属性

用 `search_all_unique_properties` 查看文档中使用的所有属性值。

## 一致性校验

每个页面设计完成后，必须执行以下校验步骤，确保与设计 Token 一致。

### 校验步骤

1. **搜索属性偏差**：调用 `search_all_unique_properties` 检查文档中使用的色值、字体等是否偏离 Token：

```
arguments: { "filePath": "<path>", "parents": ["<页面节点ID>"] }
```

2. **批量修正偏差**：发现非 Token 色值时，调用 `replace_all_matching_properties` 统一替换：

```
arguments: {
  "filePath": "<path>",
  "parents": ["<页面节点ID>"],
  "properties": {
    "fill": [{ "from": "<非Token色值>", "to": "<Token色值>" }]
  }
}
```

3. **布局问题检查**：调用 `snapshot_layout` 检查布局问题：

```
arguments: { "filePath": "<path>", "problemsOnly": true }
```

4. **视觉截图验证**：调用 `get_screenshot` 确认视觉效果：

```
arguments: { "filePath": "<path>", "nodeId": "<页面节点ID>" }
```

### 校验清单

每个页面完成后，对照从设计文档提取的 Token 值逐一核对：

- [ ] 页面背景色是否与设计文档的页面背景 Token 一致
- [ ] 顶栏背景色是否与一级表面 Token 一致，高度是否符合布局文档
- [ ] 侧边栏宽度是否与布局文档一致，背景色是否与一级表面 Token 一致
- [ ] 内容区 padding 是否与布局文档一致
- [ ] 所有文字字体是否统一（UI 文本用 UI 字体，代码区域用代码字体）
- [ ] 状态徽章色值是否与设计文档的状态色映射一致
- [ ] 边框色是否统一
- [ ] 主色按钮色值是否与主色 Token 一致
- [ ] 圆角是否在文档规定的范围内

## 多页面设计规范

同一项目设计多个页面时，必须遵循以下规范确保跨页面一致性。

### 第一个页面的职责

第一个页面需要建立全局基础设施：

1. 调用 `set_variables` 创建所有设计 Token 变量
2. 使用「标准页面模板」创建完整的顶栏 + 侧边栏 + 内容区
3. 在侧边栏中创建完整的导航项
4. 在内容区中填充页面特定内容
5. 完成后执行一致性校验

### 后续页面的创建

后续页面有两种方式创建：

**方式一：复制布局框架**（推荐）

使用 `Copy` 操作复制第一个页面的框架，再清空内容区并填充新内容：

```
// 1. 调用 find_empty_space_on_canvas 获取空位坐标
// 2. 复制整个页面
newPage=C("<第一页ID>","document",{name:"<新页面名>",positionDirection:"right",positionPadding:48})
// 3. 删除旧内容区的内容，保留结构
// 4. 在内容区中填充新页面内容
```

**方式二：基于模板重新创建**

使用「标准页面模板」重新创建，确保所有参数与第一页一致。适用于复制可能导致布局问题的场景。

### 跨页面一致性要求

1. **顶栏完全相同**：logo、组织切换器、项目切换器、通知、用户菜单在所有页面中保持一致
2. **侧边栏完全相同**：导航项、分组、选中态在所有页面中保持一致（仅选中项不同）
3. **内容区结构一致**：都按 面包屑 -> 标题栏 -> 工具栏 -> 主内容 的顺序
4. **每个页面完成后都执行一致性校验**
5. **发现偏差立即修正**：不要等所有页面完成后才统一修正

## 设计最佳实践

1. **先获取状态**：每次任务开始调用 `get_editor_state`
2. **读取设计文档**：必须先读取项目设计规范，提取 Token 值
3. **初始化设计变量**：新文档根据设计文档的值调用 `set_variables` 创建 Token 变量
4. **了解组件库**：先 `batch_get` 查看可用组件，避免从零构建
5. **基于模板创建**：所有页面必须使用「全局布局模板」，不能自由发挥
6. **分批操作**：大设计拆分为多个 `batch_design` 调用（布局 -> 公共组件 -> 页面内容 -> 细节）
7. **用布局而非坐标**：优先使用 `layout: "horizontal"/"vertical"` + `gap`/`padding`，避免手动 `x`/`y` 定位
8. **Token 优先**：所有色值、字号、间距必须来自项目设计文档
9. **截图验证**：完成一个完整区域后再截图确认，不要每步都截
10. **一致性校验**：每个页面完成后执行校验清单
11. **引用设计系统**：尽量使用已有的 ref 组件，保持一致性
12. **复制优于重建**：多页面设计时优先复制已有页面框架，再修改内容

## 完整示例：创建 Dashboard 页面

以下为完整工作流示例，所有 `<Token>` 占位符表示从项目设计文档提取的实际值：

```
// 1. 获取编辑器状态
get_editor_state({ include_schema: true })

// 2. 读取项目设计规范文档
// 根据项目实际情况查找并读取视觉规范、布局规范等设计文档
// 提取颜色、字体、间距、布局尺寸等 Token 值

// 3. 初始化设计变量（新文档时执行）
set_variables({
  filePath: "<path>",
  variables: {
    "color-bg": "<从设计文档提取>",
    "color-surface": "<从设计文档提取>",
    "color-primary": "<从设计文档提取>",
    "color-text-primary": "<从设计文档提取>",
    "color-border": "<从设计文档提取>"
  }
})

// 4. 获取可用组件
batch_get({ filePath: "<path>", patterns: [{ reusable: true }], readDepth: 2, searchDepth: 3 })

// 5. 创建全局布局框架（所有值从设计文档提取）
batch_design({
  filePath: "<path>",
  input: `page=I("document",{type:"frame",name:"Dashboard",width:1440,height:900,layout:"vertical",fill:"<页面背景色>"})
topbar=I(page,{type:"frame",name:"Topbar",width:"fill_container",height:<顶栏高度>,layout:"horizontal",fill:"<一级表面色>",padding:[0,<水平间距>],alignItems:"center",justifyContent:"space_between",stroke:"<边框色>",strokeWidth:[0,0,1,0]})
topbarLeft=I(topbar,{type:"frame",name:"TopbarLeft",layout:"horizontal",gap:16,alignItems:"center"})
logo=I(topbarLeft,{type:"text",content:"<产品名>",fontSize:<产品名字号>,fontWeight:700,fontFamily:"<UI字体>",fill:"<主文字色>"})
topbarRight=I(topbar,{type:"frame",name:"TopbarRight",layout:"horizontal",gap:12,alignItems:"center"})
userAvatar=I(topbarRight,{type:"frame",name:"UserAvatar",width:32,height:32,fill:"<主色>",cornerRadius:999})
body=I(page,{type:"frame",name:"Body",width:"fill_container",height:"fill_container",layout:"horizontal"})
sidebar=I(body,{type:"frame",name:"Sidebar",width:<侧边栏宽度>,height:"fill_container",layout:"vertical",fill:"<一级表面色>",padding:[12,0],stroke:"<边框色>",strokeWidth:[0,1,0,0]})
content=I(body,{type:"frame",name:"Content",width:"fill_container",height:"fill_container",layout:"vertical",fill:"<页面背景色>",padding:[<垂直间距>,<水平间距>],gap:20})`
})

// 6. 填充侧边栏导航 + 内容区
// 先添加侧边栏导航项，再在 content 中添加页面标题栏、工具栏、卡片等内容
// ... 分多个 batch_design 逐步添加

// 7. 一致性校验
search_all_unique_properties({ filePath: "<path>", parents: ["<pageNodeId>"] })
snapshot_layout({ filePath: "<path>", problemsOnly: true })
get_screenshot({ filePath: "<path>", nodeId: "<pageNodeId>" })
```
