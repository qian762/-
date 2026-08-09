# Vercel 部署检查清单

## 部署前检查

- [ ] 已注册 GitHub 或 Gitee 账号
- [ ] 已注册 Vercel 账号
- [ ] 代码已推送到 GitHub/Gitee 仓库
- [ ] 已在 Vercel 导入仓库

## 环境变量配置

在 Vercel 项目设置 → Environment Variables 中添加：

- [ ] SUPABASE_URL - Supabase 项目 URL
- [ ] SUPABASE_ANON_KEY - Supabase 匿名密钥
- [ ] S3_ENDPOINT - 对象存储端点
- [ ] S3_ACCESS_KEY - 对象存储访问密钥
- [ ] S3_SECRET_KEY - 对象存储密钥
- [ ] S3_BUCKET - 对象存储桶名称

## 部署后验证

- [ ] 访问 Vercel 提供的网址，首页正常显示
- [ ] 点击分类，能进入风格列表
- [ ] 点击风格，能进入照片列表
- [ ] 能上传照片
- [ ] 能添加/编辑备注
- [ ] 能创建分享链接
- [ ] 分享链接能正常访问

## 获取环境变量

### Supabase
1. 登录 Supabase 控制台
2. 进入项目设置 → API
3. 复制 URL 和 anon/public key

### S3 对象存储
1. 在扣子平台查看对象存储配置
2. 复制端点、密钥和桶名称

---

## 部署失败排查

1. 查看 Vercel 部署日志
2. 检查环境变量是否正确
3. 检查 vercel.json 配置
4. 确保代码已正确推送
