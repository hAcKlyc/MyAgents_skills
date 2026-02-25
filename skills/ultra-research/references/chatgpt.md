# ChatGPT 交互参考

> **ChatGPT 是所有服务中最不稳定的。** Cloudflare 拦截、输入框类型变化、A/B 测试按钮名、空 snapshot 状态——需要最多防御性处理。选择器基于 2026-02-25 实测，会变。

## 登录检查

- 已登录：能看到个人资料图标或用户名（如 `generic "用户名"` 或头像按钮）
- 未登录：出现登录按钮

## 导航注意

**首次 navigate 到 `chatgpt.com` 可能被 Cloudflare 拦截**（"Unable to load site"）。重试 1-2 次通常就通过。这不是登录问题，是 Cloudflare 的反自动化检测。

## ChatGPT Thinking

1. 新开 Tab → `https://chatgpt.com`
2. 检查登录
3. 确认模型选择器显示 Thinking 模式
   - 模型选择器标签可能是 "ChatGPT 5.2 Thinking" 或类似
   - 不是 → 点击选择器 → 选 Thinking 相关选项
   - **注意**：中文 UI 下按钮名可能是 "进阶思考" 或 "发散性思考"（ChatGPT 在 A/B 测试），别被不同的按钮名困住
4. 输入查询 → 提交
   - **输入框不稳定**：可能渲染为 `textbox` 或 `paragraph`
   - `submit: true` 在 `paragraph` 上会失败
   - **稳妥做法**：先 `browser_type` 填入文本，再单独找发送按钮点击

## ChatGPT Deep Research

1. 新开 Tab → `https://chatgpt.com`
2. 检查登录
3. 切换到 Deep Research 模式
   - 通常是点击 "+" 按钮或附件按钮 → 选择 "深度研究"
   - 用 snapshot 确认当前 UI 结构再操作
   - 验证：页面应出现类似"你在研究什么？"的提示
4. 输入查询 → 提交（同样注意输入框稳定性）
5. **可能不直接启动研究**：ChatGPT DR 有时先回澄清问题
   - 等 15-30 秒 → screenshot 检查
   - 如果没看到研究面板（"活动/来源"侧边栏）→ 说明未启动
   - 回复一条消息让它直接开始

## 完成检测

- **Thinking**：screenshot 看——无"停止生成"按钮 + 回复完整 + 出现操作按钮
- **Deep Research**：screenshot 看——右侧面板显示 **"Research completed in Xm"** + **"N 个源"**

## Snapshot 不稳定

ChatGPT 的 SPA 会偶尔进入瞬时状态，`browser_snapshot` 返回完全空的内容。遇到这种情况，等 2-3 秒再试就好，不是严重问题。

## 内容提取

### 通用提取（Thinking & Deep Research）

```javascript
// browser_run_code 版本
async (page) => {
  const result = await page.evaluate(() => {
    const articles = document.querySelectorAll('article');
    if (!articles.length) return null;
    const last = articles[articles.length - 1];
    return last.innerText;
  });
  return result;
}
```

**多轮对话注意**：页面可能有多个 `article`（用户消息 + AI 回复）。最后一个 article 是最新回复。但如果 DR 先回了澄清问题再启动研究，最后一个 article 可能是中间回复——用内容长度辅助判断（DR 报告通常 >10,000 字符）。

降级选择器：

```javascript
// 备选：data-message-author-role
() => {
  const msgs = document.querySelectorAll('[data-message-author-role="assistant"]');
  const last = msgs[msgs.length - 1];
  if (!last) return null;
  const md = last.querySelector('.markdown');
  return (md || last).innerText;
}
```

### 引用链接

ChatGPT 的引用链接在 article 内部，但有两个坑：
- **文本污染**：`innerText` 包含 `\n+1` 等引用计数徽章
- **URL 过长**：DR 链接带 `#:~:text=...` 片段

```javascript
async (page) => {
  return await page.evaluate(() => {
    const articles = document.querySelectorAll('article');
    const el = articles[articles.length - 1];
    if (!el) return '';
    const seen = new Set();
    const links = [];
    el.querySelectorAll('a[href]').forEach(a => {
      let href = a.href;
      const fragIdx = href.indexOf('#:~:text=');
      if (fragIdx > 0) href = href.substring(0, fragIdx);
      let text = a.innerText.trim().split('\n')[0].trim();
      if (href.startsWith('http') && text && !seen.has(href)) {
        seen.add(href);
        links.push('- [' + text + '](' + href + ')');
      }
    });
    return links.join('\n');
  });
}
```

## 已知特性

1. **Cloudflare 拦截** — 首次导航可能被挡，重试即可
2. **输入框类型不稳定** — `textbox` / `paragraph` 随机切换，别依赖 `submit: true`
3. **A/B 测试按钮名** — 同一功能可能叫不同名字（"进阶思考"/"发散性思考"），根据功能而非精确名称来定位
4. **Snapshot 偶尔返回空** — 等几秒重试
5. **DR 可能不启动** — 先回澄清问题，需检测并追加回复
6. **DR 结果很长** — 30,000-50,000 字符，需分块提取
7. **"发散性思考"开关 ≠ Deep Research** — 不要混淆这两个功能
