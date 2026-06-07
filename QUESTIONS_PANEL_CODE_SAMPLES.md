# 问题列表面板 - 代码示例

这个文件包含可直接使用的代码片段，展示完整的实现细节。

## 1. 后端 API 实现 (api/routes.py)

```python
# 在 api/routes.py 的路由定义中添加

@app.get("/api/session/<session_id>/questions")
def get_session_questions(session_id):
    """
    获取会话中所有用户提出的问题列表。
    
    Response: {
        "questions": [
            {
                "message_index": 0,
                "question_number": 1,
                "timestamp": 1717694400.5,
                "text": "问题的完整文本",
                "preview": "问题的前100个字符..."
            }
        ],
        "total": 1
    }
    """
    from api.models import get_session
    
    # 验证会话存在
    session = get_session(session_id)
    if not session:
        return bad(404, "Session not found")
    
    questions = []
    user_message_count = 0
    
    # 遍历会话中的所有消息
    for idx, msg in enumerate(session.messages or []):
        # 仅处理用户消息
        if msg.get('role') == 'user':
            text = (msg.get('content') or '').strip()
            if text:
                # 生成预览文本（前100个字符）
                preview = text[:100]
                if len(text) > 100:
                    preview += '...'
                
                questions.append({
                    'message_index': idx,              # 消息在列表中的绝对位置
                    'question_number': user_message_count + 1,  # 这是第几个问题
                    'timestamp': msg.get('timestamp', 0),  # Unix 时间戳
                    'text': text,                      # 完整问题文本
                    'preview': preview,                # 截断的预览
                })
                user_message_count += 1
    
    return j({
        'questions': questions,
        'total': len(questions)
    })
```

## 2. 前端面板逻辑 (static/panels.js)

### 2.1 状态声明

```javascript
// 在文件顶部与其他状态声明一起添加

let _currentQuestionsList = [];        // 问题列表缓存
let _selectedQuestionIndex = null;     // 当前选中的问题索引
let _questionsPanelMode = 'empty';     // 'empty' | 'list'
let _questionsLoadingTimer = null;     // 防抖定时器
```

### 2.2 核心函数实现

