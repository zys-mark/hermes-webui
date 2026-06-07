# 问题列表面板 - 快速开始指南

快速参考：按照以下步骤快速实施问题列表面板功能。

## 📋 前置检查清单

- [ ] 已读 QUESTIONS_PANEL_DESIGN.md（设计文档）
- [ ] 已读 QUESTIONS_PANEL_IMPLEMENTATION.md（实现详解）
- [ ] 已读 QUESTIONS_PANEL_CODE_SAMPLES.md（代码示例）
- [ ] Python 3.11+ 环境就绪
- [ ] 熟悉 Hermes-WebUI 的基本结构

## ⚡ 5分钟快速了解

### 功能概览
```
会话右侧面板显示所有用户问题 ➜ 点击问题 ➜ 自动跳转到提问位置 ➜ 消息高亮
```

### 核心改动（4个文件）

| 文件 | 改动 | 代码量 |
|------|------|--------|
| api/routes.py | 新增 API 端点 | 30 行 |
| static/panels.js | 新增面板逻辑 | 200 行 |
| static/messages.js | 添加消息标记 | 5 行 |
| static/style.css | 新增样式类 | 120 行 |

## 🚀 分步实施 (30-60分钟)

### 第1步：添加后端 API (10分钟)

**编辑**: `api/routes.py`

在现有路由后面添加：

```python
@app.get("/api/session/<session_id>/questions")
def get_session_questions(session_id):
    from api.models import get_session
    
    session = get_session(session_id)
    if not session:
        return bad(404, "Session not found")
    
    questions = []
    user_msg_count = 0
    
    for idx, msg in enumerate(session.messages or []):
        if msg.get('role') == 'user':
            text = (msg.get('content') or '').strip()
            if text:
                preview = text[:100] + ('...' if len(text) > 100 else '')
                questions.append({
                    'message_index': idx,
                    'question_number': user_msg_count + 1,
                    'timestamp': msg.get('timestamp', 0),
                    'text': text,
                    'preview': preview,
                })
                user_msg_count += 1
    
    return j({'questions': questions, 'total': len(questions)})
```

**验证**: 
```bash
curl "http://localhost:8000/api/session/test-session/questions"
# 应该返回: {"questions": [...], "total": 1}
```

### 第2步：添加消息标记 (5分钟)

**编辑**: `static/messages.js`

找到消息渲染函数（通常是 `renderMessage` 或 `renderMessages`），在创建消息 div 时添加：

```javascript
const msgDiv = document.createElement('div');
msgDiv.className = 'message ' + msg.role;
msgDiv.dataset.messageIndex = index;  // ← 添加这一行
msgDiv.dataset.messageId = msg.id || '';
```

找到处理新消息的地方（SSE 或 WebSocket），添加：

```javascript
// 在消息渲染后调用
if (typeof loadQuestionsPanelDebounced === 'function') {
  loadQuestionsPanelDebounced();
}
```

### 第3步：实现前端面板 (15分钟)

**编辑**: `static/panels.js`

在文件顶部添加状态声明：

```javascript
let _currentQuestionsList = [];
let _selectedQuestionIndex = null;
let _questionsPanelMode = 'empty';
let _questionsLoadingTimer = null;
```

复制 QUESTIONS_PANEL_CODE_SAMPLES.md 中的以下函数并粘贴到 panels.js：
- `loadQuestionsPanel()`
- `loadQuestionsPanelDebounced()`
- `renderQuestionsPanel()`
- `selectQuestion()`
- `formatQuestionTime()`

在面板导航部分（查找 `_currentPanel` 或类似的面板选择逻辑），添加：

```javascript
// 在面板列表中添加
case 'questions':
  loadQuestionsPanel();
  renderQuestionsPanel();
  break;
```

### 第4步：添加样式 (10分钟)

**编辑**: `static/style.css`

在文件末尾添加 QUESTIONS_PANEL_CODE_SAMPLES.md 中的 CSS 块（从 "Questions Panel Styles" 开始）。

