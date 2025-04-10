---
title: cloudflare临时邮箱创建指南
katex: true
date: 2025-04-09 21:44:52
tags: cloudflare｜域名邮箱
categories: 网络研究
---
# 事先准备：
一个域名:假设是example.com
[cloudflare](https://dash.cloudflare.com)账号
[github](https://github.com)账号:可以不使用，但是本文的部署方法是Github Action所以需要
[Resend](https://resend.com)网站：白嫖发件服务的
[官方临时邮箱文档](https://temp-mail-docs.awsl.uk)
[官方项目](https://github.com/dreamhunter2333/cloudflare_temp_email/tree/main)


# cloudflare操作
* #### 创建D1数据库
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
* #### 创建kv空间
* #### 创建前端页面
[链接直达](https://temp-mail-docs.awsl.uk/zh/guide/ui/pages)
# Github操作
1.   fork官方[Github](https://github.com/dreamhunter2333/cloudflare_temp_email/tree/main)项目
2.   打开仓库的 Actions 页面，找到 Deploy Backend Production 和 Deploy Frontend，点击 enable workflow 启用 workflow，这里并不是运行！！！
3.   然后在仓库页面 Settings -> Secrets and variables -> Actions -> Repository secrets, 添加以下 secrets:
* CLOUDFLARE_ACCOUNT_ID
* CLOUDFLARE_API_TOKEN
* BACKEND_TOML
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

# 用于发件的api key,从resend网站获取
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
* FRONTEND_ENV
```bash
VITE_API_BASE=https://back-end.example.com
VITE_CF_WEB_ANALY_TOKEN=
VITE_IS_TELEGRAM=false
```
* FRONTEND_NAME
你在cloudflare创建的前段pages项目名称（不是前端域名！！！）
* 可选内容（我并没有选用）
4.  打开仓库的 Actions 页面，找到 Deploy Backend Production 和 Deploy Frontend，点击 Run workflow 选择分支手动部署

# 配置邮件转发（cloudflare操作）
# 配置发送邮件（使用resend）
# Github Actions自动更新