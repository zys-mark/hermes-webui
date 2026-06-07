# Hermes WebUI 问题列表面板改造方案

## 1. 需求分析

### 1.1 功能需求
- **会话问题列表**：在右侧面板显示当前会话中所有用户提出的问题
- **快速导航**：点击列表中的问题可以快速跳转到对应的提问点
- **实时更新**：当新问题提出时，列表实时更新
- **视觉反馈**：支持问题高亮和滚动定位

### 1.2 用户流程
1. 用户在会话中提出多个问题
2. 右侧面板自动列出所有问题
3. 用户点击某个问题，自动滚动到该问题的消息位置
4. 被选中的问题在列表和聊天区域均有视觉强调

---

## 2. 架构改造方案

### 2.1 后端改造 (Python)

#### 2.1.1 消息解析和提取
**文件**: `api/routes.py`

添加新的 API 端点用于获取会话中的问题列表：

```python
@app.get("/api/session/<session_id>/questions")
def get_session_questions(session_id):
    """
    返回会话中所有用户问题的列表
    Response: {
        "questions": [
            {
                "turn_id": "unique-turn-id",
                "message_index": 0,
                "timestamp": 1234567890,
                "text": "用户提出的问题",
                "character_count": 50,
                "reply_index": 1  # 对应的助手回复索引
            }
        ]
    }
    """
```

#### 2.1.2 数据源
从 `models.py` 中的会话数据结构提取：
- 使用 `Session.messages` 中 `role == 'user'` 的消息
- 每条消息记录在消息列表中的索引 (用于滚动定位)
- 提取消息的前100个字符作为预览
- 记录消息的时间戳

### 2.2 前端改造 (JavaScript/CSS)

#### 2.2.1 新增面板模块
**文件**: `static/panels.js`

新增 `questions` 面板，类似现有的 `cron`、`profiles` 等面板：

```javascript
// 面板状态管理
let _currentQuestionsList = [];
let _selectedQuestionIndex = null;
let _questionsPanelMode = 'list'; // 'list' | 'empty'

// 核心函数
function loadQuestions(sessionId) {
  // 从后端获取问题列表
  // 更新 _currentQuestionsList
  // 重新渲染列表
}

function renderQuestionsList() {
  // 生成问题列表的 HTML
  // 包括问题预览、时间、索引
}

function selectQuestion(questionIndex) {
  // 更新选中状态
  // 触发滚动到对应消息
  // 高亮处理
}
```

#### 2.2.2 右侧面板集成
**文件**: `static/panels.js` 中的面板切换逻辑

在现有面板列表中添加 `questions` 面板：
- 在面板头部添加问题图标和标签
- 集成到面板导航中

#### 2.2.3 消息区域改造
**文件**: `static/messages.js` 和 `static/ui.js`

为用户消息添加可交互的功能：
- 每条用户消息添加 `data-message-index` 属性
- 支持从问题列表点击时的滚动和高亮

### 2.3 样式改造

**文件**: `static/style.css`

新增样式类：
- `.questions-panel` - 面板容器
- `.questions-list` - 列表容器
- `.question-item` - 单个问题项
- `.question-item.selected` - 选中状态
- `.question-preview` - 问题预览文本
- `.question-timestamp` - 时间戳
- `.question-index` - 消息索引号

---

## 3. 详细实现步骤

### 步骤1：后端API实现

**修改**: `api/routes.py`

```python
@app.get("/api/session/<session_id>/questions")
def get_session_questions(session_id):
    from api.models import S, get_session
    
    session = get_session(session_id)
    if not session:
        return bad(404, "Session not found")
    
    questions = []
    user_message_count = 0
    
    for idx, msg in enumerate(session.messages):
        if msg.get('role') == 'user':
            text = msg.get('content', '').strip()
            if text:
                preview = text[:100]
                if len(text) > 100:
                    preview += '...'
                
                questions.append({
                    'message_index': idx,
                    'question_number': user_message_count + 1,
                    'timestamp': msg.get('timestamp', 0),
                    'text': text,
                    'preview': preview,
                })
                user_message_count += 1
    
    return j({
        'questions': questions,
        'total': len(questions)
    })
```

### 步骤2：前端面板实现

**修改**: `static/panels.js`

添加以下函数：

