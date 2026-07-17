---
title: "从构建到上线：Ubuntu 24.04 + Nginx 部署 Astro 静态网站"
date: 2026-07-17
updated: 2026-07-17
status: draft
category: projects
tags:
  - Astro
  - Ubuntu
  - Nginx
  - Static Deployment
  - rsync
summary: "以 ZGLab 为例，整理 Astro 静态网站从本地构建、服务器目录准备、Nginx 配置、备份发布到回滚验证的完整流程，并区分已经验证的步骤与仍需在真实服务器确认的部分。"
---

# 从构建到上线：Ubuntu 24.04 + Nginx 部署 Astro 静态网站

## 这篇教程解决什么问题

Astro 静态网站执行 `npm run build` 后会生成一个 `dist/` 目录。部署的本质不是“启动 Astro”，而是把 `dist/` 中的 HTML、CSS、JavaScript 和图片交给 Web 服务器。

本文以以下目标环境为例：

- 服务器：Ubuntu 24.04；
- Web 服务器：Nginx；
- 构建模式：Astro static output；
- 发布目录：`/var/www/zglab.fun`；
- 备份目录：`/var/backups/zglab.fun`；
- 发布工具：Bash + rsync；
- 运行要求：生产环境不常驻 Node 服务。

截至 2026-07-17，本文涉及的 Astro 构建、路由访问、部署脚本语法、缺少 `dist/` 时的失败退出，以及临时目录中的备份和同步已经实际验证。**真实 Ubuntu 服务器上的 Nginx、域名解析、文件权限和 HTTPS 尚未在本次操作中完成最终验收，相关步骤需要按实际服务器验证。**

## 读者对象与前置条件

本文适合能够使用 SSH 和基本 Linux 命令，希望将个人网站部署到自己的轻量服务器上的读者。

需要准备：

- 一台可以 SSH 登录的 Ubuntu 24.04 服务器；
- 一个已经完成开发的 Astro 静态项目；
- 一个域名及其 DNS 管理权限；
- 对 `/var/www` 和 Nginx 配置具有 `sudo` 权限；
- 本地或服务器能够使用 Node.js `>= 22.12.0` 完成构建；
- 服务器安装 Nginx 和 rsync。

命令中的 `<server-user>`、`<server-ip>` 和 `<domain>` 都是占位符，必须替换为真实值。不要把密码或 Token 写入脚本和仓库。

## 部署架构

```mermaid
flowchart LR
    SRC[Astro 源码] --> CHECK[格式与类型检查]
    CHECK --> BUILD[npm run build]
    BUILD --> DIST[dist/]
    DIST --> BACKUP[备份现有站点]
    BACKUP --> RSYNC[rsync 同步]
    RSYNC --> WWW[/var/www/zglab.fun]
    WWW --> NGINX[Nginx]
    NGINX --> USER[浏览器]
```

Node.js 只参与构建。Nginx 读取构建产物时不需要 Astro CLI，也不需要 `npm run dev` 或 `npm run preview`。

## 部署前先确认项目确实是静态输出

`astro.config.mjs` 应明确或默认使用静态模式：

```js
import { defineConfig } from "astro/config";

export default defineConfig({
  site: "https://zglab.fun",
  output: "static",
  build: {
    format: "directory",
  },
});
```

检查 `package.json`：

```json
{
  "scripts": {
    "build": "astro build",
    "check": "astro check"
  }
}
```

如果项目启用了需要 Node adapter 的 SSR 页面，就不能只复制 `dist/` 后让 Nginx 当普通静态目录使用。

## 安装服务器依赖

在 Ubuntu 服务器上执行：

```bash
sudo apt update
sudo apt install -y nginx rsync dnsutils
```

检查：

```bash
nginx -v
rsync --version
sudo systemctl status nginx --no-pager
```

安装软件会修改服务器状态。生产环境执行前，应确认当前 Nginx 是否承载其他站点，并备份已有 `/etc/nginx` 配置。

如果选择在服务器上构建，还需要安装与项目要求一致的 Node.js。不要只执行 `npm install -g astro`；Astro 应作为项目本地依赖，由 `npm ci` 使用锁文件安装。

## 选择在哪里构建

### 方案一：在开发机或 CI 构建

适合低配置服务器：

```bash
npm ci
npm run format:check
npm run check
npm run build
```

然后将 `dist/` 和部署脚本传到服务器的临时位置。示意命令：

```bash
rsync --archive --delete \
  dist/ \
  <server-user>@<server-ip>:/tmp/zglab-release/

scp scripts/deploy.sh \
  <server-user>@<server-ip>:/tmp/zglab-deploy.sh
```

登录服务器后执行：

