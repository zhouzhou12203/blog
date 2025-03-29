---
title: hexo公式渲染
katex: true
date: 2025-03-29 22:20:04
tags: hexo
categories: 网络研究
---

**解决方案：**

我们避免使用 `keep` 主题的 `inject` 功能，而是采用以下方式手动集成 KaTeX。

**步骤 1: 安装 `hexo-renderer-markdown-it` (如果尚未安装)**

   ```bash
   npm install hexo-renderer-markdown-it --save
   ```
   如果安装了其他的 markdown 渲染器，需要卸载其他的 markdown 渲染器。

**步骤 2: 修改 `themes/keep/layout/_partial/head.ejs`**

1.  打开 `themes/keep/layout/_partial/head.ejs` 文件。
2.  在 `<head>` 标签的结束位置 `</head>` **之前**，添加以下代码块：

```html
<% if (page.katex) { %>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@latest/dist/katex.min.css">
  <script src="https://cdn.jsdelivr.net/npm/katex@latest/dist/katex.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/katex@latest/dist/contrib/auto-render.min.js"></script>
  <script>
    document.addEventListener("DOMContentLoaded", function() {
      renderMathInElement(document.body, {
        delimiters: [
          {left: '$$', right: '$$', display: true},
          {left: '$', right: '$', display: false},
          {left: '\\(', right: '\\)', display: false},
          {left: '\\[', right: '\\]', display: true}
        ]
      });
    });
  </script>
<% } %>
```

**步骤 3: 修改 Hexo 根目录下的 `_config.yml`**

1.  确保**没有** `mathjax` 或 `katex` 相关的配置。 如果有，注释掉或者删除它们，以避免冲突。

**步骤 4: 在需要渲染公式的文章的 Front-matter 中启用 KaTeX**

   在需要显示 KaTeX 公式的所有文章的 Front-matter 中，添加 `katex: true`。 例如：

```yaml
---
title: hexo公式渲染
katex: true
date: 2025-03-29 22:20:04
tags: hexo
categories: 网络研究
---
```

**步骤 5: 在文章中使用 KaTeX 公式**

   现在可以在文章中使用 KaTeX 公式了。 使用 `$$...$$` (显示模式) 或 `$...$` (行内模式) 包裹公式。 例如：

```markdown
This is an inline equation: $E=mc^2$.

This is a block equation:
$$
\int_{-\infty}^{\infty} e^{-x^2} dx = \sqrt{\pi}
$$
```

**步骤 6: 清理并重新生成网站**

   ```bash
   hexo clean
   hexo generate
   hexo server
   ```

**完整配置文件示例:**

*   **`themes/keep/layout/_partial/head.ejs` (片段):**

```html
  <meta charset="utf-8">
  </head>

  <% if (page.katex) { %>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@latest/dist/katex.min.css">
    <script src="https://cdn.jsdelivr.net/npm/katex@latest/dist/katex.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/katex@latest/dist/contrib/auto-render.min.js"></script>
    <script>
      document.addEventListener("DOMContentLoaded", function() {
        renderMathInElement(document.body, {
          delimiters: [
            {left: '$$', right: '$$', display: true},
            {left: '$', right: '$', display: false},
            {left: '\\(', right: '\\)', display: false},
            {left: '\\[', right: '\\]', display: true}
          ]
        });
      });
    </script>
  <% } %>
</head>
```

*   **文章 Front-matter (示例):**

```yaml
---
title: hexo公式渲染
katex: true
date: 2025-03-29 22:20:04
tags: hexo
categories: 网络研究
---
```

*   **Hexo 根目录下的 `_config.yml`:**

    ```yaml
    # 确保没有 mathjax 或 katex 相关配置
    # 如果有，请注释掉或删除
    ```

**总结:**

这种方法通过手动在 `head.ejs` 中添加 KaTeX 的 CSS 和 JavaScript 链接，以及在文章的 Front-matter 中使用 `katex: true` 标志来控制 KaTeX 的加载。 这样可以避免与 `keep` 主题的 `inject` 功能冲突，并确保 KaTeX 只在需要的页面加载。