```javascript
async function loadQuestionsPanel() {
  if (!S.session) return;
  
  try {
    const resp = await fetch(`/api/session/${S.session.session_id}/questions`);
    if (!resp.ok) return;
    
    const data = await resp.json();
    _currentQuestionsList = data.questions || [];
    _questionsPanelMode = _currentQuestionsList.length > 0 ? 'list' : 'empty';
    renderQuestionsPanel();
  } catch (err) {
    console.error('Failed to load questions:', err);
  }
}

function renderQuestionsPanel() {
  const container = document.getElementById('questionsPanel');
  if (!container) return;
  
  if (_questionsPanelMode === 'empty') {
    container.innerHTML = '<div class="panel-empty">' +
      (typeof t === 'function' ? t('no_questions_yet', 'No questions yet') : 'No questions yet') +
      '</div>';
    return;
  }
  
  const list = document.createElement('div');
  list.className = 'questions-list';
  
  _currentQuestionsList.forEach((q, idx) => {
    const item = document.createElement('div');
    item.className = 'question-item';
    if (idx === _selectedQuestionIndex) {
      item.classList.add('selected');
    }
    item.dataset.messageIndex = q.message_index;
    item.dataset.questionIndex = idx;
    
    const numBadge = document.createElement('span');
    numBadge.className = 'question-number';
    numBadge.textContent = q.question_number;
    
    const preview = document.createElement('div');
    preview.className = 'question-preview';
    preview.textContent = q.preview;
    
    const timestamp = document.createElement('div');
    timestamp.className = 'question-timestamp';
    timestamp.textContent = formatTime(q.timestamp);
    
    item.appendChild(numBadge);
    item.appendChild(preview);
    item.appendChild(timestamp);
    
    item.addEventListener('click', () => {
      selectQuestion(idx);
    });
    
    list.appendChild(item);
  });
  
  container.innerHTML = '';
  container.appendChild(list);
}

function selectQuestion(questionIndex) {
  const q = _currentQuestionsList[questionIndex];
  if (!q) return;
  
  _selectedQuestionIndex = questionIndex;
  
  // 更新列表视觉
  renderQuestionsPanel();
  
  // 滚动到对应消息
  const messageEl = document.querySelector(
    `[data-message-index="${q.message_index}"]`
  );
  if (messageEl) {
    messageEl.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
    messageEl.classList.add('highlighted');
    setTimeout(() => messageEl.classList.remove('highlighted'), 2000);
  }
}

function formatTime(timestamp) {
  if (!timestamp) return '';
  const date = new Date(timestamp * 1000);
  const now = new Date();
  const diff = now - date;
  
  if (diff < 60000) return 'just now';
  if (diff < 3600000) return Math.floor(diff / 60000) + 'm ago';
  if (diff < 86400000) return Math.floor(diff / 3600000) + 'h ago';
  return date.toLocaleDateString();
}
```

### 步骤3：消息标记改造

**修改**: `static/messages.js`

在消息渲染时添加索引标记：

```javascript
function renderMessage(msg, index) {
  // ... 现有代码 ...
  
  const msgDiv = document.createElement('div');
  msgDiv.className = 'message ' + msg.role;
  msgDiv.dataset.messageIndex = index;  // 新增
  msgDiv.dataset.messageId = msg.id || '';
  
  // ... 后续渲染代码 ...
}
```

### 步骤4：样式实现

**修改**: `static/style.css`

```css
/* Questions Panel */
#questionsPanel {
  overflow-y: auto;
  overflow-x: hidden;
}

.questions-list {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-sm);
  padding: var(--spacing-md);
}

.question-item {
  display: flex;
  align-items: flex-start;
  gap: var(--spacing-md);
  padding: var(--spacing-md);
  border-radius: var(--rounded-md);
  border: 1px solid var(--color-border);
  cursor: pointer;
  transition: all 0.2s ease;
  background-color: var(--color-surface-subtle);
}

.question-item:hover {
  background-color: var(--color-surface);
  border-color: var(--color-border-emphasis);
}

.question-item.selected {
  background-color: var(--color-primary-subtle);
  border-color: var(--color-primary);
}

.question-number {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  min-width: 24px;
  border-radius: 50%;
  background-color: var(--color-primary);
  color: var(--color-neutral);
  font-size: 12px;
  font-weight: 600;
}

.question-preview {
  flex: 1;
  font-size: 13px;
  line-height: 1.4;
  color: var(--color-text);
  word-break: break-word;
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
}

.question-timestamp {
  font-size: 11px;
  color: var(--color-text-secondary);
  white-space: nowrap;
  margin-top: 4px;
}

/* Message Highlighting */
.message.highlighted {
  animation: highlight-pulse 2s ease-in-out;
}

@keyframes highlight-pulse {
  0%, 100% {
    background-color: transparent;
  }
  50% {
    background-color: var(--color-highlight);
  }
}

.panel-empty {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 200px;
  color: var(--color-text-secondary);
  text-align: center;
  padding: var(--spacing-lg);
}
```