复制以下样式类（粘贴到 style.css）：
- `.questions-list`
- `.question-item`
- `.question-item:hover`
- `.question-item.selected`
- `.question-number`
- `.question-preview`
- `.question-timestamp`
- `.message.highlighted`
- `@keyframes question-highlight`
- `.panel-empty`

### 第5步：集成到 UI (10分钟)

**编辑**: `static/index.html`

在右侧面板容器中添加问题面板 HTML（查找 `<div id="rightPanel">` 或类似）：

```html
<!-- 添加到右侧面板内 -->
<div id="questionsPanel" class="panel-content" style="display: none;">
  <!-- 由 JavaScript 动态生成 -->
</div>
```

**编辑**: `static/boot.js` 或主初始化文件

在面板切换逻辑中添加：

```javascript
// 当切换到问题面板时
function switchToQuestionsPanel() {
  document.getElementById('questionsPanel').style.display = 'block';
  loadQuestionsPanel();
}
```

### 第6步：国际化（可选，5分钟）

**编辑**: `static/i18n.js` 或翻译文件

添加翻译键：

```javascript
{
  'tab_questions': { 'en': 'Questions', 'zh-CN': '提问' },
  'no_questions_yet': { 'en': 'No questions yet', 'zh-CN': '暂无提问' },
  'just_now': { 'en': 'just now', 'zh-CN': '刚刚' },
  'minutes_ago': { 'en': '{n}m ago', 'zh-CN': '{n}分钟前' },
}
```

## 🧪 本地测试 (10分钟)

### 1. 启动服务器

```bash
cd /path/to/hermes-webui
python3 -m pip install -r requirements.txt
python3 bootstrap.py
# 或者
python3 server.py
```

### 2. 打开浏览器

```
http://localhost:8000/session/new
```

### 3. 测试用例

- [ ] **提出第一个问题** → 右侧面板应显示 "No questions yet"
- [ ] **提出第二个问题** → 列表应出现，显示 "① Question 1"
- [ ] **点击列表项** → 应滚动到对应问题
- [ ] **消息高亮** → 被点击的消息应闪烁高亮
- [ ] **提出新问题** → 列表自动更新，显示新问题
- [ ] **时间戳** → 应显示 "just now"、"2m ago" 等相对时间

### 4. 浏览器开发者工具检查

```javascript
// 在控制台输入以下命令进行测试

// 检查状态
console.log(_currentQuestionsList)     // 应显示问题数组
console.log(_selectedQuestionIndex)    // 应显示选中索引或 null

// 手动触发加载
loadQuestionsPanel()

// 检查 API 响应
fetch('/api/session/YOUR_SESSION_ID/questions')
  .then(r => r.json())
  .then(d => console.log(d))

// 手动选择问题
selectQuestion(0)
```

## 📊 验收标准

### 功能验收

- [ ] 后端 API 正确返回问题列表
- [ ] 前端面板正确渲染问题
- [ ] 点击问题能滚动到对应消息
- [ ] 消息高亮动画正常
- [ ] 新问题实时更新列表
- [ ] 时间戳格式正确

### 性能验收

- [ ] API 响应时间 < 200ms
- [ ] 面板渲染时间 < 100ms
- [ ] 滚动帧率 ≥ 50fps
- [ ] 内存占用增加 < 5MB

### 兼容性验收

- [ ] 桌面浏览器正常显示
- [ ] 移动浏览器响应式适配
- [ ] 暗黑/亮色主题都能显示
- [ ] 多国语言正确翻译

## 🐛 故障排查

### 问题1: 列表为空或不显示

**症状**: 右侧面板显示 "No questions yet"，即使有问题

**排查**:
```javascript
// 检查 API 是否正常
fetch('/api/session/YOUR_SESSION_ID/questions')
  .then(r => console.log('Status:', r.status))
  .then(d => console.log('Data:', d))

// 检查消息是否有标记
document.querySelectorAll('[data-message-index]').length
// 应该返回 > 0
```

