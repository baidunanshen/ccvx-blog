# CCVX 博客宝塔面板部署文档

> 适用项目：本项目当前的 VitePress 静态博客  
> 部署方式：GitHub Actions 自动构建，通过 SSH 和 rsync 更新宝塔静态目录  
> 文档验证日期：2026-10-27  
> 本地验证环境：Node.js 22.15.0、pnpm 10.28.2

## 1. 部署前须知

这是一个纯静态网站，不需要启动 Node.js 常驻服务，也不需要 PM2、Java、PHP 或数据库。

构建命令：

```text
pnpm build
```

最终发布目录：

```text
docs/.vitepress/dist
```

服务器目录中最终应当直接看到以下内容：

```text
index.html
404.html
about.html
assets/
pagefind/
sop/
favicon.ico
logo.png
robots.txt
```

不要把 `node_modules`、源码目录或整个项目直接放到网站根目录。

部署前还需要准备好：

1. 一台已安装宝塔面板和 Nginx 的 Linux 服务器。
2. 一个已完成 DNS 解析的域名，本文使用 `www.ccvx.top`。
3. 服务器安全组和宝塔防火墙中已放行 `80`、`443` 端口。
4. 如果服务器位于中国大陆，域名需要先完成 ICP 备案。
5. 如果服务器上已有网站，请先确认本次操作不会覆盖其他站点。

本文按“域名根路径部署”编写，即网站地址为 `https://www.ccvx.top/`。项目默认使用根路径 `/`，GitHub Actions 工作流会显式设置 `VITEPRESS_BASE=/`。

服务器上已有 `https://cjm.ccvx.top` 前端站点时，要为博客新建独立宝塔站点，不要把 `www.ccvx.top` 直接添加为 `cjm.ccvx.top` 站点的别名。两个站点可以使用不同的网站根目录和 Nginx 配置，同时共用服务器的 `80`、`443` 端口。

首次部署建议按以下顺序执行：

```text
创建宝塔站点 → 配置 DNS → 配置 Nginx → 配置 HTTPS
→ 手动上传一次构建产物 → 修正部署目录权限
→ 配置 GitHub Actions → 首次运行 → 上线验收
```

自动部署要求网站目录已经存在。GitHub Actions 配置见第 2 节，站点和首次内容配置见第 3 至第 8 节。

## 2. GitHub Actions 自动部署

仓库中的自动部署工作流：

```text
.github/workflows/deploy.yml
```

工作流在以下情况运行：

1. 向 `main` 分支推送代码。
2. 在 GitHub 的 Actions 页面手动点击 `Run workflow`。

执行过程为：

```text
检出代码 → 安装依赖 → 构建 → 校验配置 → SSH 预检 → rsync 上传 → 检查产物
```

工作流只把 `docs/.vitepress/dist` 中的内容同步到服务器，不会上传源码和 `node_modules`。目标目录和 SSH 配置会在上传前校验。

### 2.1 服务器一次性准备

服务器需要安装 `rsync`。Debian/Ubuntu 执行：

```bash
apt-get update
apt-get install -y rsync
```

CentOS/RHEL 执行：

```bash
yum install -y rsync
```

建议为 GitHub Actions 创建专用 SSH 用户，而不要使用服务器 root 密码：

```bash
useradd --create-home --shell /bin/bash blogdeploy
mkdir -p /home/blogdeploy/.ssh
chmod 700 /home/blogdeploy/.ssh
touch /home/blogdeploy/.ssh/authorized_keys
chmod 600 /home/blogdeploy/.ssh/authorized_keys
```

将 GitHub Actions 的公钥追加到 `/home/blogdeploy/.ssh/authorized_keys`，并确认文件最终所有者为 `blogdeploy`。

让部署用户可以写入网站目录，同时让 Nginx 用户 `www` 继续拥有读取权限。下面的命令仅针对已经确认过路径的博客目录执行：

```bash
chown -R blogdeploy:www /www/wwwroot/ccvx-blog
find /www/wwwroot/ccvx-blog -type d -exec chmod 755 {} \;
find /www/wwwroot/ccvx-blog -type f -exec chmod 644 {} \;
```

