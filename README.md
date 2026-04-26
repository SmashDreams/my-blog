<p align="center">
  <strong>SmashDreams' Blog</strong> 🚀
  <br>
  <em>在代码与文字之间，寻找无限可能</em>
</p>

<div align="center">
  <img alt="SmashDreams Blog" src="/logo.png" width="200px">
</div>

## 简介

这是 SmashDreams 的个人博客，基于 Astro 5.0+ & Tailwind CSS 构建的静态博客系统。

### 技术栈

- ⚡ **Astro 5** - 极速的静态网站生成器
- 🎨 **Tailwind CSS + daisyUI** - 优雅的 UI 框架
- 📝 **MDX** - 灵活的 Markdown 写作体验
- 🔍 **Pagefind** - 毫秒级全文搜索
- 🌓 **深色/浅色主题** - 舒适的阅读体验

### 功能特性

- 响应式设计（移动端优先）
- 在线文章发布与管理
- 可视化配置编辑器
- RSS 订阅
- 追番管理
- 网站导航
- 评论系统支持
- 访问统计

## 本地开发

```sh
# 安装依赖
pnpm install

# 启动开发服务器
pnpm dev

# 构建
pnpm build

# 本地预览
pnpm preview
```

## 部署

### 腾讯云 Nginx 部署

1. 本地构建：
   ```sh
   pnpm build
   ```

2. 将 `dist/` 目录上传到服务器：
   ```sh
   scp -r dist/ root@your-server-ip:/var/www/smashdreams/
   ```

3. 配置 Nginx（参考 `nginx.conf`），确保域名 `SmashDreams.online` 指向正确目录

### Docker 部署（可选）

```sh
docker build -t smashdreams-blog .
docker run -d -p 80:80 smashdreams-blog
```

## License

MIT