```bash
chmod +x /tmp/zglab-deploy.sh
SOURCE_DIR=/tmp/zglab-release \
  /tmp/zglab-deploy.sh
```

上传前先检查目标路径。`rsync --delete` 会删除目标目录中源目录没有的文件，因此目标必须是专用临时发布目录，不能误写成包含其他数据的目录。

### 方案二：在服务器拉取源码并构建

适合服务器资源足够且已经安全配置私有仓库凭据的情况：

```bash
git clone <private-repository-url>
cd <repository-directory>
npm ci
npm run check
npm run build
./scripts/deploy.sh
```

Node 进程在构建完成后退出，不需要常驻。不过私有仓库认证必须使用服务器侧安全机制，不要把 GitHub Token 写入远程 URL、Shell 历史或 `.env.example`。

## 准备部署目录

首次部署前创建目录：

```bash
sudo mkdir -p /var/www/zglab.fun
sudo mkdir -p /var/backups/zglab.fun
```

不要为了省事执行：

```bash
sudo chmod -R 777 /var/www/zglab.fun
```

更合理的权限取决于发布用户和 Nginx worker 用户。可以让发布用户拥有写权限，同时保持 Nginx 只读。修改前先检查：

```bash
id
ps -eo user,group,comm | grep '[n]ginx'
namei -l /var/www/zglab.fun
```

权限方案必须结合服务器现有管理方式确定，本文不假设某个固定用户名。

## 配置 Nginx

创建站点配置：

```bash
sudoedit /etc/nginx/sites-available/zglab.fun
```

基础 HTTP 配置：

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name zglab.fun www.zglab.fun;

    root /var/www/zglab.fun;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~* \.(css|js|svg|webp|png|jpg|jpeg|ico)$ {
        expires 7d;
        add_header Cache-Control "public";
        try_files $uri =404;
    }
}
```

关键配置含义：

- `root` 指向 `dist/` 内容最终所在目录，不是源码目录；
- `index index.html` 让目录请求读取入口文件；
- `try_files $uri $uri/ =404` 先查同名文件，再查目录，均不存在时返回 404；
- 静态资源设置有限缓存，HTML 不在该正则中，避免入口页面长期缓存旧版本。

启用配置：

```bash
sudo ln -s /etc/nginx/sites-available/zglab.fun \
  /etc/nginx/sites-enabled/zglab.fun
```

如果链接已经存在，不要重复创建。可以先检查：

```bash
ls -l /etc/nginx/sites-enabled/
```

语法检查：

```bash
sudo nginx -t
```

只有 `nginx -t` 成功后才重载：

```bash
sudo systemctl reload nginx
```

重载通常比重启影响小，但仍然会改变线上配置。多站点服务器应在变更前保存原配置。

## 为什么 Astro 的目录格式与 `try_files` 匹配

当前构建使用：

```js
build: {
  format: 'directory',
}
```

因此 `/projects/` 对应：

```text
dist/projects/index.html
```

`try_files $uri $uri/ =404` 访问 `/projects/` 时会命中目录，再由 `index index.html` 返回页面。

如果配置只写：

```nginx
try_files $uri =404;
```

目录型路由可能无法按预期解析。

## 使用部署脚本发布

ZGLab 的 `scripts/deploy.sh` 默认读取：

```bash
SOURCE_DIR=<project-root>/dist
DEPLOY_DIR=/var/www/zglab.fun
BACKUP_ROOT=/var/backups/zglab.fun
```

正常执行：

```bash
./scripts/deploy.sh
```

自定义路径：

```bash
DEPLOY_DIR=/srv/www/zglab.fun \
BACKUP_ROOT=/srv/backups/zglab.fun \
./scripts/deploy.sh
```

脚本会：

1. 检查 `dist/` 是否存在；
2. 检查 rsync 是否可用；
3. 创建部署和备份目录；
4. 如果现有站点非空，先复制到时间戳备份目录；
5. 使用 `rsync --archive --delete` 同步新构建；
6. 任一步失败时返回非零状态。

脚本不会把密码写进文件，也不会删除最近备份。备份目录仍需由独立运维策略管理容量。

## 验证站点

先检查文件：

```bash
find /var/www/zglab.fun -maxdepth 2 -type f | sort | sed -n '1,80p'
```

在服务器本机检查 Nginx：

```bash
curl --noproxy '*' -I http://127.0.0.1/
curl --noproxy '*' -I \
  -H 'Host: zglab.fun' \
  http://127.0.0.1/projects/
