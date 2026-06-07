# 问题列表面板改造方案 - 总体概览

## 📌 项目概述

### 需求描述
在 Hermes-WebUI 的会话窗口右侧面板上，显示当前会话中所有用户提出的问题列表，支持快速导航到提问点。

### 核心功能
✅ **问题列表展示** - 右侧面板列出所有用户问题  
✅ **快速导航** - 点击问题自动跳转到提问位置  
✅ **实时更新** - 新问题时自动更新列表  
✅ **视觉反馈** - 消息高亮和选中态显示  

## 🏗️ 技术架构

### 系统架构
```
┌─────────────────────────────────────────────────────────┐
│                    Hermes WebUI                          │
├───────────────────────┬─────────────────┬───────────────┤
│  Left Sidebar         │  Center Chat    │ Right Panel   │
│                       │                 │ (Questions)   │
│  • Sessions List      │ • User Q1 ◄─────┼─→ ① Question1 │
│  • Workspace Nav      │ • Assistant A1  │   2m ago      │
│                       │ • User Q2       │               │
│  • Settings Menu      │ • Assistant A2  │ ② Question2   │
│                       │                 │   1m ago      │
│                       │ • User Q3 ◄─────┼─→ ③ Question3 │
│                       │ • Assistant A3  │   now         │
└───────────────────────┴─────────────────┴───────────────┘
```

### 数据流
```
用户输入问题
    ↓
SSE: submitted_message
    ↓
messages.js: 渲染消息 + 添加 data-message-index
    ↓
panels.js: loadQuestionsPanel()
    ↓
api/routes.py: GET /api/session/{id}/questions
    ↓
提取 role='user' 的消息
    ↓
返回 { questions: [...] }
    ↓
前端: renderQuestionsPanel()
    ↓
用户点击问题
    ↓
selectQuestion(index)
    ↓
scrollIntoView() + 高亮动画
```

## 📁 核心文件修改

