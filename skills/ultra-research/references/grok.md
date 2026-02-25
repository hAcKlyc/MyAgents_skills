# Grok 交互参考

> 选择器基于 2026-02-25 实测。Grok 的独特价值是**能搜索 X/Twitter 内容**，时效性信息（讨论、观点、趋势）是其强项。

## 登录检查

- 已登录：能看到个人头像按钮（如 `button "pfp"`）
- 未登录：出现 `link "登录"` + `link "注册"`

## 可用模式

| 模式 | 描述 | 适用 |
|------|------|------|
| 快速模式 | Grok 4.1，快速回复 | 简单事实查证 |
| 专家模式 | Thinks hard，深度推理 | 分析性问题 |
| DeepSearch | 独立按钮，深度搜索 | 多源搜索、广度覆盖 |

还有 Grok 4.20(Beta) 和 Heavy 模式，但稳定性未知，一般用快速/专家即可。

## Grok 快速/专家模式

1. `browser_navigate` → `https://grok.com`
2. `browser_snapshot` → 检查登录
3. 检查模型选择器（`button "模型选择"` 或类似）
   - 当前模式显示在按钮内（如 `"专家模式"`、`"快速模式"`）
   - 需要切换 → 点击按钮 → 从菜单选择对应模式
   - 模式选项：`menuitem "快速模式 ..."` / `menuitem "专家模式 ..."`
4. 输入查询 → 提交
   - **输入区域可能是 `paragraph` 而非 `textbox`**（和 ChatGPT 类似）
   - 用 snapshot 确认当前元素类型
   - 如果 `submit: true` 不起作用，手动找发送按钮点击

## Grok DeepSearch

1. 新开 Tab → `https://grok.com`
2. 检查登录 + 进入页面
3. 找到 `button "DeepSearch"` 并点击（在输入框下方，和 Imagine 并列）
4. DeepSearch 激活后输入查询 → 提交
5. DeepSearch 可能有使用次数限制（免费账号），遇到限制提示就跳过

## 完成检测

- **快速/专家模式**：screenshot 看——回复完成 + 无加载动画 + 出现操作按钮
- **DeepSearch**：screenshot 看——搜索完成提示 + 完整报告渲染。可能需要较长时间（类似其他 DR 模式）

## 内容提取

Grok 的回复结构相对简单。用 snapshot 或 evaluate 找到回复内容容器：

```javascript
async (page) => {
  return await page.evaluate(() => {
    // Grok 的回复通常在 article 或带有消息样式的容器中
    // 先尝试常见选择器，失败则用 snapshot 探测
    const articles = document.querySelectorAll('article');
    if (articles.length) return articles[articles.length - 1].innerText;
    // 备选：找最长的文本块
    const blocks = document.querySelectorAll('[class*="message"], [class*="response"], [class*="content"]');
    let longest = '';
    blocks.forEach(b => {
      if (b.innerText.length > longest.length) longest = b.innerText;
    });
    return longest || null;
  });
}
```

**选择器很可能变化**——Grok 的 DOM 结构首次记录，没有多次验证。如果脚本返回空，用 `browser_snapshot` 探测页面结构重新定位。

### 引用链接

Grok 搜索模式通常会附带引用来源（特别是 X/Twitter 帖子链接）。提取方式类似其他服务：

```javascript
async (page) => {
  return await page.evaluate(() => {
    const seen = new Set();
    const links = [];
    // 找回复区域内的链接
    const articles = document.querySelectorAll('article');
    const container = articles.length ? articles[articles.length - 1] : document.body;
    container.querySelectorAll('a[href]').forEach(a => {
      const href = a.href;
      const text = a.innerText.trim().split('\n')[0].trim();
      if (href.startsWith('http') && !href.includes('grok.com') && text && !seen.has(href)) {
        seen.add(href);
        links.push('- [' + text + '](' + href + ')');
      }
    });
    return links.join('\n');
  });
}
```

**X/Twitter 链接是高价值信息**——Grok 的独特优势就在这里，提取时要特别注意保留 x.com/twitter.com 的帖子链接。

## 已知特性

1. **输入框可能是 `paragraph`** — 和 ChatGPT 类似，不要依赖 `submit: true`
2. **DeepSearch 有次数限制** — 免费账号可能有每日配额
3. **X/Twitter 内容是核心价值** — Grok 能搜到其他 AI 搜不到的推文和讨论
4. **DOM 结构首次记录** — 选择器可靠性低于其他服务，多用 snapshot 探测
5. **模式菜单中英混合** — 模式名中文，描述英文（如 `"专家模式 Thinks hard"`）
