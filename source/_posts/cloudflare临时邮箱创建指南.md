---
title: cloudflare临时邮箱创建指南
katex: true
date: 2025-04-09 21:44:52
tags: cloudflare｜域名邮箱
categories: 网络研究
---

###### 写在前面：因为cli和UI部署不方便更新，而且也有点过于繁琐，所以写了如何使用Github Action部署，在部署过程中也是晕倒了很多莫名其妙的报错，最终有了这份成功的经验
<!-- more -->

# 事先准备：
- 一个域名:假设是example.com，所有代码配置中的example.com替换成你的根域名
- [cloudflare](https://dash.cloudflare.com)账号
- [github](https://github.com)账号:可以不使用，但是本文的部署方法是Github Action所以需要
- [Resend](https://resend.com)网站：白嫖发件服务的,这里需要得到api填入后端变量
- [官方临时邮箱文档](https://temp-mail-docs.awsl.uk)
- [官方项目](https://github.com/dreamhunter2333/cloudflare_temp_email/tree/main)
- 参考：[【教程】小白也能看懂的自建Cloudflare临时邮箱教程（域名邮箱）](https://linux.do/t/topic/316819)


# cloudflare操作
#### 1. 创建D1数据库
- 进入cloudflare控制台，找到储存和数据库，展开看到D1 SQL 数据库
- 创建新的数据库，记录名称和ID
![image](https://zhouzhou12203.github.io/picx-images-hosting/image.39ld83nvj9.webp)
- 进入新的D1数据库-控制台，将下列代码块复制到输入框，点击**执行**
```bash
CREATE TABLE IF NOT EXISTS raw_mails (
    id INTEGER PRIMARY KEY,
    message_id TEXT,
    source TEXT,
    address TEXT,
    raw TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_raw_mails_address ON raw_mails(address);

CREATE TABLE IF NOT EXISTS address (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT UNIQUE,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_address_name ON address(name);

CREATE TABLE IF NOT EXISTS auto_reply_mails (
    id INTEGER PRIMARY KEY,
    source_prefix TEXT,
    name TEXT,
    address TEXT UNIQUE,
    subject TEXT,
    message TEXT,
    enabled INTEGER DEFAULT 1,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_auto_reply_mails_address ON auto_reply_mails(address);

CREATE TABLE IF NOT EXISTS address_sender (
    id INTEGER PRIMARY KEY,
    address TEXT UNIQUE,
    balance INTEGER DEFAULT 0,
    enabled INTEGER DEFAULT 1,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_address_sender_address ON address_sender(address);

CREATE TABLE IF NOT EXISTS sendbox (
    id INTEGER PRIMARY KEY,
    address TEXT,
    raw TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_sendbox_address ON sendbox(address);

CREATE TABLE IF NOT EXISTS settings (
    key TEXT PRIMARY KEY,
    value TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY,
    user_email TEXT UNIQUE NOT NULL,
    password TEXT NOT NULL,
    user_info TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_users_user_email ON users(user_email);

CREATE TABLE IF NOT EXISTS users_address (
    id INTEGER PRIMARY KEY,
    user_id INTEGER,
    address_id INTEGER UNIQUE,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_users_address_user_id ON users_address(user_id);

CREATE INDEX IF NOT EXISTS idx_users_address_address_id ON users_address(address_id);

CREATE TABLE IF NOT EXISTS user_roles (
    id INTEGER PRIMARY KEY,
    user_id INTEGER UNIQUE NOT NULL,
    role_text TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_user_roles_user_id ON user_roles(user_id);

CREATE TABLE IF NOT EXISTS user_passkeys (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    passkey_name TEXT NOT NULL,
    passkey_id TEXT NOT NULL,
    passkey TEXT NOT NULL,
    counter INTEGER DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_user_passkeys_user_id ON user_passkeys(user_id);

CREATE UNIQUE INDEX IF NOT EXISTS idx_user_passkeys_user_id_passkey_id ON user_passkeys(user_id, passkey_id);
```
![image](https://zhouzhou12203.github.io/picx-images-hosting/image.39ld83utla.webp)
#### 3. 创建kv空间
- 同样在储存和数据库-找到KV
- 创建新的KV空间，记录名称和ID
#### 4. [创建前端页面（先部署后端）](#advanced-syntax)
#### 5. [配置邮件转发（先部署后端）](#advanced)

# Github操作
#### 1. fork官方[Github](https://github.com/dreamhunter2333/cloudflare_temp_email/tree/main)项目
#### 2. 打开仓库的 Actions 页面，找到 Deploy Backend Production 和 Deploy Frontend，点击 enable workflow 启用 workflow，这里并不是运行！！！
#### 3. 然后在仓库页面 Settings -> Secrets and variables -> Actions -> Repository secrets, 添加以下 secrets:
- CLOUDFLARE_ACCOUNT_ID
    - Workers 和 Pages页面右侧复制
![image](https://zhouzhou12203.github.io/picx-images-hosting/image.1zig1vd56e.webp)
- CLOUDFLARE_API_TOKEN
    - Workers 和 Pages页面创建一个api
    - 你的cloudflare api，建议至少要有workers、D1、Pages、KV的权限
- BACKEND_TOML
    - 把下面的内容进行相应替换
```bash
name = "cloudflare_temp_email"
main = "src/worker.ts"
compatibility_date = "2024-09-23"
compatibility_flags = [ "nodejs_compat" ]

# 如果你想使用自定义域名，你需要添加 routes 配置，这里的域名是后端api的域名
 routes = [
 { pattern = "mail-backend.example.com", custom_domain = true },
]   

# 如果你想要使用定时任务清理邮件，取消下面的注释，并修改 cron 表达式
# [triggers]
# crons = [ "0 0 * * *" ]

# 通过 Cloudflare 发送邮件
# send_email = [
#    { name = "SEND_MAIL" },
# ]

[vars]
# 邮箱名称前缀，不需要后缀可配置为空字符串或者不配置
PREFIX = ""
# 用于临时邮箱的所有域名, 支持多个域名，你所持有的根域名
DOMAINS = ["example.com"]
JWT_SECRET = "8ae****************8de2a"

# admin 控制台密码, 不配置则不允许访问控制台
ADMIN_PASSWORDS = ["your password"]

# 这里给了一个空数组，也就是说没有登录的用户没有可用的域名，如果你想给没有登录的用户使用域名，你可以加上自己的域名[“各自域名”] 它是一个数组也可以多个
DEFAULT_DOMAINS = [""]

# 是否允许自动回复邮件，不配置则不允许
ENABLE_AUTO_REPLY = false

# 是否允许用户创建邮件, 不配置则不允许
ENABLE_USER_CREATE_EMAIL = true

# 允许用户删除邮件, 不配置则不允许
ENABLE_USER_DELETE_EMAIL = true

# 默认发送邮件余额，如果不设置，将为 0
DEFAULT_SEND_BALANCE = "1"

# 用于发件的api key,从resend网站获取,官方不建议在这里填写，需要手动在后端workers里配置:种类-纯文本、变量名-RESEND_TOKEN、值-re_9B1************sC2fx6zNzR，不需要""
RESEND_TOKEN = "re_9B1************sC2fx6zNzR"

# admin 角色配置, 如果用户角色等于 ADMIN_USER_ROLE 则可以访问 admin 控制台
ADMIN_USER_ROLE = "admin"


# 可以无限发送邮件的角色
NO_LIMIT_SEND_ROLE = "admin,vip"

# 禁用匿名用户创建邮箱，如果设置为 true，则用户只能在登录后创建邮箱地址
DISABLE_ANONYMOUS_USER_CREATE_EMAIL = true

# 设置系统角色 vip , user , admin
USER_ROLES = [
  { domains = ["example.com"], prefix = "", role = "vip" },
  { domains = ["example.com"], prefix = "", role = "user" },
  { domains = ["example.com"], prefix = "", role = "admin" }
]

# 网站私有密码, 配置后需要密码才能访问网站
# PASSWORDS = ["123", "456"]

# D1 数据库的名称和 ID 可以在 cloudflare 控制台查看
[[d1_databases]]
binding = "DB"
# D1 数据库名称
database_name = "mail_temp"
# D1 数据库 ID
database_id = "c0025c********************939d714" 

# kv config 用于用户注册发送邮件验证码，如果不启用用户注册或不启用注册验证，可以不配置
[[kv_namespaces]]
binding = "KV"
id = "78b******************75647"

# 新建地址限流配置 /api/new_address
# [[unsafe.bindings]]
# name = "RATE_LIMITER"  
# type = "ratelimit"
# namespace_id = "1001"
# # 10 requests per minute
# simple = { limit = 10, period = 60 }

# 绑定其他 worker 处理邮件，例如通过 auth-inbox ai 能力解析验证码或激活链接
# [[services]]
# binding = "AUTH_INBOX"
# service = "auth-inbox"

```
- FRONTEND_ENV
    - 把下面的VITE_API_BASE替换成你的后端域名
```bash
VITE_API_BASE=https://back-end.example.com
VITE_CF_WEB_ANALY_TOKEN=
VITE_IS_TELEGRAM=false
```
- FRONTEND_NAME
    - 你在cloudflare创建的前段pages项目名称（不是前端域名！！！）
- 可选内容（我并没有选用）
    - 略
#### 4. 打开仓库的 Actions 页面，找到 Deploy Backend和 Deploy Frontend，点击 Run workflow 选择分支手动部署

#### 5. Github Actions自动更新
- 打开仓库的 Actions 页面，找到 Upstream Sync，点击 enable workflow 启用 workflow
- 如果 Upstream Sync 运行失败，到仓库主页点击 Sync 手动同步即可
- 修改 Upstream Sync 的 schedule 配置可自定义更新间隔，参考 cron 表达式

<div id="advanced-syntax"></div>

# 创建前端页面
- 进入下列**链接直达**，在如下样式的输入框里输入你的后端地址(back-end.example.com)，之后会生成一个压缩包下载
![image](https://zhouzhou12203.github.io/picx-images-hosting/image.8l09swbovn.webp)
- 点击Workers和Pages，选择Pages，选择上传创建，文件选取刚才下载的文件
- [链接直达](https://temp-mail-docs.awsl.uk/zh/guide/ui/pages)
![image](https://zhouzhou12203.github.io/picx-images-hosting/image.7sneb5wd9g.webp)
<div id="advanced"></div>

# 配置邮件转发
- 在cloudflare的帐户主页，点进你所持有的根域名，找到新的页面侧边栏的的**电子邮件**选项
![image](https://zhouzhou12203.github.io/picx-images-hosting/image.3uv0uhn10z.webp)
- 电子邮件展开，点进电子邮件路由，切换到路由规则
- 启用Catch-all 地址并编辑
    - 操作：发送到worker
    - 目标位置：选择你的后端项目名称
    - 这个的作用是把所有@example.com的邮箱邮件转到后端，接收到域名邮箱里

- （额外选择）自定义地址，创建地址，这会创建单独的一个mail@example.com的转发规则，如果想要把发送到mail@example.com的邮件都转发到自己的其余邮箱，选择转发到电子邮件

    - 转发到的电子邮箱需要在目标地址里添加并验证
    - 经测试发现自定义地址的规则会覆盖Catch-all的规则，即如果Catch-all转发到workers，而mail@example.com设定转发到your-qq-number@qq.com，那么在域名邮箱里mail@example.com这个邮箱不会收到邮件，而是全部转发到your-qq-number@qq.com

# 配置发送邮件（使用resend）
- 进入[网站](https://resend.com)，注册账号（这里可以原汤化原食，直接用你刚设置好的域名邮箱注册
- 需要在Domins里绑定你的域名 example.com，它会让你添加解析记录，按照它的要求在cloudflare域名的侧边栏DNS里添加相应种类的解析
- 创建api key，这里的api key就是你在后端项目里要填的RESEND_TOKEN