**修复**:
- 确保 API 端点已添加到 `api/routes.py`
- 确保消息标记已添加到 `static/messages.js`
- 检查会话中确实有用户消息

### 问题2: 点击无反应

**症状**: 点击问题列表但不滚动

**排查**:
```javascript
// 检查消息 DOM 是否存在
document.querySelector('[data-message-index="0"]')
// 应该返回一个 DOM 元素

// 手动尝试滚动
document.querySelector('[data-message-index="0"]')
  ?.scrollIntoView()
```

**修复**:
- 确保消息有 `data-message-index` 属性
- 检查 `selectQuestion()` 函数是否正确实现
- 查看浏览器控制台是否有错误

### 问题3: 样式不对齐

**症状**: 样式显示不正确，或响应式布局破裂

**排查**:
```javascript
// 检查 CSS 变量是否定义
getComputedStyle(document.documentElement)
  .getPropertyValue('--color-primary')
// 应该返回一个颜色值

// 检查样式是否加载
document.styleSheets
```

**修复**:
- 确保所有 CSS 类已添加到 `static/style.css`
- 检查 CSS 变量名称是否与现有的一致
- 刷新浏览器缓存（Ctrl+Shift+Delete）

### 问题4: 性能问题（缓慢或卡顿）

**症状**: 列表加载慢，或滚动不流畅

**排查**:
```javascript
// 测量 API 响应时间
performance.mark('api-start')
fetch('/api/session/...').then(() => {
  performance.mark('api-end')
  performance.measure('api', 'api-start', 'api-end')
  console.log(performance.getEntriesByName('api')[0])
})

// 检查问题列表大小
_currentQuestionsList.length
```

**修复**:
- 如果问题 > 100 个，实施虚拟滚动
- 使用防抖减少 API 调用
- 检查是否有内存泄漏（打开DevTools → Memory）

## 📚 延伸学习

### 相关文档
- [QUESTIONS_PANEL_DESIGN.md](./QUESTIONS_PANEL_DESIGN.md) - 完整设计文档
- [QUESTIONS_PANEL_IMPLEMENTATION.md](./QUESTIONS_PANEL_IMPLEMENTATION.md) - 详细实现指南
- [QUESTIONS_PANEL_CODE_SAMPLES.md](./QUESTIONS_PANEL_CODE_SAMPLES.md) - 代码示例
- [ARCHITECTURE.md](./ARCHITECTURE.md) - Hermes-WebUI 架构
- [CONTRIBUTING.md](./CONTRIBUTING.md) - 贡献指南

### 增强想法

完成基础版后，可以考虑以下增强：

1. **搜索过滤** - 在列表中搜索问题
2. **问题分类** - 按主题/标签分组
3. **批量操作** - 导出问题列表
4. **统计分析** - 问题类型分布图
5. **跨会话索引** - 全局问题搜索

## 📞 获取帮助

如果遇到问题：

1. 查看 [BUGS.md](./BUGS.md) 中的已知问题
2. 查看 [docs/troubleshooting.md](./docs/troubleshooting.md)
3. 在 GitHub Issues 中搜索相关问题
4. 查看测试文件（`tests/`）了解预期行为

## ✅ 完成清单

实施完成后检查：

- [ ] 所有文件已修改
- [ ] 无语法错误（运行 `python -m py_compile api/routes.py`）
- [ ] 所有测试通过（`pytest tests/`）
- [ ] 手动测试用例全部通过
- [ ] 文档已更新
- [ ] 代码已自查
- [ ] 性能满足要求
- [ ] 兼容性测试完毕

## 🎉 下一步

实施完成后：

1. **编写测试** - 添加到 `tests/` 目录
2. **提交 PR** - 按照 CONTRIBUTING.md 提交代码评审
3. **更新文档** - 更新 ARCHITECTURE.md 和 CHANGELOG.md
4. **发布版本** - 更新版本号和发布说明

祝你实施顺利！ 🚀