如果宝塔实际使用的 Web 用户不是 `www`，请通过 `id www` 或站点设置确认，并替换上述组名。

### 2.2 生成专用 SSH 密钥

在本地执行：

```powershell
ssh-keygen -t ed25519 -C "github-actions-baota-blog" -f "$HOME\.ssh\ccvx-blog-actions"
```

提示输入口令时直接按两次 Enter，生成无交互口令的部署密钥。随后把 `.pub` 公钥加入服务器的 `authorized_keys`，私钥只保存到 GitHub Actions Secret。

不要把私钥公钥放反，也不要把私钥提交到仓库。

### 2.3 配置 GitHub Actions Secrets

在 GitHub 仓库进入：

```text
Settings → Secrets and variables → Actions
```

在 `Secrets` 中新增：

| 名称 | 内容 |
| --- | --- |
| `DEPLOY_HOST` | 服务器公网 IP 或 SSH 域名 |
| `DEPLOY_PORT` | SSH 端口；未修改时填 `22` |
| `DEPLOY_USER` | 部署用户，例如 `blogdeploy` |
| `DEPLOY_SSH_PRIVATE_KEY` | 私钥文件的完整内容，包含 `BEGIN` 和 `END` 行 |
| `DEPLOY_KNOWN_HOSTS` | 服务器 SSH 主机公钥，生成方法见下文 |

`DEPLOY_KNOWN_HOSTS` 不要通过跳过主机校验来替代。可以在本地可信终端执行：

```bash
ssh-keyscan -p 22 your-server.example.com 2>/dev/null
```

把命令输出完整保存到 Secret。如果 SSH 端口不是 `22`，请把命令中的端口改为真实端口。首次配置时最好与云服务器控制台或服务器上 `ssh-keygen -lf` 显示的指纹核对一次。

在 `Variables` 中可选新增：

| 名称 | 用途 |
| --- | --- |
| `DEPLOY_PATH` | 网站目录，默认 `/www/wwwroot/ccvx-blog` |
| `DEPLOY_URL` | 生产地址，本项目填写 `https://www.ccvx.top`，用于部署后 HTTP 检查 |

`DEPLOY_PATH` 如果配置，必须严格使用 `/www/wwwroot/` 下的单层目录名，例如 `/www/wwwroot/ccvx-blog`。工作流会拒绝不符合规则的路径。

`DEPLOY_URL` 未配置时，工作流默认检查 `https://www.ccvx.top`。如果首次部署时 HTTPS 尚未配置，请先完成证书配置再运行工作流，或者暂时在仓库变量中把 `DEPLOY_URL` 覆盖为实际可访问的地址。

### 2.4 首次运行

配置好 Secrets 后，进入 GitHub Actions，选择 `Deploy Blog to Baota`，点击 `Run workflow`。

首次运行重点检查：

1. `Install dependencies` 与 `Build site` 成功。
2. `Check server target` 成功，没有 `Permission denied` 或 `Host key verification failed`。
3. `Upload static site` 成功，没有 `rsync: command not found`。
4. 工作流摘要显示部署成功。
5. 如果配置了 `DEPLOY_URL`，`Smoke test public URL` 成功。

首次运行成功后，后续只需推送 `main`，无需再手动执行构建或上传。

## 3. 备用方案：本机构建并手动上传

在项目根目录 `D:\Projects\Personal\博客` 执行：

```powershell
pnpm install --frozen-lockfile
pnpm build
```

构建成功后确认 `docs\.vitepress\dist\index.html` 存在。

随后将 `docs\.vitepress\dist` 目录中的全部内容上传到服务器。可以通过宝塔“文件”页面直接上传整个目录，也可以先压缩后上传。

如果使用 PowerShell 生成压缩包，可执行：

```powershell
$stamp = Get-Date -Format "yyyyMMdd-HHmmss"
$source = (Resolve-Path "docs\.vitepress\dist").Path
$archive = Join-Path (Get-Location) "blog-dist-$stamp.zip"
Compress-Archive -Path "$source\*" -DestinationPath $archive -Force
Write-Host "已生成：$archive"
```

