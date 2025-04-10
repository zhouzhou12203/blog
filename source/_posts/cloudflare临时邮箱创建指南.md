---
title: cloudflare临时邮箱创建指南
katex: true
date: 2025-04-09 21:44:52
tags: cloudflare
categories: 网络研究
---
事先准备：
一个域名
[cloudflare](https://dash.cloudflare.com)账号
[github](https://github.com)账号:可以不使用，但是本文的部署方法是Github Action所以需要
[Resend](https://resend.com)网站：白嫖发件服务的
[官方临时邮箱文档](https://temp-mail-docs.awsl.uk)
[官方项目](https://github.com/dreamhunter2333/cloudflare_temp_email/tree/main)


# cloudflare操作
* #### 创建D1数据库
* #### 创建kv空间
* #### 创建前端页面
# Github操作
1.   fork项目
2.   打开仓库的 Actions 页面，找到 Deploy Backend Production 和 Deploy Frontend，点击 enable workflow 启用 workflow，这里并不是运行！！！
3.   然后在仓库页面 Settings -> Secrets and variables -> Actions -> Repository secrets, 添加以下 secrets:
* CLOUDFLARE_ACCOUNT_ID
* CLOUDFLARE_API_TOKEN
* BACKEND_TOML
```bash
#cloudflare创建后端works的名称
name = "cloudflare_temp_email"
main = "src/worker.ts"
compatibility_date = "2024-09-23"
compatibility_flags = [ "nodejs_compat" ]

# 如果你想使用自定义域名，你需要添加 routes 配置，这里的域名是后端api的域名
 routes = [
 { pattern = "mail-backend.zhou12203.top", custom_domain = true },
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
DOMAINS = ["yourdomain"]
# 这里需要一个密钥，打开[GitHub](https://www.librechat.ai/toolkit/creds_generator) 生成后复制“JWT_SECRET”里的内容
JWT_SECRET = "8ae9c3ed3af734449da54b4d2bf169d*************b73c9cee167ca98de2a"

# admin 控制台密码, 不配置则不允许访问控制台
ADMIN_PASSWORDS = ["your admin password"]

# 这里给了一个空数组，也就是说没有登录的用户没有可用的域名，如果你想给没有登录的用户使用域名，你可以加上自己的域名[“各自域名”] 它是一个数组也可以多个
DEFAULT_DOMAINS = [""]

# 是否允许自动回复邮件，不配置则不允许
ENABLE_AUTO_REPLY = false

# 是否允许用户创建邮件, 不配置则不允许
ENABLE_USER_CREATE_EMAIL = true

# 允许用户删除邮件, 不配置则不允许
ENABLE_USER_DELETE_EMAIL = true

# 允许自动回复邮件
ENABLE_AUTO_REPLY = true

# 默认发送邮件余额，如果不设置，将为 0
DEFAULT_SEND_BALANCE = 1

#你的resend api key，如果不在这里注明，就要在cloudflare后端works里添加
# RESEND_TOKEN = "re_9db4******KthokyniVNsC2fx6zNzR"

# admin 角色配置, 如果用户角色等于 ADMIN_USER_ROLE 则可以访问 admin 控制台
ADMIN_USER_ROLE = admin

# 是否启用 webhook
ENABLE_WEBHOOK = false
# 前端地址，用于发送 webhook 的邮件 url
FRONTEND_URL = https://xxxx.xxx


# 可以无限发送邮件的角色
NO_LIMIT_SEND_ROLE = admin,vip

#禁用匿名用户创建邮箱，如果设置为 true，则用户只能在登录后创建邮箱地址
DISABLE_ANONYMOUS_USER_CREATE_EMAIL = false

# 设置系统角色 vip , user , admin
ADMIN_USER_ROLE = [{"domains":["你的根域名"],"prefix":"","role":"vip"},{"domains":["你的根域名"],"prefix":"","role":"user"},{"domains":["你的根域名"],"prefix":"","role":"admin"}]

# 网站私有密码, 配置后需要密码才能访问
# PASSWORDS = ["123", "456"]



# D1 数据库的名称和 ID 可以在 cloudflare 控制台查看
[[d1_databases]]
binding = "DB"  #不要修改
database_name = "mail_temp" # 在cloudflare里面你创建的D1数据库名称
database_id = "c00e3*******c8f7-47f2939d714" # 在cloudflare里面你创建的D1数据库id

# kv config 用于用户注册发送邮件验证码，如果不启用用户注册或不启用注册验证，可以不配置
[[kv_namespaces]]
binding = "KV" #不要修改
id = "78b1e*********d4275647"  #在cloudflare里面找kv空间的id

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
* FRONTEND_NAME
* 可选内容（我并没有选用）
4.  打开仓库的 Actions 页面，找到 Deploy Backend Production 和 Deploy Frontend，点击 Run workflow 选择分支手动部署

# 配置邮件转发（cloudflare操作）
# 配置发送邮件（使用resend）
# Github Actions自动更新