```javascript
/**
 * 加载会话中的问题列表
 * 从 /api/session/{id}/questions 端点获取数据
 */
async function loadQuestionsPanel() {
  if (!S || !S.session) return;
  
  const sessionId = S.session.session_id;
  if (!sessionId) return;
  
  try {
    // 调用后端 API
    const response = await fetch(
      `/api/session/${encodeURIComponent(sessionId)}/questions`,
      {
        method: 'GET',
        credentials: 'include',
        headers: {
          'Content-Type': 'application/json'
        }
      }
    );
    
    if (!response.ok) {
      if (response.status === 404) {
        // 会话不存在
        _currentQuestionsList = [];
        _questionsPanelMode = 'empty';
        renderQuestionsPanel();
        return;
      }
      throw new Error(`HTTP ${response.status}`);
    }
    
    const data = await response.json();
    _currentQuestionsList = data.questions || [];
    _questionsPanelMode = _currentQuestionsList.length > 0 ? 'list' : 'empty';
    _selectedQuestionIndex = null;  // 重置选中状态
    
    renderQuestionsPanel();
  } catch (err) {
    console.error('Failed to load questions panel:', err);
    _questionsPanelMode = 'error';
    renderQuestionsPanel();
  }
}

/**
 * 防抖版本的 loadQuestionsPanel
 * 用于在频繁事件中调用，防止过度请求
 */
function loadQuestionsPanelDebounced() {
  if (_questionsLoadingTimer) clearTimeout(_questionsLoadingTimer);
  _questionsLoadingTimer = setTimeout(() => {
    loadQuestionsPanel();
    _questionsLoadingTimer = null;
  }, 300);  // 300ms 防抖
}

/**
 * 渲染问题列表 UI
 */
function renderQuestionsPanel() {
  const container = document.getElementById('questionsPanel');
  if (!container) return;
  
  // 清空容器
  container.innerHTML = '';
  
  if (_questionsPanelMode === 'empty') {
    // 空状态显示
    const emptyDiv = document.createElement('div');
    emptyDiv.className = 'panel-empty';
    
    const message = typeof t === 'function' 
      ? t('no_questions_yet', 'No questions yet') 
      : 'No questions yet';
    emptyDiv.textContent = message;
    
    container.appendChild(emptyDiv);
    return;
  }
  
  if (_questionsPanelMode === 'error') {
    const errorDiv = document.createElement('div');
    errorDiv.className = 'panel-error';
    errorDiv.textContent = 'Failed to load questions';
    container.appendChild(errorDiv);
    return;
  }
  
  // 创建列表容器
  const listDiv = document.createElement('div');
  listDiv.className = 'questions-list';
  
  // 为每个问题创建列表项
  _currentQuestionsList.forEach((question, index) => {
    const itemDiv = document.createElement('div');
    itemDiv.className = 'question-item';
    if (index === _selectedQuestionIndex) {
      itemDiv.classList.add('selected');
    }
    itemDiv.dataset.messageIndex = question.message_index;
    itemDiv.dataset.questionIndex = index;
    
    // 问题编号徽章
    const numberBadge = document.createElement('span');
    numberBadge.className = 'question-number';
    numberBadge.textContent = question.question_number;
    numberBadge.setAttribute('title', `Question #${question.question_number}`);
    
    // 问题预览文本
    const previewDiv = document.createElement('div');
    previewDiv.className = 'question-preview';
    previewDiv.textContent = question.preview;
    previewDiv.setAttribute('title', question.text);  // 完整文本作为 tooltip
    
    // 时间戳
    const timestampDiv = document.createElement('div');
    timestampDiv.className = 'question-timestamp';
    timestampDiv.textContent = formatQuestionTime(question.timestamp);
    
    // 组装列表项
    itemDiv.appendChild(numberBadge);
    itemDiv.appendChild(previewDiv);
    itemDiv.appendChild(timestampDiv);
    
    // 点击事件处理
    itemDiv.addEventListener('click', (e) => {
      e.preventDefault();
      e.stopPropagation();
      selectQuestion(index);
    });
    
    listDiv.appendChild(itemDiv);
  });
  
  container.appendChild(listDiv);
  
  // 应用国际化
  if (typeof applyLocaleToDOM === 'function') {
    applyLocaleToDOM();
  }
}

/**
 * 选中一个问题
 * - 更新选中状态
 * - 滚动到对应消息
 * - 应用高亮效果
 */
function selectQuestion(questionIndex) {
  if (!Array.isArray(_currentQuestionsList) || questionIndex < 0 || questionIndex >= _currentQuestionsList.length) {
    return;
  }
  
  const question = _currentQuestionsList[questionIndex];
  const messageIndex = question.message_index;
  
  // 更新选中状态
  _selectedQuestionIndex = questionIndex;
  
  // 重新渲染列表（高亮选中项）
  renderQuestionsPanel();
  
  // 在聊天区域中找到对应的消息
  const messageEl = document.querySelector(`[data-message-index="${messageIndex}"]`);
  if (!messageEl) {
    console.warn(`Message element with index ${messageIndex} not found`);
    return;
  }
  
  // 平滑滚动到消息
  messageEl.scrollIntoView({
    behavior: 'smooth',
    block: 'nearest'
  });
  
  // 添加高亮动画（2秒）
  messageEl.classList.add('highlighted');
  setTimeout(() => {
    messageEl.classList.remove('highlighted');
  }, 2000);
}

/**
 * 格式化问题的时间戳
 * 相对时间显示（如 "5 minutes ago"）
 */
function formatQuestionTime(timestamp) {
  if (!timestamp) return '';
  
  try {
    const date = new Date(timestamp * 1000);  // 转换为毫秒
    const now = new Date();
    const diffMs = now - date;
    
    const seconds = Math.floor(diffMs / 1000);
    const minutes = Math.floor(seconds / 60);
    const hours = Math.floor(minutes / 60);
    const days = Math.floor(hours / 24);
    
    // 多语言支持
    const t_func = typeof t === 'function' ? t : (k, def) => def;
    
    if (seconds < 60) {
      return t_func('just_now', 'just now');
    } else if (minutes < 60) {
      return t_func('minutes_ago', '{n}m ago').replace('{n}', minutes);
    } else if (hours < 24) {
      return t_func('hours_ago', '{n}h ago').replace('{n}', hours);
    } else if (days < 7) {
      return t_func('days_ago', '{n}d ago').replace('{n}', days);
    } else {
      // 显示完整日期
      return date.toLocaleDateString();
    }
  } catch (err) {
    return '';
  }
}
```

### 2.3 与现有面板系统集成

```javascript
// 在面板导航/标签部分添加

