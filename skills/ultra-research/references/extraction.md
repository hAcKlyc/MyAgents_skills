# 通用提取技术

提取是研究流程中最脆弱的环节——DOM 随时变，没有一招通杀。这里记录通用策略和降级路径。各服务的具体选择器和脚本在各自的 reference 文件中。

## 核心原则

**`browser_run_code` 是首选提取方式。** 一次执行完整脚本，比逐步 `browser_evaluate` 更高效。只在 `browser_run_code` 失败时才降级。

## 降级路径

按优先级尝试：

1. **`browser_run_code`** — 执行各 reference 中的提取脚本
2. **`browser_evaluate`** — 单步执行选择器（脚本整体失败时拆开用）
3. **`browser_snapshot`** — 直接从 accessibility tree 拿文本（选择器全失效时）
4. **逐屏 snapshot 拼接** — 滚动 + 多次 snapshot（内容超长或结构复杂时）
5. **`browser_take_screenshot`** — 最后手段，至少留个截图记录

选择器失效时，不要反复重试同一个选择器。用 `browser_snapshot` 探测当前页面结构，构造新选择器。

## 分块提取

`browser_evaluate` 返回值上限约 **10,000 字符**。DR 结果常有 30,000-50,000 字符。

```javascript
// 1. 获取总长度
() => { return targetElement.innerText.length; }
// 2. 分段提取（每段 ≤10,000 字符）
() => { return targetElement.innerText.substring(0, 10000); }
() => { return targetElement.innerText.substring(10000, 20000); }
// ... 以此类推
```

`browser_run_code` 没有这个限制——如果需要提取长内容，优先用它。

## 引用链接提取

正文提取后，额外提取引用 URL。各服务的链接位置不同：

- **ChatGPT** — 在 article 内部，但文本有引用计数污染（`\n+1`），需清洗
- **Gemini DR** — 不在报告 panel 里，在页面侧边栏，需全页面扫描
- **Grok** — 搜索模式附带引用来源，特别是 X/Twitter 帖子链接（高价值信息）
- **Claude** — 通常不带引用链接（原创分析模式）

各服务的具体提取脚本见各自 reference 文件。

通用清洗模板：
```javascript
// 清洗链接文本 + 去重
const seen = new Set();
const links = [];
container.querySelectorAll('a[href]').forEach(a => {
  let href = a.href;
  // 截断过长的 text fragment
  const fragIdx = href.indexOf('#:~:text=');
  if (fragIdx > 0) href = href.substring(0, fragIdx);
  let text = a.innerText.trim().split('\n')[0].trim();
  if (href.startsWith('http') && text && !seen.has(href)) {
    seen.add(href);
    links.push('- [' + text + '](' + href + ')');
  }
});
```

## 保存规范

- 每份结果保存为独立 md，编号递增：`01-gemini-pro.md`、`02-chatgpt-dr.md`...
- 后续轮次继续编号：`06-gemini-pro-子问题.md`
- 引用链接附在文件末尾 `## 参考链接` 章节
- 提取失败不影响正文保存——能拿多少存多少
- 每保存一份，更新 todo.md