上传和部署时请保留目录结构，尤其不要漏掉 `assets` 和 `pagefind` 两个目录。

## 4. 在宝塔创建网站

登录宝塔面板后：

1. 进入“网站”。
2. 点击“添加站点”。
3. 填写已经解析到服务器的域名 `www.ccvx.top`。
4. 网站根目录填写：

   ```text
   /www/wwwroot/ccvx-blog
   ```

5. 不创建数据库。
6. PHP 版本选择“纯静态”或“不设置 PHP”。
7. 如果面板提供 FTP，可跳过，后续直接通过文件管理上传。
8. 创建站点。

站点创建后，宝塔可能会在根目录放入默认的 `index.html`、`404.html` 等文件。上传本项目产物时，替换默认的 `index.html` 和 `404.html`。

上传完成后的正确目录结构是：

```text
/www/wwwroot/ccvx-blog/
├── index.html
├── 404.html
├── about.html
├── assets/
├── pagefind/
├── sop/
├── favicon.ico
├── logo.png
└── robots.txt
```

如果上传的是压缩包，请在宝塔文件管理器中把压缩包解压到 `/www/wwwroot/ccvx-blog`，并确保 `index.html` 直接位于该目录下，而不是多嵌套一层 `dist` 目录。

## 5. 配置域名与网站目录

在域名服务商处添加 DNS 记录。

如果使用独立子域名：

| 记录类型 | 主机记录 | 记录值 |
| --- | --- | --- |
| A | `www` | 服务器公网 IPv4 |

如果要让该域名直接作为主站，请根据实际情况配置 `@` 或 `www`。

回到宝塔：

1. 打开“网站”中的站点设置。
2. 确认“网站目录/运行目录”为 `/www/wwwroot/ccvx-blog`。
3. 确认站点未启用 PHP 或伪静态规则，静态博客由 Nginx 直接提供文件。
4. 如果服务器有系统防火墙或云厂商安全组，放行 TCP `80`、`443`；不要放行博客应用端口，因为该网站没有应用端口。

首次测试可以先访问：

```text
http://www.ccvx.top/
```

此时首页应能打开，但 HTTPS 和路由规则还需继续配置。

## 6. 配置 Nginx 路由和缓存

本项目生成的是 `.html` 页面，例如 `/about.html`、`/sop/more.html`。为了让访问 `/about` 时也能返回正确页面，并让默认 404 页面生效，需要调整 Nginx 配置。

在宝塔进入：

```text
网站 → 站点设置 → 配置文件
```

在站点的 `server { ... }` 中合并以下规则。只合并缺失或冲突的 `location`，不要重复放置同名规则，也不要覆盖宝塔自动生成的 SSL 配置。

```nginx
error_page 404 /404.html;

location = /404.html {
    internal;
}

location / {
    try_files $uri $uri.html $uri/ =404;
}

location ^~ /assets/ {
    try_files $uri =404;
    expires 1y;
    add_header Cache-Control "public, immutable";
}

location ~* \.(?:css|js|webp|png|jpe?g|gif|svg|ico|woff2?)$ {
    try_files $uri =404;
    expires 7d;
    add_header Cache-Control "public";
}
```

注意：

- `try_files $uri $uri.html $uri/ =404;` 用于支持直接访问 `/about` 这类无后缀地址。
- `assets` 中的文件名带构建哈希，可以设置一年缓存。
- 不要给 `index.html` 或 `pagefind` 设置长期强缓存，否则更新文章或搜索索引后，用户可能仍看到旧内容。
- 如果宝塔已经存在 `location /`，请修改原规则而不是再添加一条，否则 Nginx 可能报重复配置。

保存后，可以点击宝塔中的“重载配置”，或在 SSH 中执行：

```bash
nginx -t
systemctl reload nginx
```

只有 `nginx -t` 显示配置成功后，才执行重载。如果检测失败，先把配置改回修改前的版本，再根据报错处理。

## 7. 配置 HTTPS

推荐使用宝塔的免费 SSL：

1. 进入站点设置 → SSL。
2. 选择 Let's Encrypt。
3. 勾选域名后申请证书。
4. 证书签发成功后，开启“强制 HTTPS”。
5. 访问 `https://www.ccvx.top/` 验证。

