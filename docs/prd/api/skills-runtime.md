# Skills 与运行时

## 八、Skills 技能系统

### 8.1 两种导入方式

```
Skills 数据表：

┌──────────────────────────────────────────────┐
│ skills                                        │
├──────────────────────────────────────────────┤
│ source_type:  'manual' | 'zip'                │
│                                             │
│ 手动模式 (source_type='manual'):              │
│   definition: TEXT  ← 直接在 UI 编辑 Markdown │
│                                             │
│ ZIP 模式 (source_type='zip'):                 │
│   zip_hash: VARCHAR(64)   ← ZIP 包 SHA256    │
│   file_tree: JSON          ← 文件树快照       │
│   definition: TEXT         ← 解压后的 SKILL.md│
└──────────────────────────────────────────────┘
```

### 8.2 ZIP 包格式

```
my-skill.zip
├── SKILL.md              # 必须：Skill 定义
│                         #   ---
│                         #   name: my-skill
│                         #   description: ...
│                         #   ---
│                         #   # Skill 正文
├── agents/               # 可选：Agent 配置
│   └── openai.yaml
├── tools/                # 可选：辅助脚本
│   └── lint.sh
└── examples/             # 可选：示例
    └── sample.md
```

### 8.4 ZIP 导入完整流程

```
┌─────────────────────────────────────────────────────────────────┐
│                         Web 端 (Next.js)                        │
├─────────────────────────────────────────────────────────────────┤
│  <input type="file" accept=".zip" onChange={handleImport} />    │
│       │                                                         │
│       ▼                                                         │
│  const formData = new FormData()                                │
│  formData.append('file', file)       // 文件对象                │
│  formData.append('name', name)       // Skill 名称（可选）      │
│  formData.append('enabled', 'true')  // 导入后默认启用          │
│       │                                                         │
│       ▼                                                         │
│  POST /api/orgs/:org_id/skills/import/zip                       │
│  Content-Type: multipart/form-data                              │
│  Body: { file: Blob, name?: string, enabled?: boolean }         │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      后端 (Go + Gin)                             │
├─────────────────────────────────────────────────────────────────┤
│  // POST /api/orgs/:org_id/skills/import/zip                    │
│  func (h *SkillHandler) ImportZip(c *gin.Context) {             │
│      orgID := c.Param("org_id")                                 │
│                                                                │
│      // 1. 校验文件大小上限 10MB                                 │
│      c.Request.Body = http.MaxBytesReader(c.Writer,             │
│          c.Request.Body, 10<<20)                                │
│                                                                │
│      // 2. 解析 multipart form                                  │
│      file, header, err := c.Request.FormFile("file")            │
│      if err != nil {                                            │
│          respondError(c, "ERR_INVALID_UPLOAD", err)             │
│          return                                                 │
│      }                                                          │
│      defer file.Close()                                         │
│                                                                │
│      // 3. 读取到内存（不落盘）                                  │
│      zipBytes, _ := io.ReadAll(file)                            │
│      zipHash := sha256Hex(zipBytes) // 存入 zip_hash 字段       │
│                                                                │
│      // 4. 内存中解压 ZIP                                       │
│      zipReader, _ := zip.NewReader(                             │
│          bytes.NewReader(zipBytes), int64(len(zipBytes)))       │
│                                                                │
│      var definition string                                      │
│      var fileTree []FileNode                                    │
│                                                                │
│      for _, f := range zipReader.File {                         │
│          // 跳过目录、跳过系统文件（__MACOSX 等）                │
│          if f.FileInfo().IsDir() ||                              │
│             strings.HasPrefix(f.Name, "__MACOSX/") {            │
│              continue                                           │
│          }                                                      │
│                                                                │
│          rc, _ := f.Open()                                      │
│          content, _ := io.ReadAll(rc)                           │
│          rc.Close()                                             │
│                                                                │
│          // 记录到文件树                                         │
│          fileTree = append(fileTree, FileNode{                  │
│              Path:    f.Name,                                   │
│              Size:    int64(len(content)),                      │
│          })                                                     │
│                                                                │
│          // 提取 SKILL.md                                       │
│          if f.Name == "SKILL.md" {                              │
│              definition = string(content)                       │
│          }                                                      │
│      }                                                          │
│                                                                │
│      // 5. 校验：必须有 SKILL.md                                │
│      if definition == "" {                                      │
│          respondError(c, "ERR_MISSING_SKILL_MD", nil)           │
│          return                                                 │
│      }                                                          │
│                                                                │
│      // 6. 解析 SKILL.md 头部元信息                              │
│      meta := parseSkillFrontMatter(definition)                  │
│      // meta.Name, meta.Description                             │
│                                                                │
│      // 7. 写入数据库                                            │
│      skill := &Skill{                                           │
│          OrgID:       &orgID,                                   │
│          Name:        meta.Name,                                │
│          Description: meta.Description,                         │
│          SourceType:  "zip",                                    │
│          Definition:  definition,                               │
│          ZipHash:     zipHash,                                  │
│          FileTree:    marshalJSON(fileTree),                    │
│          Enabled:     true,                                     │
│      }                                                          │
│      h.skillRepo.Create(skill)                                  │
│                                                                │
│      // 8. 返回结果                                              │
│      c.JSON(200, gin.H{                                         │
│          "id":   skill.ID,                                      │
│          "name": skill.Name,                                    │
│          "file_count": len(fileTree),                           │
│      })                                                         │
│  }                                                              │
└─────────────────────────────────────────────────────────────────┘
```

> **约束**：ZIP 文件全程在内存中处理，最大 10MB。前端在 `onChange` 中可做客户端预检（文件类型 + 大小），后端 `MaxBytesReader` 兜底。重复导入同一 ZIP（相同 SHA256）不自动去重，由用户判断是否覆盖更新。

### 8.3 启用控制

```
                 ┌─────────────────┐
                 │  Skill A (全局)  │
                 │  enabled: true   │────────── 全局开关
                 └────────┬────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
    ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴─────┐
    │ Agent CEO │   │ Agent CTO │   │ Agent Dev │
    │ Skill A ✅ │   │ Skill A ✅ │   │ Skill A ❌ │── 单Agent开关
    └───────────┘   └───────────┘   └───────────┘
```

---
