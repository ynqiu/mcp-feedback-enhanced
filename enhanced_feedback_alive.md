# Keep Interactive Feedback Alive — 定时保持连接功能说明

## 功能效果

启用后，当用户在网页端**无操作**超过设定间隔时，页面会**自动提交 `"-"` 作为反馈**，使 AI 的 `interactive_feedback` 调用进入下一个循环，从而防止会话超时断开。

整体流程如下：

```
用户无操作 → 定时器触发 → 自动提交 "-"
→ AI 收到 "-" → AI 重新调用 interactive_feedback
→ 页面刷新出新会话 → 继续等待用户真实反馈
```

---

## 使用方法

1. 打开 MCP Feedback Enhanced 网页 UI
2. 点击 **⚙️ 设置** 标签页
3. 找到 **🔗 定时保持连接** 卡片
4. 勾选 **启用定时保持连接**
5. 设置 **间隔时间**（默认 300 秒 = 5 分钟，建议不低于 60 秒）

---

## 代码修改说明

### 修改文件

`src/mcp_feedback_enhanced/web/static/js/app.js`

### 修改内容

函数 `FeedbackApp.prototype.performKeepAliveSubmit`（约第 2307 行）：

| | 修改前 | 修改后 |
|---|---|---|
| **行为** | 发送静默 `keep_alive` WebSocket 信号，仅延长后端超时截止时间 | 自动提交 `"-"` 作为真实反馈，触发 interactive_feedback 会话循环 |
| **对 AI 的影响** | 对 AI 完全透明，不产生任何消息 | AI 会收到 `"-"` 并重新调用 interactive_feedback |
| **效果** | 防止后端 `wait_for_feedback` 超时 | 强制循环 interactive_feedback，保持整个会话活跃 |

**关键逻辑（修改后）**：
```javascript
FeedbackApp.prototype.performKeepAliveSubmit = function() {
    // 只在 FEEDBACK_WAITING 状态且 WebSocket 就绪时执行
    var currentState = this.uiManager.getFeedbackState();
    if (currentState !== window.MCPFeedback.Utils.CONSTANTS.FEEDBACK_WAITING) {
        return; // 不干扰用户正在进行的操作
    }
    if (!this.webSocketManager.isReady()) { return; }

    // 直接发送 "-" 作为反馈，保持 interactive_feedback 会话循环
    this.webSocketManager.send({
        type: 'submit_feedback',
        feedback: '-',
        images: [],
        settings: {}
    });
};
```

### 同步更新的翻译文件（可选，仅影响 UI 描述文字）

- `src/mcp_feedback_enhanced/web/locales/zh-CN/translation.json`
- `src/mcp_feedback_enhanced/web/locales/zh-TW/translation.json`
- `src/mcp_feedback_enhanced/web/locales/en/translation.json`

---

## 迁移到另一台电脑

### 方式A：Git 同步（推荐）

```bash
# 本机提交
git add src/mcp_feedback_enhanced/web/static/js/app.js
git add src/mcp_feedback_enhanced/web/locales/
git commit -m "feat: auto-submit dash to keep interactive_feedback alive"
git push

# 另一台电脑
git pull
```

### 方式B：直接替换文件

1. 在另一台电脑找到安装路径：
   ```bash
   find ~/.local/share/uv/tools/mcp-feedback-enhanced -name "app.js" 2>/dev/null
   ```
2. 将本机修改后的 `app.js` 复制过去替换即可
   - 本机路径：`/Users/ynqiu/DocsNosync/Python_Project/mcp-feedback-enhanced/src/mcp_feedback_enhanced/web/static/js/app.js`

> **注意**：translation.json 不是必须更新的，只影响设置界面里的功能描述文字，不影响功能本身。