申请证书前确认域名已经正确解析，并且 `80`、`443` 端口没有被安全组拦截。

如果使用云厂商证书，也可以在 SSL 页面选择“其他证书”，粘贴证书和私钥。证书生效后，确认浏览器地址栏没有证书警告，且页面没有 HTTP 混合内容。

## 8. 设置文件权限

自动部署时，网站目录需要允许 `DEPLOY_USER` 写入。如果部署用户是前面创建的 `blogdeploy`，执行：

```bash
chown -R blogdeploy:www /www/wwwroot/ccvx-blog
find /www/wwwroot/ccvx-blog -type d -exec chmod 755 {} \;
find /www/wwwroot/ccvx-blog -type f -exec chmod 644 {} \;
```

如果使用的是其他部署用户，把 `blogdeploy` 替换为该用户。不要为了省事设置 `777` 权限。

如果 `DEPLOY_USER` 本身就是 `www`，则使用 `chown -R www:www /www/wwwroot/ccvx-blog`。

GitHub Actions 上传时会设置目录 `755`、文件 `644`。如果宝塔实际使用的 Web 用户不是 `www`，请通过 `id www` 或站点设置确认，并调整目录所属组。

## 9. 上线验收

完成以上步骤后，按下面清单检查：

1. 打开 `https://www.ccvx.top/`，首页显示正常。
2. 浏览器开发者工具的 Network 中没有 `404`、CSS 加载失败或 JavaScript 加载失败。
3. 首页文章列表可以点击，文章页面可以正常打开。
4. 直接访问 `https://www.ccvx.top/about` 和 `/about.html`，两者都应返回关于页面。
5. 直接访问不存在的地址，例如 `/this-page-does-not-exist`，应显示项目自定义的 404 页面。
6. 点击主题自带的搜索，搜索结果可以正常显示。
7. 刷新文章页、移动端访问和深浅色模式切换正常。
8. 检查浏览器控制台没有明显的 JavaScript 报错。

也可以使用以下命令检查关键响应：

```bash
curl -I https://www.ccvx.top/
curl -I https://www.ccvx.top/about
curl -I https://www.ccvx.top/404.html
```

首页应返回 `200`。`/about` 在 Nginx 路由正确时应返回 `200`，而不是 `404`。

## 10. 后续更新文章

正常发布流程：

1. 本地修改 Markdown 或主题配置。
2. 可选：执行 `pnpm dev` 本地预览。
3. 提交并推送到 `main`：

   ```powershell
   git add docs
   git commit -m "docs: update blog"
   git push origin main
   ```

4. 打开 GitHub Actions，确认 `Deploy Blog to Baota` 运行成功。
5. 访问首页和一个文章页，确认更新已经生效。

推送后不需要登录宝塔上传文件。工作流会先备份服务器上的当前版本，再同步新的静态产物。静态文件更新不需要重启 Nginx；只有修改了 Nginx 配置，才需要重载 Nginx。

如果页面没有变化，先使用浏览器无痕窗口或强制刷新，排除浏览器缓存；再检查 GitHub Actions 是否成功，以及服务器上的 `index.html` 是否已经更新。

## 11. 可选：在服务器上构建

如果希望通过 Git 拉取源码并在服务器构建，需要先安装与项目兼容的 Node.js 和 pnpm。建议与本地验证版本保持一致：

```bash
node -v
corepack enable
corepack prepare pnpm@10.28.2 --activate
pnpm -v
```

如果服务器没有 Corepack，可使用：

```bash
npm install -g pnpm@10.28.2
```

将源码放到独立目录，例如 `/www/wwwroot/ccvx-blog-src`，不要直接放到网站根目录：

```bash
cd /www/wwwroot/ccvx-blog-src
pnpm install --frozen-lockfile
pnpm build
```

构建完成后，将 `docs/.vitepress/dist` 中的内容发布到 `/www/wwwroot/ccvx-blog`。发布前先备份旧产物，避免构建失败时影响正在访问的网站。

宝塔的 Node 项目管理器适合启动常驻服务，本项目不需要使用它，也不需要用 PM2 守护进程。