```

再从外部设备检查：

```bash
curl -I http://<domain>/
curl -I http://<domain>/projects/
curl -I http://<domain>/research/
```

建议至少验证：

- 首页；
- 列表页；
- 一个动态生成的项目详情页；
- CSS 和头像资源；
- 不存在路径返回 404；
- 手机浏览器的导航与布局；
- 浏览器控制台没有资源加载错误。

## 域名与 HTTPS

域名需要通过 DNS A/AAAA 记录指向服务器公网地址。修改后可以检查：

```bash
dig +short <domain> A
dig +short <domain> AAAA
```

DNS 变更可能需要等待缓存更新。

公开站点应配置 HTTPS。证书申请方式与服务器现有网关、Nginx 配置和 DNS 环境有关。本次项目尚未执行真实证书申请，因此本文不把某组 Certbot 命令写成“已验证流程”。完成 TLS 后至少需要重新执行：

```bash
sudo nginx -t
curl -I https://<domain>/
```

并确认 HTTP 到 HTTPS 的跳转、证书域名和自动续期策略。

## 回滚

列出备份：

```bash
sudo ls -lah /var/backups/zglab.fun
```

回滚前先记录当前状态，并再次备份当前目录。随后选择明确的时间戳：

```bash
sudo rsync --archive --delete \
  /var/backups/zglab.fun/<timestamp>/ \
  /var/www/zglab.fun/
```

验证：

```bash
sudo nginx -t
curl --noproxy '*' -I \
  -H 'Host: zglab.fun' \
  http://127.0.0.1/
```

静态文件回滚通常不需要重载 Nginx，因为 Nginx 配置没有改变。但如果同时修改了站点配置，必须把配置回滚作为单独步骤处理。

## 常见问题

### Nginx 返回 403

优先检查：

- `root` 路径是否正确；
- `index.html` 是否存在；
- Nginx 用户是否能遍历父目录；
- 文件和目录权限是否可读。

```bash
namei -l /var/www/zglab.fun/index.html
sudo nginx -T | grep -nE "server_name|root|zglab"
```

### 首页正常，项目页 404

检查构建格式和 `try_files`：

```bash
find dist/projects -maxdepth 2 -type f
```

如果构建产物是目录格式，Nginx 需要检查 `$uri/`。

### 页面样式丢失

检查 HTML 中的资源路径和 Nginx 响应：

```bash
grep -n 'stylesheet' /var/www/zglab.fun/index.html
curl -I http://<domain>/_astro/<asset-name>.css
```

不要手工修改构建后的哈希资源名。

### 发布后仍看到旧页面

可能原因包括浏览器缓存、上游 CDN 缓存、反向代理缓存，或部署到了错误目录。先比较服务器文件时间和 HTML 内容，再使用无缓存请求验证：

```bash
curl -H 'Cache-Control: no-cache' http://<domain>/
```

### `rsync --delete` 删除了不应删除的文件

`--delete` 的目标是让部署目录与 `dist/` 完全一致，因此部署目录不能同时保存上传文件、日志或证书。业务数据必须放在站点目录之外。

## 本次已验证与待验证内容

已验证：

- Astro 静态构建成功；
- `dist/` 可以生成所有目标页面；
- 开发服务器路由返回 200；
- 部署脚本通过 `bash -n`；
- 缺少 `dist/` 时脚本返回非零状态；
- 临时目录中可以先备份旧文件，再同步新构建。

待在真实服务器验证：

- `/var/www` 与 `/var/backups` 的最终权限；
- 真实域名解析；
- Nginx 站点配置是否与其他站点冲突；
- HTTP/HTTPS 外部访问；
- 证书申请与自动续期；
- 真实备份容量和清理策略。

## 经验总结

Astro 静态网站的部署链路可以很简单，但“简单”不等于只执行一次复制。一个可维护的流程至少应包含：

1. 使用锁文件安装并完成构建检查；
2. 确认 `dist/` 是唯一发布源；
3. 让 Nginx 的路由策略与构建格式一致；
4. 覆盖前先备份；
5. 同步失败必须返回非零状态；
6. 用本机 Host 请求和外部域名分别验证；
7. 保留明确的回滚路径；
8. 将未验证的 DNS、权限和 TLS 状态如实记录。

相关笔记：

- [从一张静态页面到可维护的个人数字档案](from-zero-to-maintainable-astro-personal-site.md)
- [静态网站也需要工程化：备份发布、功能开关与动态接口边界](../knowledge/static-site-release-and-runtime-boundaries.md)

## 参考资料

- [Astro 部署指南](https://docs.astro.build/en/guides/deploy/)，访问于 2026-07-17。
- [Nginx ngx_http_core_module：root、index 与 try_files](https://nginx.org/en/docs/http/ngx_http_core_module.html)，访问于 2026-07-17。
- [rsync 官方手册](https://download.samba.org/pub/rsync/rsync.1)，访问于 2026-07-17。
