# Gemini 交互参考

> 选择器基于 2026-02-25 实测验证。DOM 会变——选择器失效时用 `browser_snapshot` 探测当前结构，不要反复重试旧选择器。

## 登录检查

- 已登录：能看到 Google 账号按钮（如 `button "Google 账号：..."`）
- 未登录：出现 `link "登录"` 或登录页面

## Gemini Pro

1. `browser_navigate` → `https://gemini.google.com/app`
2. `browser_snapshot` → 检查登录
3. **检查模式** — Gemini 新对话默认"快速"模式，每次都要确认：
   - 找模式选择器按钮（如 `button "打开模式选择器"` 或其子元素）
   - 不是 "Pro" → 点击切换
4. 输入查询 → 提交
   - 输入框 placeholder 可能变化（曾为 `"在此处输入提示"`，实测也见过 `"为 Gemini 输入提示"`）
   - 用 snapshot 确认当前 placeholder 再操作

## Gemini Deep Research

1. 新开 Tab → `https://gemini.google.com/app`
2. 检查登录
3. 找到工具入口 → 选择 Deep Research
   - 通常是 `button "工具"` → `button "Deep Research"`
   - 验证：placeholder 应变为类似"你想研究什么？"
4. 输入查询 → 提交
5. **两步确认**：提交后 Gemini 先生成研究计划，等用户确认
   - 等 10-20 秒 → snapshot
   - 找到类似 `button "开始研究"` → 点击
   - 没找到就等——可能还在生成计划

## 完成检测

- **Pro**：screenshot 看——无加载动画 + 回复完整 + 出现操作按钮（复制、点赞）
- **Deep Research**：screenshot 看——出现完整报告面板。关键文字：**"我已经完成了研究"**。不要用 progressbar/spinner 判断——它们是 UI 永久元素

## 内容提取

### Pro 模式

```javascript
// browser_run_code 版本
async (page) => {
  // 首选：data-content-type 选择器
  const text = await page.evaluate(() => {
    const turns = document.querySelectorAll('[data-content-type="modelTurn"]');
    if (turns.length) return turns[turns.length - 1].innerText;
    const responses = document.querySelectorAll('model-response');
    if (responses.length) return responses[responses.length - 1].innerText;
    return null;
  });
  return text;
}
```

### Deep Research 模式

DR 报告在独立面板中，不是普通对话消息。面板可能很长（30,000+ 字符）。

```javascript
// browser_run_code 版本 — 自动分块
async (page) => {
  const result = await page.evaluate(() => {
    // 找报告面板：markdown-main-panel 或 innerText 很长的 panel
    const panels = document.querySelectorAll('[class*="panel"]');
    for (const p of panels) {
      if (p.className.includes('markdown-main-panel') || p.innerText.length > 2000) {
        return p.innerText;
      }
    }
    return null;
  });
  return result ? result.substring(0, 50000) : null; // 截断安全上限
}
```

如果 `browser_run_code` 失败，降级到 `browser_evaluate` + 分块提取（见 [extraction.md](extraction.md)）。

### 引用链接

**Pro 不带引用链接**——正常的，Pro 是纯文本回复。

**DR 引用链接不在报告 panel 里**，在页面侧边栏。需全页面扫描：

```javascript
async (page) => {
  return await page.evaluate(() => {
    const seen = new Set();
    const links = [];
    document.querySelectorAll('a[href]').forEach(a => {
      const href = a.href;
      const text = a.innerText.trim().replace(/\n.*$/, '').trim();
      if (href.startsWith('http') && !href.includes('gemini.google.com') && text && !seen.has(href)) {
        seen.add(href);
        links.push('- [' + text + '](' + href + ')');
      }
    });
    return links.join('\n');
  });
}
```

## 已知特性

1. **新对话默认"快速"模式** — 每次新 Tab 都要重新切 Pro
2. **DR 两步确认** — 提交后不自动开始，需手动点"开始研究"
3. **DR 完成检测** — progressbar/spinner 是永久 UI 元素，以文字判断
4. **模式选择器结构可能变** — 有时是 `group` 包 `button`，有时直接是 `button`。用 snapshot 看当前结构
