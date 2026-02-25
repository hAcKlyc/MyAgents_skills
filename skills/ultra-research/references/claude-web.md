# Claude Web 交互参考

> 选择器基于 2026-02-25 验证。Claude 的 web UI 相对稳定，但 DOM 同样会变。

## 登录检查

- 已登录：能看到输入框（如 `textbox "Write your prompt to Claude"`）
- 未登录：出现 `button "Continue with Google"` 或登录页

## Claude Opus Extended

1. 新开 Tab → `https://claude.ai/new`
2. 检查登录
3. 检查模型选择器
   - 应显示 "Opus 4.6 Extended" 或 "Opus 4.6"
   - 不是 → 点击模型选择器 → 选 Opus → 确认 Extended thinking 开关开启
   - 模型选择器可能有 `testid: model-selector-dropdown`，也可能变化
4. 输入查询 → 提交

## 完成检测

screenshot 看：回复完整 + 出现操作按钮（复制、重试等） + 无流式文字动画。如果有 Artifact 面板，也要等它渲染完。

## 内容提取

Claude 的回复分两部分：**对话文本（prose）** 和 **Artifact（代码块/文档）**。很多时候 Artifact 才是主要内容（完整的分析文档），对话部分只是简短说明。两者都要提取。

### Artifact 内容

```javascript
async (page) => {
  return await page.evaluate(() => {
    const preBlocks = document.querySelectorAll('pre');
    const seen = new Set();
    let result = [];
    preBlocks.forEach(p => {
      const text = p.innerText.trim();
      if (text && !seen.has(text)) {
        seen.add(text);
        result.push(text);
      }
    });
    return result.join('\n\n---\n\n');
  });
}
```

**去重关键**：`code` 嵌套在 `pre` 内。只查 `pre` + Set 去重，不要 `querySelectorAll('code, pre')`。

### 对话文本

```javascript
async (page) => {
  return await page.evaluate(() => {
    const main = document.querySelector('[class*="prose"]')
      || document.querySelector('main')
      || document.body;
    const elements = main.querySelectorAll('h1, h2, h3, h4, p, li, blockquote');
    let result = [];
    elements.forEach(el => {
      if (el.closest('pre') || el.closest('code') || el.closest('nav')) return;
      const text = el.innerText.trim();
      if (text) result.push(text);
    });
    return result.join('\n');
  });
}
```

### 合并保存

将 prose + artifact 合并为一份文件。如果 artifact 是完整文档，它通常是主要内容——prose 部分作为上下文说明附在前面。

### 引用链接

**Claude Opus 通常不带引用链接**——它是原创分析模式，不像 DR 附引用来源。偶尔回复中有链接时：

```javascript
async (page) => {
  return await page.evaluate(() => {
    const main = document.querySelector('[class*="prose"]') || document.querySelector('main') || document.body;
    const seen = new Set();
    const links = [];
    main.querySelectorAll('a[href]').forEach(a => {
      if (a.href.startsWith('http') && !a.href.includes('claude.ai') && !a.href.includes('anthropic.com') && a.innerText.trim() && !seen.has(a.href)) {
        seen.add(a.href);
        links.push('- [' + a.innerText.trim() + '](' + a.href + ')');
      }
    });
    return links.join('\n');
  });
}
```

## 已知特性

1. **Artifact 是主要内容** — Claude 经常把长篇分析放 Artifact，对话只有概述。两者都要提取
2. **pre 内嵌套 code** — 只用 `pre` 选择器 + Set 去重
3. **旧选择器可能失效** — `[data-testid="chat-message-content"]` 等旧选择器可能返回空，用通用选择器
4. **不带引用链接** — 正常行为。如需来源，可追问让 Claude 列出参考