// 如果使用标签式导航，添加问题面板标签
const PANEL_DEFINITIONS = {
  // ... 现有面板 ...
  questions: {
    id: 'questions',
    label: 'tab_questions',  // i18n 键
    icon: '?',                // 或使用 SVG icon
    order: 4,
    onLoad: loadQuestionsPanel,
    onRender: renderQuestionsPanel,
    canDisplay: () => S && S.session && S.session.session_id
  }
};

// 在面板切换函数中调用
function showPanel(panelName) {
  _currentPanel = panelName;
  
  if (panelName === 'questions') {
    loadQuestionsPanel();
  }
  
  // ... 其他面板逻辑 ...
}
```

## 3. 消息标记改造 (static/messages.js)

```javascript
/**
 * 修改消息渲染函数，添加数据属性
 * 这样可以通过 data-message-index 识别每条消息
 */

// 找到现有的消息渲染函数（通常是 renderMessage 或类似名称）
// 并修改为：

function renderMessage(msg, index, insertBefore = null) {
  // 构建消息容器
  const msgDiv = document.createElement('div');
  
  // 关键修改：添加 data-message-index 属性
  msgDiv.className = 'message ' + msg.role;
  msgDiv.dataset.messageIndex = index;           // ← 新增
  msgDiv.dataset.messageId = msg.id || '';
  msgDiv.dataset.messageRole = msg.role;
  
  // ... 现有的消息内容构建代码 ...
  
  // 例如：
  const contentDiv = document.createElement('div');
  contentDiv.className = 'message-content';
  contentDiv.textContent = msg.content;
  msgDiv.appendChild(contentDiv);
  
  // 插入 DOM
  const messagesContainer = document.getElementById('messages');
  if (insertBefore) {
    messagesContainer.insertBefore(msgDiv, insertBefore);
  } else {
    messagesContainer.appendChild(msgDiv);
  }
  
  return msgDiv;
}

/**
 * 在 SSE 消息处理中，新消息到达时刷新问题列表
 * 修改现有的 handleMessageReceived 或 onmessage 处理器
 */

// 原有的代码中找到处理消息的地方，添加：
function handleMessageReceived(data) {
  // ... 现有的消息处理逻辑 ...
  
  // 新增：刷新问题列表面板
  if (typeof loadQuestionsPanelDebounced === 'function') {
    loadQuestionsPanelDebounced();
  }
}
```

## 4. 样式实现 (static/style.css)

```css
/* ============================================
   Questions Panel Styles
   ============================================ */

#questionsPanel {
  display: flex;
  flex-direction: column;
  overflow-y: auto;
  overflow-x: hidden;
  height: 100%;
}

.questions-list {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-sm, 8px);
  padding: var(--spacing-md, 12px);
}

