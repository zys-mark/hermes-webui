# 问题列表面板 - 实现指南

## 快速参考架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        Hermes WebUI                              │
├──────────────────────┬──────────────────────┬──────────────────┤
│                      │                      │                  │
│   Left Sidebar       │   Center Chat        │  Right Panel     │
│  (Sessions)          │   (Messages)         │  (新增!)         │
│                      │                      │                  │
│  [Session List]      │  User Q1: "..."      │  ┌────────────┐ │
│                      │  Assistant: "..."    │  │ Questions  │ │
│                      │                      │  ├────────────┤ │
│                      │  User Q2: "..."      │  │ ① Q1 ...   │ │
│                      │  (highlighted)◄──────┼──┤   5min ago │ │
│                      │  Assistant: "..."    │  ├────────────┤ │
│                      │                      │  │ ② Q2 ...   │ │
│                      │  User Q3: "..."      │  │   2min ago │ │
│                      │  Assistant: "..."    │  ├────────────┤ │
│                      │                      │  │ ③ Q3 ...   │ │
│                      │                      │  │   now      │ │
│                      │                      │  └────────────┘ │
└──────────────────────┴──────────────────────┴──────────────────┘
       ▲                         ▲                       ▲
       │                         │                       │
    存储/读取              渲染消息，添加          点击跳转，
    会话列表              data-message-index     高亮消息
```

## 数据流工作流

```
1️⃣  用户提交新问题
   │
   └─► SSE: submitted_message (role='user')
       │
       ├─► messages.js: handleMessageReceived()
       │   └─► 添加到 S.messages[]，刷新显示
       │
       └─► loadQuestionsPanel() 触发
           │
           └─► fetch /api/session/{id}/questions
               │
               ├─► api/routes.py: extract user messages
               │   └─► 构建 { questions: [...] }
               │
               └─► renderQuestionsPanel()
                   └─► 更新右侧列表UI


2️⃣  用户点击右侧问题
   │
   └─► selectQuestion(index)
       │
       ├─► 更新 _selectedQuestionIndex
       ├─► renderQuestionsPanel() 高亮选中项
       │
       └─► scrollIntoView + 消息高亮动画
           └─► 用户看到该问题的聊天内容
```

## 模块依赖图

```
┌─────────────────────────────────────────────────────────────────┐
│                      server.py (路由层)                          │
└──────────────┬────────────────────────┬──────────────────────────┘
               │
       ┌───────┴────────────────────────┴────────────────┐
       │                                                  │
┌──────▼────────────┐                          ┌─────────▼────────┐
│   api/routes.py   │                          │ api/models.py    │
│ (新增endpoint)    │◄─────── Session ────────►│ (session 数据)   │
└──────┬────────────┘                          └────────┬─────────┘
       │                                                │
       │ /api/session/{id}/questions                    │
       │返回 { questions: [ ... ] }                     │
       │                                                │
       └────────────────┬──────────────────────────────┘
                        │
                        │ HTTP JSON
                        │ (CORS, CSRF保护)
                        │
                    ┌───▼────────────────────────────────────┐
                    │   浏览器 (静态资源)                     │
                    │                                        │
        ┌───────────┴────┬─────────────┬──────────────────┐ │
        │                │             │                  │ │
    ┌───▼──────┐  ┌──────▼────┐  ┌────▼──────┐  ┌───────▼─┐
    │panels.js │  │messages.js│  │style.css  │  │i18n.js  │
    │(问题面板)│  │(消息标记) │  │(样式类)   │  │(翻译)   │
    └──────────┘  └───────────┘  └───────────┘  └─────────┘
```

## 状态管理流程

### 后端状态 (Python)

```python
# Session 模型 (已有)
class Session:
    session_id: str
    messages: List[Message]  # [{ role: 'user'|'assistant', content: str, ... }]
    created_at: timestamp
    