### 步骤5：面板导航集成

**修改**: `static/panels.js` 中的面板切换逻辑

在现有面板列表中添加：

```javascript
const PANEL_TABS = {
  // ... 现有面板 ...
  questions: {
    icon: '?',  // 或使用 SVG
    label: 'tab_questions',  // i18n key
    load: loadQuestionsPanel,
    render: renderQuestionsPanel
  }
};
```

### 步骤6：消息收到时的自动刷新

**修改**: `static/messages.js` 中的 SSE 事件处理

```javascript
function handleMessageReceived(turn) {
  // ... 现有代码 ...
  
  // 新增：刷新问题列表
  if (typeof loadQuestionsPanel === 'function') {
    loadQuestionsPanel();
  }
}
```

---

## 4. 国际化支持

**修改**: `static/i18n.js` 或对应的 i18n 配置

添加以下翻译键：

```javascript
{
  'tab_questions': {
    'en': 'Questions',
    'zh-CN': '提问',
  },
  'no_questions_yet': {
    'en': 'No questions yet',
    'zh-CN': '暂无提问',
  },
  'question_count': {
    'en': '{n} questions',
    'zh-CN': '{n} 个提问',
  }
}
```

---

## 5. 测试计划

### 5.1 单元测试
- [ ] 后端 API 返回正确的问题列表
- [ ] 问题提取逻辑处理空消息
- [ ] 时间戳格式化正确

### 5.2 集成测试
- [ ] 新消息时实时更新问题列表
- [ ] 点击问题能正确滚动到消息
- [ ] 多会话间问题列表独立

### 5.3 UI 测试
- [ ] 面板在桌面和移动设备上正确显示
- [ ] 问题预览文本正确截断
- [ ] 选中状态视觉效果清晰
- [ ] 滚动动画流畅

---

## 6. 性能考虑

### 6.1 优化策略
1. **虚拟滚动**：对于超过100个问题的会话，实现虚拟滚动
2. **懒加载**：首次打开面板时按需加载
3. **缓存**：会话会话期间缓存问题列表，减少API调用
4. **增量更新**：新问题时仅追加而非全量重新加载

### 6.2 实现建议
```python
# 后端缓存策略
_questions_cache = {}  # { session_id: (timestamp, questions) }
CACHE_TTL = 300  # 5分钟

def get_session_questions_cached(session_id):
    cached = _questions_cache.get(session_id)
    now = time.time()
    if cached and (now - cached[0]) < CACHE_TTL:
        return cached[1]
    
    questions = compute_session_questions(session_id)
    _questions_cache[session_id] = (now, questions)
    return questions
```

---

## 7. 扩展功能建议

### 7.1 第一阶段（基础版本）
- [x] 问题列表展示
- [x] 快速导航
- [x] 实时更新

### 7.2 第二阶段（增强版本）
- [ ] 问题搜索/过滤
- [ ] 按时间/主题分组
- [ ] 问题标签/分类
- [ ] 问题统计（总数、类型分布等）

### 7.3 第三阶段（高级功能）
- [ ] 问题导出（markdown/pdf）
- [ ] 问题书签
- [ ] 跨会话问题索引
- [ ] 语义相似性检测（找重复问题）

---

## 8. 相关文件总结

| 文件 | 修改内容 | 优先级 |
|------|--------|--------|
| api/routes.py | 新增问题列表API | 必须 |
| static/panels.js | 新增问题面板逻辑 | 必须 |
| static/messages.js | 添加消息索引标记 | 必须 |
| static/style.css | 新增样式类 | 必须 |
| static/ui.js | 可选的UI辅助函数 | 可选 |
| static/i18n.js | 国际化支持 | 建议 |
| tests/ | 新增相应测试 | 建议 |

---

## 9. 集成检查清单

- [ ] 后端API已实现并测试
- [ ] 前端面板已实现
- [ ] 样式已添加并测试响应式布局
- [ ] 消息索引标记已添加
- [ ] SSE事件处理已集成
- [ ] 国际化已完成
- [ ] 单元测试已编写
- [ ] 集成测试已编写
- [ ] UI测试已执行
- [ ] 文档已更新（CONTRIBUTING.md, ARCHITECTURE.md）

