# SmashDreams Blog 部署与配置指南

本文档涵盖从零开始部署博客到腾讯云，以及配置所有功能（GitHub App、Giscus 评论、Umami 分析、TMDB 追番等）的完整流程。

---

## 目录

1. [腾讯云 Nginx 部署](#1-腾讯云-nginx-部署)
2. [域名与 SSL](#2-域名与-ssl)
3. [GitHub Actions CI/CD](#3-github-actions-cicd)
4. [GitHub App 配置（在线写作）](#4-github-app-配置在线写作)
5. [Giscus 评论系统](#5-giscus-评论系统)
6. [Umami 网站分析](#6-umami-网站分析)
7. [TMDB 追番（可选）](#7-tmdb-追番可选)
8. [Bilibili 追番（可选）](#8-bilibili-追番可选)
9. [本地开发指南](#9-本地开发指南)

---

## 1. 腾讯云 Nginx 部署

### 1.1 服务器初始化

```bash
# SSH 登录服务器
ssh root@<你的服务器IP>

# 更新系统
apt update && apt upgrade -y

# 安装 Nginx
apt install nginx -y

# 启动并设置开机自启
systemctl start nginx
systemctl enable nginx
```

### 1.2 部署博客

```bash
# 本地构建
cd /home/smashdreams/project/my-blog/SmashDreams
pnpm build

# 创建部署目录
ssh root@<服务器IP> "mkdir -p /var/www/smashdreams"

# 上传构建产物
scp -r dist/* root@<服务器IP>:/var/www/smashdreams/

# 上传 Nginx 配置
scp nginx.conf root@<服务器IP>:/etc/nginx/conf.d/smashdreams.conf

# 重载 Nginx
ssh root@<服务器IP> "nginx -t && systemctl reload nginx"
```

### 1.3 验证部署
浏览器访问 `http://<服务器IP>`，看到博客首页即部署成功。

---

## 2. 域名与 SSL

### 2.1 DNS 解析

在腾讯云 DNS 解析控制台添加记录：

| 记录类型 | 主机记录 | 记录值 |
|---------|---------|--------|
| A | @ | 你的服务器公网 IP |
| A | www | 你的服务器公网 IP |

### 2.2 申请 SSL 证书

```bash
# 使用 acme.sh 申请 Let's Encrypt 证书（推荐）
# 安装 acme.sh
curl https://get.acme.sh | sh

# 申请证书（确保域名已指向服务器）
~/.acme.sh/acme.sh --issue -d SmashDreams.online -d www.SmashDreams.online --nginx

# 安装证书
~/.acme.sh/acme.sh --install-cert -d SmashDreams.online \
  --key-file /etc/nginx/ssl/SmashDreams.online.key \
  --fullchain-file /etc/nginx/ssl/SmashDreams.online.pem \
  --reloadcmd "systemctl reload nginx"
```

或者使用腾讯云免费 SSL 证书（推荐）：
1. 登录 [腾讯云 SSL 证书控制台](https://console.cloud.tencent.com/ssl)
2. 申请免费 DV 证书（TrustAsia）
3. 下载 Nginx 格式的证书
4. 上传到服务器 `/etc/nginx/ssl/` 目录

### 2.3 开启 HTTPS

```bash
# 证书就绪后，nginx.conf 中的 HTTPS 部分会自动生效
ssh root@<服务器IP> "nginx -t && systemctl reload nginx"
```

---

## 3. GitHub Actions CI/CD

### 3.1 配置服务器 SSH 密钥

```bash
# 在本地生成部署专用密钥对
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/smashdreams-deploy

# 将公钥添加到服务器的 authorized_keys
ssh-copy-id -i ~/.ssh/smashdreams-deploy.pub root@<服务器IP>

# 查看私钥内容（复制备用）
cat ~/.ssh/smashdreams-deploy
```

### 3.2 配置 GitHub Secrets

在 GitHub 仓库 → Settings → Secrets and variables → Actions 中添加：

| Secret | 值 |
|--------|-----|
| `TENCENT_HOST` | 你的腾讯云服务器 IP |
| `TENCENT_USER` | `root` |
| `TENCENT_SSH_KEY` | 上面生成的私钥内容（完整，含 `-----BEGIN OPENSSH PRIVATE KEY-----`） |
| `TENCENT_PORT` | `22`（默认） |
| `PUBLIC_GITHUB_OWNER` | `SmashDreams` |
| `PUBLIC_GITHUB_REPO` | `SmashDreams-blog` |
| `PUBLIC_GITHUB_BRANCH` | `main` |
| `BLOG_SLUG_KEY` | （可选，用于加密文章 slug） |

### 3.3 GitHub Actions 工作流程

已创建 `.github/workflows/deploy.yml`，每次推送到 `main` 分支时会自动：

1. **Checkout** 代码
2. **安装依赖**（pnpm）
3. **构建**（`pnpm build`，含 pagefind 搜索索引）
4. **rsync 到腾讯云**（通过 SSH 密钥认证）
5. **部署到** `/var/www/smashdreams/`

> **注意**: 首次部署前，需确保服务器 `/var/www/smashdreams/` 目录已存在且 Nginx 配置已上传。

---

## 4. GitHub App 配置（在线写作）

这是博客的**核心功能**，用于在浏览器中在线发布文章和管理配置。

### 4.1 创建 GitHub App

1. 访问 [GitHub Settings → Developer settings → GitHub Apps](https://github.com/settings/apps) → **New GitHub App**
2. 填写：

| 字段 | 值 |
|------|-----|
| GitHub App name | `SmashDreams-blog-writer`（或任意名称） |
| Homepage URL | `https://SmashDreams.online` |
| Webhook | **Active** 取消勾选（不需要） |
| Permissions → Repository → Contents | **Read & write** |
| Permissions → Repository → Metadata | **Read-only** |
| Where can this GitHub App be installed? | **Only on this account** |

3. 点击 **Create GitHub App**
4. 记住 **App ID**（在 General 页面顶部），后面需要

### 4.2 生成私钥

在 GitHub App 设置页面：
1. 滚动到 **Private keys** 部分
2. 点击 **Generate a private key**
3. 下载 `.pem` 文件（**妥善保管！**）
4. 这个 `.pem` 文件就是后面在 `/write` 页面需要导入的密钥

### 4.3 安装 GitHub App

1. 在 GitHub App 设置页面左侧，点击 **Install App**
2. 点击你的用户名旁边的 **Install**
3. 选择 **Only select repositories** → 选择 `SmashDreams/SmashDreams-blog`
4. 点击 **Install**

### 4.4 配置项目

在 `smashdreams.config.yaml` 中已包含：

```yaml
github:
  owner: SmashDreams
  repo: SmashDreams-blog
  branch: main
  appId: 在部署端配置环境变量    # 替换为上面记录的 App ID
  encryptKey: smashdreams_encrypt_key_2024  # 建议改为复杂随机字符串
```

> **注意**: `appId` 也可以通过环境变量配置（优先级更高）：
> - `PUBLIC_GITHUB_APP_ID`

### 4.5 使用流程

1. 浏览器访问 `https://SmashDreams.online/write`
2. 点击 **导入密钥**，选择刚才下载的 `.pem` 文件
3. 授权成功后即可在线发布文章

---

## 5. Giscus 评论系统

### 5.1 准备工作

1. 确保博客仓库是 **公开的**（Giscus 要求）
2. 确保仓库已启用 **GitHub Discussions**

在仓库 Settings → General → Features → 勾选 **Discussions**

### 5.2 安装 Giscus GitHub App

1. 访问 [Giscus App](https://github.com/apps/giscus)
2. 点击 **Install**
3. 选择 `SmashDreams/SmashDreams-blog`

### 5.3 获取配置参数

1. 访问 [giscus.app](https://giscus.app/zh-CN)
2. 在 **仓库** 输入: `SmashDreams/SmashDreams-blog`
3. 页面会自动生成 `repoId`、`categoryId` 等参数
4. 复制这些参数

### 5.4 启用评论

编辑 `smashdreams.config.yaml`:

```yaml
comments:
  enable: true
  type: giscus
  giscus:
    repo: SmashDreams/SmashDreams-blog
    repoId: "R_kgDO..."         # 从 giscus.app 获取
    category: General
    categoryId: "DIC_kwDO..."   # 从 giscus.app 获取
    mapping: pathname
    lang: zh-CN
    inputPosition: top
    reactionsEnabled: '1'
    emitMetadata: '0'
    loading: lazy
```

---

## 6. Umami 网站分析

### 6.1 自建 Umami（推荐）

```bash
# 在腾讯云服务器上用 Docker 部署 Umami（需先安装 Docker）
docker run -d \
  --name umami \
  -p 3000:3000 \
  -e DATABASE_URL=postgresql://umami:umami@localhost:5432/umami \
  -e DATABASE_TYPE=postgresql \
  ghcr.io/umami-software/umami:latest
```

或者使用 [Umami Cloud](https://umami.is/)（免费额度足够个人博客使用）

### 6.2 获取配置参数

在 Umami 后台：
1. 添加站点 → 输入 `SmashDreams.online`
2. 获取 **Website ID** 和 **Share ID**
3. 如使用自建实例，记录 **baseUrl**

### 6.3 启用 Umami

编辑 `smashdreams.config.yaml`:

```yaml
umami:
  enable: true
  baseUrl: https://your-umami-instance.com  # 自建地址，或使用 Umami Cloud
  shareId: "xxxxxxxxxxxxxxxx"               # 公开分享 ID
  websiteId: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"  # 站点 ID
  timezone: Asia/Shanghai
```

---

## 7. TMDB 追番（可选）

### 7.1 获取 TMDB API Key

1. 访问 [TMDB](https://www.themoviedb.org/) 注册账号
2. 前往 [API Settings](https://www.themoviedb.org/settings/api)
3. 申请 API Key（v3 auth）

### 7.2 创建 TMDB List

1. 登录 TMDB，进入你的**个人资料页**
2. 创建新的 **List**（列表），添加想追的动漫
3. 从浏览器地址栏获取 **List ID**（如 `https://www.themoviedb.org/list/8556173` 中的 `8556173`）

### 7.3 配置追番

编辑 `smashdreams.config.yaml`:

```yaml
anime:
  tmdb:
    apiKey: "your-tmdb-api-key"
    listId: "your-list-id"
```

---

## 8. Bilibili 追番（可选）

### 8.1 获取 Bilibili UID

1. 打开 [Bilibili](https://www.bilibili.com/) 并登录
2. 进入**个人空间**
3. 从浏览器地址栏获取 UID（如 `https://space.bilibili.com/1536411565` 中的 `1536411565`）

### 8.2 配置 Bilibili

编辑 `smashdreams.config.yaml`:

```yaml
anime:
  bilibili:
    uid: 'your-bilibili-uid'
```

---

## 9. 本地开发指南

### 9.1 运行开发服务器

```bash
pnpm dev          # 启动开发服务器（默认 http://localhost:4321）
pnpm check        # TypeScript 类型检查
pnpm build        # 生产构建（包含 pagefind 搜索索引）
pnpm preview      # 预览构建产物
```

### 9.2 环境变量

创建 `.env` 文件（基于 `.env.example`）：

```bash
PUBLIC_GITHUB_OWNER=SmashDreams
PUBLIC_GITHUB_REPO=SmashDreams-blog
PUBLIC_GITHUB_BRANCH=main
PUBLIC_GITHUB_APP_ID=   # GitHub App ID
```

---

## 快速部署检查清单

- [ ] Nginx 安装并运行
- [ ] 域名 DNS 已指向服务器
- [ ] SSL 证书已配置
- [ ] GitHub Actions 的 Secrets 已配置
- [ ] GitHub App 已创建并安装到仓库
- [ ] `.env.example` 已更新
- [ ] `smashdreams.config.yaml` 中 `appId` 已填写
- [ ] Giscus 评论仓库开启 Discussions
- [ ] Umami 分析实例已部署
- [ ] `pnpm build` 构建成功
- [ ] GitHub Actions 自动部署验证通过
