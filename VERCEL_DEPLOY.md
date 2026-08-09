# Vercel 部署指南

## 第一步：准备账号

1. 注册 GitHub 或 Gitee 账号
2. 注册 Vercel 账号（vercel.com，用 GitHub/Gitee 登录）

## 第二步：推送代码

### 使用 Gitee（推荐，国内访问快）

```bash
# 1. 在 Gitee 创建新仓库（如 furniture-album）

# 2. 在项目根目录执行
git init
git add .
git commit -m "初始提交"
git branch -M main
git remote add origin https://gitee.com/你的用户名/furniture-album.git
git push -u origin main
```

### 使用 GitHub

```bash
# 1. 在 GitHub 创建新仓库（如 furniture-album）

# 2. 在项目根目录执行
git init
git add .
git commit -m "初始提交"
git branch -M main
git remote add origin https://github.com/你的用户名/furniture-album.git
git push -u origin main
```

## 第三步：在 Vercel 部署

1. 访问 vercel.com，登录账号
2. 点击「Add New Project」
3. 导入你的仓库（GitHub 或 Gitee）
4. 配置项目：
   - Framework Preset: 选择 `Other`
   - Root Directory: `./`
   - Build Command: 留空（使用 vercel.json 配置）
   - Output Directory: 留空
5. 添加环境变量（点击 Environment Variables）：

| 变量名 | 值 | 说明 |
|--------|-----|------|
| SUPABASE_URL | 你的 Supabase URL | 从 Supabase 项目设置获取 |
| SUPABASE_ANON_KEY | 你的 Supabase Anon Key | 从 Supabase 项目设置获取 |
| S3_ENDPOINT | S3 端点地址 | 从扣子平台对象存储获取 |
| S3_ACCESS_KEY | S3 访问密钥 | 从扣子平台对象存储获取 |
| S3_SECRET_KEY | S3 密钥 | 从扣子平台对象存储获取 |
| S3_BUCKET | S3 存储桶名称 | 从扣子平台对象存储获取 |

6. 点击「Deploy」

## 第四步：获取访问地址

部署成功后，Vercel 会提供一个永久访问地址：
- 格式：`https://你的项目名.vercel.app`
- 可以自定义域名

## 第五步：添加到手机桌面

1. 用手机浏览器打开 Vercel 提供的地址
2. iOS：点击分享 → 添加到主屏幕
3. Android：点击菜单 → 添加到桌面

---

## 常见问题

### Q: 部署失败怎么办？
A: 检查 Vercel 部署日志，通常是环境变量配置问题。

### Q: 前端页面空白？
A: 检查 API 地址是否正确，Web 环境使用相对路径 `/api/v1`。

### Q: 图片无法上传？
A: 检查 S3 环境变量是否正确配置。

### Q: 数据库连接失败？
A: 检查 SUPABASE_URL 和 SUPABASE_ANON_KEY 是否正确。

### Q: 访问 API 返回 404？
A: 确保 vercel.json 中的路由配置正确，API 路径以 `/api/v1` 开头。

---

## 项目结构

```
├── vercel.json          # Vercel 配置
├── VERCEL_DEPLOY.md     # 部署指南
├── .gitignore           # Git 忽略文件
── client/              # 前端代码（Expo）
│   ├── app/             # 路由
│   ├── screens/         # 页面
│   ├── utils/
│   │   └── api-config.ts  # API 地址配置
│   ── package.json     # 含 build:web 脚本
└── server/              # 后端代码（Express）
    ├── src/
    │   ├── index.ts     # 入口（兼容 Vercel）
    │   ├── routes/      # API 路由
    │   └── storage/     # 数据库和存储
    └── package.json
```

## 技术说明

- **前端**：Expo Web 导出为静态文件，由 Vercel 托管
- **后端**：Express 应用转换为 Vercel Serverless 函数
- **API 路由**：所有 `/api/v1/*` 请求转发到后端
- **静态文件**：其他请求由前端静态文件处理
- **环境变量**：Supabase 和 S3 配置在 Vercel 中设置