## 12. 常见问题

| 现象 | 常见原因 | 处理方式 |
| --- | --- | --- |
| 首页空白 | `index.html` 未上传，或资源目录缺失 | 确认根目录直接包含 `index.html`、`assets/`、`pagefind/` |
| CSS/JS 大量 404 | `base` 配置与部署路径不一致 | 宝塔根域名部署时确认工作流中的 `VITEPRESS_BASE` 为 `/` |
| `/about` 返回 404，但 `/about.html` 正常 | Nginx 没有配置 `$uri.html` 回退 | 使用本文第 6 节的 `try_files` 规则 |
| 任意页面都显示宝塔默认页 | 网站目录配置错误，或默认 `index.html` 未被替换 | 核对站点运行目录和根目录中的 `index.html` |
| 搜索不可用 | `pagefind` 目录未上传，或目录权限错误 | 上传 `pagefind/`，检查 Nginx Network 中的搜索请求 |
| 更新后仍是旧页面 | 浏览器、CDN 或缓存配置导致 | 强制刷新，检查文件时间；不要缓存 `index.html` |
| Actions 提示缺少 Secret | 仓库 Secrets 未配置或名称不一致 | 按第 2.3 节补齐五个必需 Secret |
| `Permission denied (publickey)` | 公钥未正确安装，或 SSH 用户/端口错误 | 检查 `authorized_keys`、用户 `.ssh` 权限和 `DEPLOY_PORT` |
| `Host key verification failed` | `DEPLOY_KNOWN_HOSTS` 不正确 | 重新执行 `ssh-keyscan`，核对主机指纹后更新 Secret |
| `rsync: command not found` | 服务器未安装 rsync | 按第 2.1 节安装 |
| `Permission denied` 发生在上传阶段 | 部署用户不能写入网站目录 | 按第 8 节修正目录所有者与权限 |
| 403 Forbidden | 文件权限或所有者不正确 | 按第 8 节修正目录权限，目录 `755`、文件 `644` |
| SSL 申请失败 | DNS 未生效、80/443 被拦截或域名有 CDN 冲突 | 检查解析、端口和 CDN 回源设置 |
| `nginx -t` 报重复 location | 配置文件中重复添加了同名指令 | 删除重复项，只保留一条 `location /` |

## 13. 回滚

GitHub Actions 每次上传前会在部署用户的 `$HOME/ccvx-blog-backups` 中生成备份，例如：

```text
/home/blogdeploy/ccvx-blog-backups/ccvx-blog-8b527d3c6a2f-20261027-153000.tar.gz
```

如果新版本出现页面错误：

1. 停止继续向 `main` 推送，并取消正在运行的部署工作流。
2. 选择最近一次确认正常的备份，先解压到独立回滚目录：

   ```bash
   rm -rf /home/blogdeploy/ccvx-rollback
   mkdir -p /home/blogdeploy/ccvx-rollback
   tar -xzf /home/blogdeploy/ccvx-blog-backups/<backup-file>.tar.gz \
     -C /home/blogdeploy/ccvx-rollback
   ```

3. 将回滚目录完整同步回网站目录，清理新版本特有文件：

   ```bash
   rsync -rlpt --delete-after --chmod=D755,F644 \
     --exclude=/.user.ini --exclude=/.htaccess \
     /home/blogdeploy/ccvx-rollback/ \
     /www/wwwroot/ccvx-blog/
   ```

4. 清理浏览器缓存后重新验收首页和文章页。

不要在没有确认备份可用时直接清空网站根目录。

## 14. 上线记录

部署完成后建议自行记录以下信息，密钥和证书私钥不要写入 Git：

| 项目 | 内容 |
| --- | --- |
| 生产域名 | `https://www.ccvx.top` |
| 服务器 IP |  |
| 网站根目录 | `/www/wwwroot/ccvx-blog` |
| Node.js 版本（仅服务器构建时） |  |
| pnpm 版本（仅服务器构建时） |  |
| SSL 到期时间 |  |
| 最近部署时间 |  |
| 最近部署版本/提交 |  |
| 备份位置 |  |