# 新增 API 层
def get_session_questions(session_id) -> Dict:
    session = get_session(session_id)
    questions = []
    for idx, msg in enumerate(session.messages):
        if msg['role'] == 'user':
            questions.append({
                'message_index': idx,
                'question_number': len(questions) + 1,
                'timestamp': msg.get('timestamp', 0),
                'preview': msg['content'][:100],
                'text': msg['content']
            })
    return {'questions': questions, 'total': len(questions)}
```

### 前端状态 (JavaScript)

```javascript
// Global state in panels.js
let S = {
  session: { session_id, title, ... },      // 当前会话
  messages: [ /* 消息列表，每个有 data-message-index */ ]
};

let _currentQuestionsList = [
  {
    message_index: 0,      // 在 S.messages 中的索引
    question_number: 1,    // 第几个问题
    timestamp: 1234567890,
    preview: "用户提问的前100个字符...",
    text: "完整问题文本"
  },
  // ... 更多问题
];

let _selectedQuestionIndex = null;  // 当前选中的问题索引
let _questionsPanelMode = 'list' | 'empty';
```

## 交互时序图

```
用户                浏览器              WebSocket/SSE          服务器
 │                  │                        │                  │
 │─ 输入问题 ─────►│                         │                  │
 │                  │─ 发送消息 ─────────────►│                  │
 │                  │                        │─ 保存到 S.messages
 │                  │◄─ SSE event ───────────│                  │
 │                  │ submitted_message      │                  │
 │                  │                        │                  │
 │                  │ loadQuestionsPanel()   │                  │
 │                  │─ GET /api/session/... ─┼─────────────────►│
 │                  │                        │  extract questions
 │                  │◄─ JSON response ───────┼──────────────────│
 │                  │  { questions: [...] }  │                  │
 │                  │                        │                  │
 │                  │ renderQuestionsPanel() │                  │
 │                  │ 列表显示在右侧面板      │                  │
 │                  │                        │                  │
 │─ 点击列表问题 ──►│ selectQuestion(idx)    │                  │
 │                  │                        │                  │
 │                  │ scrollIntoView() +      │                  │
 │                  │ 高亮动画               │                  │
 │                  │                        │                  │
 │◄─ 看到聊天内容 ──│                        │                  │
```

## 消息标记详解

在 `static/messages.js` 中：

```html
<!-- 渲染后的结构 -->
<div id="messages">
  <div class="message user" data-message-index="0">
    <!-- Q1 内容 -->
  </div>
  
  <div class="message assistant" data-message-index="1">
    <!-- A1 内容 -->
  </div>
  
  <div class="message user" data-message-index="2">
    <!-- Q2 内容 -->
  </div>
  
  <div class="message assistant" data-message-index="3">
    <!-- A2 内容 -->
  </div>
</div>

<!-- 问题列表点击后，滚动到并高亮 data-message-index="2" -->
```

## 实现优先级

### Phase 1: MVP (最小可行产品) - Week 1
```
Priority: CRITICAL
├─ 后端API: 获取问题列表
├─ 前端面板: 显示问题列表
├─ 消息标记: 添加 data-message-index
├─ 导航: 点击跳转 (简单 scrollIntoView)
└─ 样式: 基础布局和选中状态
```

### Phase 2: 完善 (Polish) - Week 2
```
Priority: HIGH
├─ 实时更新: SSE 集成
├─ 高亮效果: 消息脉冲动画
├─ i18n: 多语言支持
├─ 缓存策略: API 优化
└─ 测试: 单元和集成测试
```

### Phase 3: 增强 (Enhanced) - Week 3+
```
Priority: MEDIUM
├─ 虚拟滚动: 大规模问题列表
├─ 搜索过滤: 问题关键词搜索
├─ 统计面板: 问题类型分析
└─ 导出功能: Markdown/PDF 导出
```

## 文件修改清单

### 必须修改

```
api/routes.py
  ├─ 新增 @app.get("/api/session/<session_id>/questions")
  └─ 增加 30-50 行代码

static/panels.js
  ├─ 新增 loadQuestionsPanel() 函数
  ├─ 新增 renderQuestionsPanel() 函数
  ├─ 新增 selectQuestion() 函数
  ├─ 新增 formatTime() 辅助函数
  ├─ 添加到面板导航
  └─ 增加 150-200 行代码