.question-item {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-sm, 8px);
  padding: var(--spacing-md, 12px);
  border-radius: var(--rounded-md, 8px);
  border: 1px solid var(--color-border-subtle, #3B4A50);
  cursor: pointer;
  transition: all 0.2s ease;
  background-color: var(--color-surface-subtle, #22333B);
  position: relative;
}

.question-item:hover {
  background-color: var(--color-surface, #11100E);
  border-color: var(--color-border, #3B4A50);
  transform: translateX(2px);
}

.question-item.selected {
  background-color: var(--color-primary-subtle, rgba(230, 125, 34, 0.1));
  border-color: var(--color-primary, #E67D22);
  box-shadow: 0 0 0 3px var(--color-primary-subtle, rgba(230, 125, 34, 0.1));
}

.question-item > :first-child {
  display: flex;
  align-items: center;
  gap: var(--spacing-sm, 8px);
}

.question-number {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  min-width: 24px;
  border-radius: 50%;
  background-color: var(--color-primary, #E67D22);
  color: var(--color-neutral, #0A0908);
  font-size: 11px;
  font-weight: 600;
  flex-shrink: 0;
}

.question-item.selected .question-number {
  background-color: var(--color-primary-dark, #D45920);
}

.question-preview {
  flex: 1;
  font-size: 13px;
  line-height: 1.4;
  color: var(--color-text, #EAE0D5);
  word-break: break-word;
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  padding: 0 var(--spacing-sm, 8px);
  cursor: inherit;
}

.question-preview:hover {
  -webkit-line-clamp: unset;
}

.question-timestamp {
  font-size: 11px;
  color: var(--color-text-secondary, #C6AC8F);
  white-space: nowrap;
  margin-left: 32px;  /* 对齐到问题文本 */
}

/* ============================================
   Message Highlighting (聊天区域中的消息高亮)
   ============================================ */

.message {
  transition: background-color 0.2s ease;
}

.message.highlighted {
  animation: question-highlight 2s ease-in-out;
}

@keyframes question-highlight {
  0% {
    background-color: transparent;
  }
  20%, 80% {
    background-color: var(--color-highlight, rgba(230, 125, 34, 0.2));
  }
  100% {
    background-color: transparent;
  }
}

/* ============================================
   Empty State
   ============================================ */

.panel-empty {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: var(--color-text-secondary, #C6AC8F);
  text-align: center;
  padding: var(--spacing-lg, 16px);
  font-size: 13px;
}

.panel-error {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: var(--color-error, #F87171);
  text-align: center;
  padding: var(--spacing-lg, 16px);
  font-size: 13px;
}

/* ============================================
   Responsive Design
   ============================================ */

@media (max-width: 768px) {
  .question-item {
    padding: var(--spacing-sm, 8px);
    gap: var(--spacing-xs, 4px);
  }
  
  .question-preview {
    font-size: 12px;
    -webkit-line-clamp: 1;
  }
  
  .question-timestamp {
    margin-left: 28px;
    font-size: 10px;
  }
}

/* ============================================
   Dark Mode Support (自动继承)
   ============================================ */

:root {
  /* 使用现有的 CSS 变量，无需额外定义 */
}

.dark {
  /* 自动使用深色主题的变量 */
}
```

## 5. 国际化支持 (static/i18n.js 或翻译文件)

```javascript
// 添加到翻译文件中（通常是 JSON 或 JS 对象）

const translations = {
  en: {
    'tab_questions': 'Questions',
    'no_questions_yet': 'No questions yet',
    'question_count': '{n} questions',
    'just_now': 'just now',
    'minutes_ago': '{n}m ago',
    'hours_ago': '{n}h ago',
    'days_ago': '{n}d ago',
    'question_label': 'Question #{n}',
    'click_to_jump': 'Click to jump to question',
  },
  
  'zh-CN': {
    'tab_questions': '提问',
    'no_questions_yet': '暂无提问',
    'question_count': '{n} 个提问',
    'just_now': '刚刚',
    'minutes_ago': '{n} 分钟前',
    'hours_ago': '{n} 小时前',
    'days_ago': '{n} 天前',
    'question_label': '第 {n} 个问题',
    'click_to_jump': '点击跳转到问题处',
  },
  
  'ja': {
    'tab_questions': '質問',
    'no_questions_yet': 'まだ質問がありません',
    'question_count': '{n} 個の質問',
    'just_now': 'たった今',
    'minutes_ago': '{n} 分前',
    'hours_ago': '{n} 時間前',
    'days_ago': '{n} 日前',
    'question_label': '質問 #{n}',
    'click_to_jump': 'クリックして質問に移動',
  }
};
```

## 6. 单元测试示例

```python
# tests/test_questions_panel.py

import pytest
from api.models import Session
from api.routes import app

@pytest.fixture
def test_client():
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

def test_get_session_questions_empty(test_client):
    """测试没有问题的会话"""
    session_id = "test-session-empty"
    # ... 创建空会话 ...
    
    resp = test_client.get(f"/api/session/{session_id}/questions")
    assert resp.status_code == 200
    data = resp.get_json()
    assert data['questions'] == []
    assert data['total'] == 0

def test_get_session_questions_multiple(test_client):
    """测试多个问题的会话"""
    session_id = "test-session-multi"
    messages = [
        {'role': 'user', 'content': 'What is Python?', 'timestamp': 1000},
        {'role': 'assistant', 'content': 'Python is...', 'timestamp': 1001},
        {'role': 'user', 'content': 'How to learn?', 'timestamp': 1002},
        {'role': 'assistant', 'content': 'You can...', 'timestamp': 1003},
    ]
    # ... 创建包含消息的会话 ...
    
    resp = test_client.get(f"/api/session/{session_id}/questions")
    assert resp.status_code == 200
    data = resp.get_json()
    assert len(data['questions']) == 2
    assert data['total'] == 2
    
    # 验证第一个问题
    q1 = data['questions'][0]
    assert q1['question_number'] == 1
    assert q1['message_index'] == 0
    assert 'What is Python?' in q1['text']
    
    # 验证第二个问题
    q2 = data['questions'][1]
    assert q2['question_number'] == 2
    assert q2['message_index'] == 2

def test_question_preview_truncation(test_client):
    """测试长问题的截断"""
    long_text = "a" * 200
    session_id = "test-session-long"
    messages = [
        {'role': 'user', 'content': long_text, 'timestamp': 1000},
    ]
    # ... 创建会话 ...
    
    resp = test_client.get(f"/api/session/{session_id}/questions")
    data = resp.get_json()
    q = data['questions'][0]
    
    assert len(q['preview']) <= 103  # 100 + "..."
    assert q['preview'].endswith('...')
    assert len(q['text']) == 200  # 完整文本未截断

def test_non_existent_session(test_client):
    """测试不存在的会话"""
    resp = test_client.get("/api/session/non-existent/questions")
    assert resp.status_code == 404
```

## 7. 集成测试示例 (JavaScript/Playwright)

```javascript
// tests/test_questions_panel_integration.js

describe('Questions Panel Integration', () => {
  beforeEach(async () => {
    // 访问会话页面
    await page.goto('/session/test-session-123');
    // 等待聊天加载
    await page.waitForSelector('#messages');
  });
  
  test('should display questions panel on mount', async () => {
    const panel = await page.$('#questionsPanel');
    expect(panel).toBeTruthy();
  });
  
  test('should load and display questions', async () => {
    // 触发加载
    await page.evaluate(() => loadQuestionsPanel());
    
    // 等待列表渲染
    await page.waitForSelector('.question-item');
    
    const items = await page.$$('.question-item');
    expect(items.length).toBeGreaterThan(0);
  });
  
  test('should scroll to message on click', async () => {
    await page.evaluate(() => loadQuestionsPanel());
    await page.waitForSelector('.question-item');
    
    // 点击第一个问题
    await page.click('.question-item:first-child');
    
    // 验证消息被高亮
    const highlightedMsg = await page.$('.message.highlighted');
    expect(highlightedMsg).toBeTruthy();
  });
  
  test('should highlight selected question', async () => {
    await page.evaluate(() => loadQuestionsPanel());
    await page.waitForSelector('.question-item');
    
    // 点击问题
    await page.click('.question-item:nth-child(1)');
    
    // 检查是否有 selected 类
    const selected = await page.$('.question-item.selected');
    expect(selected).toBeTruthy();
  });
  
  test('should update on new question submission', async () => {
    const initialCount = await page.evaluate(() => _currentQuestionsList.length);
    
    // 提交新问题
    const composer = await page.$('#msg');
    await composer.type('New test question');
    await page.click('button[type="submit"]');
    
    // 等待 SSE 事件处理
    await new Promise(r => setTimeout(r, 500));
    
    // 验证列表已更新
    const newCount = await page.evaluate(() => _currentQuestionsList.length);
    expect(newCount).toBe(initialCount + 1);
  });
});
```

---

这些代码片段可以直接复制使用，只需根据实际项目结构进行小的调整。

