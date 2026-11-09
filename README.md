# test_blog

Hugo 静态博客，通过 GitHub Pages 自动部署。

## 写作与发布

1. 在 `content/posts/` 下新建 Markdown 文件，frontmatter 必填四个字段：

   ```yaml
   ---
   title: "文章标题"
   date: 2026-11-09T17:00:00+08:00
   tags: ["tag1", "tag2"]
   author: "aatianya"
   ---
   ```

2. 提交 PR，合并到 `main` 分支。
3. GitHub Actions（`.github/workflows/hugo.yml`）自动构建并发布到 GitHub Pages。

也可用 `hugo new content/posts/文章名.md` 生成带上述 frontmatter 的草稿。

## 本地预览

```bash
hugo server -D
```

## 部署地址

https://aatianya.github.io/test_blog/