static/messages.js
  ├─ 在消息渲染时添加 data-message-index
  ├─ SSE 事件处理中调用 loadQuestionsPanel()
  └─ 修改 10-20 行代码

static/style.css
  ├─ .questions-panel
  ├─ .questions-list
  ├─ .question-item
  ├─ .question-item.selected
  ├─ .question-item:hover
  ├─ .question-number
  ├─ .question-preview
  ├─ .question-timestamp
  ├─ .message.highlighted
  ├─ @keyframes highlight-pulse
  └─ 增加 100-150 行样式代码
```

### 建议修改

```
static/i18n.js
  └─ 添加新的 i18n 键

ARCHITECTURE.md
  └─ 更新文档描述新面板

TESTING.md
  └─ 添加手动测试指南

tests/test_questions_panel.py (新建)
  └─ 单元测试
  
tests/test_questions_integration.py (新建)
  └─ 集成测试
```

## API 规范

### 请求

```
GET /api/session/<session_id>/questions

Query Parameters:
  - limit: int (可选，默认100)
  - offset: int (可选，默认0)

Headers:
  - X-Hermes-CSRF-Token: string (自动添加)
```

### 响应成功 (200)

```json
{
  "questions": [
    {
      "message_index": 0,
      "question_number": 1,
      "timestamp": 1717694400.5,
      "text": "完整的问题文本，可能很长",
      "preview": "完整的问题文本，可能很长..."
    },
    {
      "message_index": 2,
      "question_number": 2,
      "timestamp": 1717694500.3,
      "text": "第二个问题",
      "preview": "第二个问题"
    }
  ],
  "total": 2
}
```

### 响应错误 (404, 500 等)

```json
{
  "error": "Session not found"
}
```

## 测试场景

### 单元测试

```python
def test_get_session_questions_empty():
    # 没有问题的会话
    
def test_get_session_questions_multiple():
    # 多个问题
    
def test_question_preview_truncation():
    # 长问题截断为100字符
    
def test_question_ordering():
    # 问题按时间顺序排列
    
def test_non_existent_session():
    # 返回 404
```

### 集成测试

```javascript
describe('Questions Panel', () => {
  it('should display questions on mount', async () => {
    // 打开会话，问题列表应显示
  });
  
  it('should scroll to message on click', async () => {
    // 点击列表中的问题，应滚动到对应消息
  });
  
  it('should highlight selected question', async () => {
    // 选中问题应有视觉强调
  });
  
  it('should update on new question', async () => {
    // 用户提出新问题后，列表自动更新
  });
  
  it('should work on mobile viewport', async () => {
    // 响应式布局测试
  });
});
```

## 性能基准

### 预期数据量

| 会话规模 | 消息数 | 问题数 | API 响应时间 |
|---------|--------|--------|-------------|
| 小型    | < 50   | < 10   | < 50ms      |
| 中型    | 50-200 | 10-50  | 50-100ms    |
| 大型    | 200+   | 50+    | 100-200ms   |

### 优化目标

- API 响应时间: < 200ms
- 面板渲染时间: < 100ms
- 滚动帧率: 60fps
- 内存占用: < 5MB (问题列表 + DOM)

## 常见问题 (FAQ)

### Q: 如何处理非常长的问题（> 10000字符）？
A: 后端在提取时限制在前 10000 个字符，preview 限制在 100 字符。前端渲染时使用 text-overflow: ellipsis。

### Q: 问题列表是否应该包含助手回复？
A: 不应该。仅列出用户问题，可以通过点击快速看到完整的 QA 对。

### Q: 如何处理编辑或删除的消息？
A: 目前版本假设消息不可编辑/删除。如果支持，需要在后端进行额外的验证（检查 is_deleted 标志等）。

### Q: 多用户会话中如何区分提问者？
A: 如果支持多用户，可在 question_item 中添加 avatar 和 username。当前版本假设单用户。

### Q: 是否应该支持暗黑模式？
A: 是的。使用现有的 CSS 变量系统（--color-*, --spacing-* 等）自动支持。

