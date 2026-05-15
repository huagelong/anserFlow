# 管理后台页面结构

## 十一、前端页面结构

### 11.1 管理后台

```
/admin
├── /login                    登录页
├── /dashboard                仪表盘
├── /organizations            组织管理
│   ├── /[id]/members         成员管理 + 邀请
│   └── /[id]/settings        组织设置
├── /agents                   Agent 管理
│   ├── /create               创建 Agent
│   ├── /[id]/edit            编辑（人设/运行时/Skills）
│   └── /[id]/logs            执行日志
├── /skills                   Skills 管理
│   ├── /create               手动创建
│   ├── /import               导入 ZIP
│   └── /[id]/edit            编辑
├── /projects                 项目管理
│   ├── /create               创建 + GitHub 关联(HTTP/SSH)
│   └── /[id]/issues          Issue 状态Tab列表
├── /groups                   群组
│   └── /[id]/chat            群聊界面
└── /settings                 全局系统设置
    ├── #general               基本信息（JWT过期/邀请默认值）
    ├── #eino                  Eino 调度引擎
    ├── #auth                  OAuth / CORS
    ├── #smtp                  邮件服务
    ├── #sandbox               Docker 沙箱
    ├── #queue                 Asynq 任务队列
    ├── #upgrade               自动更新
    └── #runtimes              运行时管理（注册/编辑/启停 + 默认Skills绑定）
```

公开邀请页使用独立路由 `/invite/:token`，不挂在后台导航下；浏览器分享链接和桌面端 deep link 最终都落到该页面。
