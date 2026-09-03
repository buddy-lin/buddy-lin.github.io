# 简历网站部署指南 — 微信静默部署与 SSH 直连

> 目标：把林俊颖简历网站从 `buddy-lin.github.io` 迁到独立域名 `resumer.me`（海外服务器 + HTTPS），实现微信内静默打开、不再提示"非微信官方网页"。

> ⚠️ **安全说明**：本仓库(`buddy-lin.github.io`)为**公开仓库**。以下文档中的服务器 IP、登录密码均用 **占位符** 表示，真实值记录在项目本地 `.workbuddy/memory/` 中，请勿把真实凭据提交到公开仓库。

---

## 一、部署架构

| 环节 | 取值 |
|------|------|
| **域名** | `resumer.me`（注册商：阿里云，实名认证已通过，无 ClientHold） |
| **服务器** | UCloud（优刻得）轻量应用服务器 · 香港地域 · Ubuntu 24.04 LTS |
| **公网 IP** | `<服务器公网IP>` |
| **系统** | Ubuntu 24.04 LTS，1核2G，42MB 应用 |
| **防火墙** | 已放行 22 / 80 / 443 |
| **代码源** | GitHub 公开仓库 `buddy-lin/buddy-lin.github.io`（含 42MB assets、视频） |
| **登录方式** | 控制台 VNC + 公网 SSH（见第四节） |

---

## 二、部署步骤（已完成）

> 委托服务器直接从公开 GitHub 仓库 `git clone`，无需上传文件。

```bash
# 1. 安装 nginx + git
sudo apt install -y nginx git

# 2. 拉取简历网站到 /var/www/html
sudo rm -rf /var/www/html
sudo git clone https://github.com/buddy-lin/buddy-lin.github.io.git /var/www/html

# 3. 配置 Nginx site 指向 resumer.me
# 4. 启动 nginx
```

**配置的 Nginx server 块（/etc/nginx/sites-available/resume）：**
```nginx
server {
    listen 80;
    server_name resumer.me www.resumer.me;
    root /var/www/html;
    index index.html;
    location / { try_files $uri $uri/ =404; }
    location ~* \.(css|js|png|jpg|jpeg|gif|svg|mp4|webp)$ {
        expires 7d;
        add_header Cache-Control "public";
    }
}
```

---

## 三、当前状态

- ✅ 网站已通过 `http://<服务器公网IP>` 正常访问（HTTP 200），`<title>林俊颖 (Buddy) | 营销专家 ...</title>` 内容正确。
- ✅ 静态资源（图片、视频 `video_tool_intro.mp4`）均返回 200。
- ⏳ **待办**：
  1. 配置 HTTPS（`certbot --nginx`，微信静默的必要条件）
  2. 阿里云 DNS 解析：A 记录 `@`/`www` → `<服务器公网IP>`
  3. 验证 `https://resumer.me` 返回 200 + 微信静默打开

---

## 四、SSH 直连（进行中）

### 4.1 目标
让本地 WorkBuddy 能直接通过 SSH 登录服务器，实现后续部署全自动，无需用户在 VNC 手动操作。

### 4.2 已配置
在服务器端执行以下命令开启密码登录 + root 授权：

```bash
echo 'root:<root密码>' | sudo chpasswd
sudo usermod -aG sudo ubuntu
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

### 4.3 问题：外网 SSH 仍连不上
**现象**：`nc`/socket 测试 TCP 22 端口 OPEN，但读取 SSH banner 超时（`Error reading SSH protocol banner`），paramiko、`ssh -v`、旧算法均失败。

**根因分析**：TCP 22 能连但无 SSH 握手，说明 **ssh 服务未正常绑定公网接口**，或 UCloud 轻量服务器在 22 端口前有探活层拦截非 VNC 来源。常见原因：
- sshd `ListenAddress` 被限制为内网 IP（如 `10.7.80.64`）而非 `0.0.0.0`
- ufw / 安全组把 22 端口来源限制为特定 IP

### 4.4 排查 / 修复命令（待执行）
```bash
# 强制 sshd 监听所有接口 + 确保密码/root登录 + 放行22 + 重启 + 查看监听状态
sudo sed -i 's/^#\?ListenAddress.*/ListenAddress 0.0.0.0/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo ufw allow 22/tcp 2>/dev/null
sudo systemctl restart ssh
sudo ss -tlnp | grep ':22 '
echo "===SSH_FIX_DONE==="
```

> 观察 `ss -tlnp` 输出：若显示 `*:22` 或 `0.0.0.0:22` 且进程为 `sshd`，说明已监听公网，本地即可连入。

---

## 五、敏感信息（勿提交公开仓库）

| 项 | 真实值存放 |
|----|-----------|
| 服务器公网 IP | 项目本地 `.workbuddy/memory/2026-09-03.md` |
| root / ubuntu 密码 | 项目本地 `.workbuddy/memory/2026-09-03.md` |
| 阿里云 DNS 记录值 | 项目本地 `.workbuddy/memory/2026-09-03.md` |

> 请勿把这些真实值写入本文件的 git 提交版本，防止泄露。