| 文件 | 修改内容 | 优先级 | 工作量 |
|------|---------|--------|--------|
| **api/routes.py** | 新增 API 端点 `GET /api/session/{id}/questions` | ⭐⭐⭐⭐⭐ | 30行 |
| **static/panels.js** | 新增问题面板逻辑，包含加载、渲染、导航 | ⭐⭐⭐⭐⭐ | 200行 |
| **static/messages.js** | 为消息添加 `data-message-index` 属性 | ⭐⭐⭐⭐⭐ | 5行 |
| **static/style.css** | 新增样式类（面板、列表、动画等） | ⭐⭐⭐⭐⭐ | 120行 |
| **static/index.html** | 新增问题面板容器（可选）| ⭐⭐⭐ | 3行 |
| **static/i18n.js** | 国际化支持 | ⭐⭐ | 10行 |
| **tests/** | 单元和集成测试 | ⭐⭐ | 200行 |

## 🎯 实施时间表

### Week 1: 基础实现（5-8小时）
- Day 1: 后端 API + 消息标记
- Day 2: 前端面板逻辑
- Day 3: 样式和本地测试

### Week 2: 完善和测试（3-5小时）
- Day 1: 国际化支持
- Day 2: 单元和集成测试
- Day 3: 性能优化和文档

## 🔧 关键设计决策

### 1. 数据提取策略
```
选择方案：后端提取 + 前端渲染
原因：
  • 后端知道完整的消息数据结构
  • 可以统一处理数据验证和格式化
  • 前端无需复杂的逻辑分析
  • 便于缓存和性能优化
```

### 2. 导航实现
```
选择方案：data-message-index 属性 + scrollIntoView()
原因：
  • 不需要修改消息 ID 生成逻辑
  • 简单可靠，无副作用
  • 响应式友好（自动处理不同视口）
  • 支持平滑滚动动画
```

### 3. 实时更新
```
选择方案：防抖 + API 轮询
原因：
  • 防止过频繁的 API 调用
  • 不需要双向通信（SSE 已支持）
  • 简化状态管理
  • 易于测试和调试
```

## 💾 状态管理

### 后端状态（Python）
```python
# 会话消息存储
class Session:
    messages: List[Dict]  # [{ role, content, timestamp, ... }]

# 数据提取（无状态）
def get_session_questions(session_id) -> Dict:
    # 从会话中提取用户消息
    # 返回 { questions: [...], total: int }
```

### 前端状态（JavaScript）
```javascript
// 问题列表状态
_currentQuestionsList = [
  { message_index, question_number, timestamp, text, preview },
  ...
]

// UI 状态
_selectedQuestionIndex = null  // 当前选中
_questionsPanelMode = 'list' | 'empty' | 'error'
```

## 📊 API 规范

### 端点
```
GET /api/session/<session_id>/questions
```

### 请求
```
Headers:
  - X-Hermes-CSRF-Token: (自动添加)
  - Content-Type: application/json
```

### 响应（200 OK）
```json
{
  "questions": [
    {
      "message_index": 0,
      "question_number": 1,
      "timestamp": 1717694400.5,
      "text": "完整问题文本",
      "preview": "完整问题文本..."
    }
  ],
  "total": 1
}
```

### 错误响应
```json
{
  "error": "Session not found"
}
```

## 🎨 UI 设计

### 问题列表项布局
```
┌────────────────────────────────────┐
│ ① What is the difference...        │  
│    2m ago                          │
├────────────────────────────────────┤
│ ② How can I implement...           │
│    1m ago                          │
├────────────────────────────────────┤
│ ③ Can you explain...               │  ← selected
│    now                             │
└────────────────────────────────────┘
```

### 消息高亮效果
```
[原始消息]
    ↓
[闪烁高亮 2秒]  ← 使用 @keyframes
    ↓
[恢复正常]
```

## 🧪 测试策略

### 单元测试
- API 返回正确的问题列表
- 问题预览文本截断
- 时间戳格式化
- 错误处理

### 集成测试
- 消息渲染时添加索引属性
- 新消息时实时更新列表
- 点击跳转到正确位置
- 多会话间独立性

### UI 测试
- 桌面和移动响应式
- 暗黑/亮色主题支持
- 多语言显示
- 性能基准（API < 200ms）

## 📈 性能指标

| 指标 | 目标 | 测试方法 |
|------|------|---------|
| API 响应时间 | < 200ms | `curl` + `time` |
| 面板渲染时间 | < 100ms | DevTools Performance |
| 滚动帧率 | ≥ 50fps | DevTools Rendering |
| 内存增长 | < 5MB | DevTools Memory |
| 问题加载（100+） | < 500ms | 虚拟滚动后 |

## 🔄 扩展路线图

### Phase 1: MVP ✅
- [x] 基础问题列表
- [x] 快速导航
- [x] 实时更新

### Phase 2: 增强 🔄
- [ ] 问题搜索过滤
- [ ] 时间分组统计
- [ ] 虚拟滚动（100+ 问题）
- [ ] 复制问题功能

### Phase 3: 高级功能 📅
- [ ] 问题标签/分类
- [ ] 导出 Markdown/PDF
- [ ] 跨会话问题索引
- [ ] 语义相似性检测

## ⚠️ 已知限制和注意事项

### 当前版本的局限
1. **单用户场景** - 假设消息来自单个用户
2. **不支持编辑** - 假设消息提交后不可修改
3. **问题 > 100 个** - 建议实施虚拟滚动
4. **离线场景** - 依赖网络连接

### 最佳实践
1. 使用防抖减少 API 调用
2. 缓存问题列表数据
3. 使用相对时间显示（"2m ago"）
4. 提供清晰的加载和错误状态

## 📚 相关文档

| 文档 | 用途 |
|------|------|
| [QUESTIONS_PANEL_DESIGN.md](./QUESTIONS_PANEL_DESIGN.md) | 详细设计和需求分析 |
| [QUESTIONS_PANEL_IMPLEMENTATION.md](./QUESTIONS_PANEL_IMPLEMENTATION.md) | 实现指南和架构图 |
| [QUESTIONS_PANEL_CODE_SAMPLES.md](./QUESTIONS_PANEL_CODE_SAMPLES.md) | 完整代码示例 |
| [QUESTIONS_PANEL_QUICKSTART.md](./QUESTIONS_PANEL_QUICKSTART.md) | 快速开始指南 |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Hermes-WebUI 架构文档 |

## 🎓 学习资源

### 核心概念
- **SSE (Server-Sent Events)** - 实时更新机制
- **DOM 数据属性** - `data-*` 属性用法
- **CSS 动画** - `@keyframes` 动画实现
- **JavaScript 防抖** - 控制频率的函数调用

### 相关技术
- Flask 路由和响应处理
- 原生 JavaScript DOM 操作
- CSS 布局和响应式设计
- HTTP API 设计最佳实践

## 🚀 快速开始

### 最快路径（30分钟）
1. 复制代码示例到相应文件
2. 重启服务器
3. 打开浏览器测试

### 详细路径（2小时）
1. 阅读设计文档
2. 按照快速开始指南逐步实施
3. 编写和运行测试
4. 进行性能优化

### 完整路径（5小时）
1. 理解整个架构
2. 按照实现指南详细学习
3. 自定义和优化代码
4. 添加额外功能和测试
5. 文档更新和发布

## ❓ 常见问题

### Q: 对现有代码有影响吗？
A: 最小化影响。只添加新函数和样式，仅修改消息渲染和 SSE 处理两个地方。

### Q: 支持大规模会话吗？
A: MVP 支持 < 100 问题。超过 100 个建议实施虚拟滚动。

### Q: 如何处理网络错误？
A: API 失败时显示 "Failed to load" 状态，用户可以手动刷新。

### Q: 国际化如何支持？
A: 通过 i18n 系统支持多语言，包含翻译键和相对时间格式化。

### Q: 支持移动设备吗？
A: 完全支持。使用响应式 CSS，适配各种视口大小。

## 📞 获取支持

遇到问题请查阅：
1. QUESTIONS_PANEL_QUICKSTART.md - 故障排查部分
2. docs/troubleshooting.md - 通用故障排查
3. 浏览器开发者工具 - 查看错误和网络请求
4. 项目 GitHub Issues - 搜索相关问题

## 📝 贡献指南

如果你改进或优化了这个功能：
1. 更新相关文档
2. 添加更多测试用例
3. 提交 PR 并通过代码审查
4. 遵循 CONTRIBUTING.md 的规范

---

**版本**: 1.0  
**最后更新**: 2026-06-06  
**状态**: 设计完成，等待实施  

祝你实施顺利！如有问题，请参考相关文档或提出 Issue。🚀

