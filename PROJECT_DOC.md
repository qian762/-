# 优品汇家居相册APP - 项目总文档

> **文档版本**：v1.0  
> **最后更新**：2025-08-08  
> **用途**：后续开发基准文档

---

## 一、项目概述

| 项目 | 说明 |
|------|------|
| 产品定位 | 优品汇家居相册与展示移动应用 |
| 核心场景 | 按分类 → 风格 → 照片三级结构组织家具照片，支持备注和分享 |
| 目标平台 | Android + iOS + Web（Expo 三端） |
| 部署方式 | 网页版，服务持续运行 |
| 用户体系 | **无登录注册，全局共享数据** |
| 照片存储 | Supabase Storage（云端对象存储） |
| 访问地址 | `https://ce487bdd-1bb7-46b9-a834-256675720a87.dev.coze.site` |

---

## 二、目录结构

```
/workspace/projects/
├── client/                          # React Native 前端
│   ├── app/                         # Expo Router 路由目录
│   │   ├── _layout.tsx              # 根布局（Stack 导航）
│   │   ├── (tabs)/                  # Tab 导航分组
│   │   │   ├── _layout.tsx          # Tab 布局配置
│   │   │   ├── index.tsx            # 首页 Tab → HomeScreen
│   │   │   ├── share.tsx            # 分享 Tab → ShareCenterScreen
│   │   │   └── profile.tsx          # 我的 Tab → ProfileScreen
│   │   ├── styles-list.tsx          # 风格列表页路由
│   │   ├── photos.tsx               # 照片详情页路由
│   │   ├── photo-fullscreen.tsx     # 全屏查看页路由
│   │   └── share-view.tsx           # 分享查看页路由（公开）
│   ├── screens/                     # 页面实现目录
│   │   ├── home/index.tsx           # 首页 - 分类列表
│   │   ├── styles-list/index.tsx    # 风格列表页
│   │   ├── photos/index.tsx         # 照片详情页（3列网格）
│   │   ├── photo-fullscreen/index.tsx # 全屏查看页
│   │   ├── share-center/index.tsx   # 分享中心页
│   │   ├── share-view/index.tsx     # 分享查看页（只读）
│   │   └── profile/index.tsx        # 个人中心页
│   ├── components/                  # 可复用组件
│   │   └── Screen.tsx               # 页面容器组件
│   ├── hooks/                       # 自定义 Hooks
│   │   └── useSafeRouter.ts         # 安全路由 Hook
│   ├── utils/                       # 工具函数
│   │   └── index.ts                 # createFormDataFile 等
│   ├── global.css                   # 主题配色（Tailwind CSS）
│   └── app.config.ts                # Expo 配置
├── server/                          # Express 后端
│   ├── src/
│   │   ├── index.ts                 # 服务入口
│   │   ├── routes/                  # 路由文件
│   │   │   ├── categories.ts        # 分类管理 API
│   │   │   ├── styles.ts            # 风格管理 API
│   │   │   ├── style-photos.ts      # 风格照片列表 API
│   │   │   ├── photo-manage.ts      # 照片上传/删除/备注 API
│   │   │   └── shares.ts            # 分享管理 API
│   │   └── storage/
│   │       ├── s3.ts                # S3 对象存储工具
│   │       └── database/
│   │           ├── shared/schema.ts # Drizzle ORM Schema
│   │           └── supabase-client.ts # Supabase 客户端
│   └── package.json
├── PRD.md                           # 需求文档
├── DESIGN.md                        # 设计风格文档
└── PROJECT_DOC.md                   # 本文档
```

---

## 三、数据库设计

### 3.1 表结构

#### categories（家具分类）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | serial | 主键 |
| name | varchar(100) | 分类名称 |
| icon | varchar(50) | 图标名称（Lucide Icon） |
| sort_order | integer | 排序序号 |
| is_preset | boolean | 是否预置分类 |
| created_at | timestamp | 创建时间 |

**预置数据**：
| 分类 | 图标 |
|------|------|
| 床 | Bed |
| 沙发 | Armchair |
| 衣柜 | Warehouse |
| 电视柜 | Tv |
| 桌子 | Table |

#### styles（风格）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | serial | 主键 |
| category_id | integer | 所属分类（FK → categories.id，级联删除） |
| name | varchar(100) | 风格名称 |
| created_at | timestamp | 创建时间 |

#### photos（照片）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | serial | 主键 |
| style_id | integer | 所属风格（FK → styles.id，级联删除） |
| image_url | text | **已废弃** - 原签名URL字段 |
| image_key | varchar(500) | Storage 中的存储路径 |
| thumbnail_key | varchar(500) | 缩略图存储路径 |
| note | text | 备注内容（可为空） |
| created_at | timestamp | 创建时间 |

#### shares（分享记录）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | serial | 主键 |
| share_token | varchar(64) | 分享链接唯一标识（UUID） |
| style_id | integer | 分享的风格（FK → styles.id，级联删除） |
| expired_at | timestamp | 过期时间（NULL 表示永久） |
| created_at | timestamp | 创建时间 |

### 3.2 数据关系

```
categories (1) ──→ (N) styles (1) ──→ (N) photos
                                      │
                                      └──→ shares (通过 style_id)
```

### 3.3 级联删除规则

- 删除分类 → 删除其下所有风格和照片
- 删除风格 → 删除其下所有照片和分享
- 删除照片 → 同步清理 Storage 中的原图和缩略图

---

## 四、API 接口

### 4.1 基础信息

| 项目 | 说明 |
|------|------|
| Base URL | `process.env.EXPO_PUBLIC_BACKEND_BASE_URL` |
| 路径前缀 | `/api/v1` |
| 数据格式 | JSON |
| 文件上传 | multipart/form-data |

### 4.2 分类管理

| 方法 | 路径 | 说明 | 参数 |
|------|------|------|------|
| GET | `/categories` | 获取所有分类 | - |
| POST | `/categories` | 创建分类 | Body: `{ name, icon? }` |
| DELETE | `/categories/:id` | 删除分类 | Path: `id` |

### 4.3 风格管理

| 方法 | 路径 | 说明 | 参数 |
|------|------|------|------|
| GET | `/categories/:categoryId/styles` | 获取分类下的风格 | Path: `categoryId` |
| POST | `/styles` | 创建风格 | Body: `{ name, category_id }` |
| DELETE | `/styles/:id` | 删除风格 | Path: `id` |

### 4.4 照片管理

| 方法 | 路径 | 说明 | 参数 |
|------|------|------|------|
| GET | `/styles/:styleId/photos` | 获取风格下的照片 | Path: `styleId` |
| POST | `/photos/upload` | 上传照片 | FormData: `image`, `style_id` |
| PATCH | `/photos/:id/note` | 更新备注 | Path: `id`, Body: `{ note }` |
| DELETE | `/photos/:id` | 删除照片 | Path: `id` |

**照片上传响应**：
```json
{
  "id": 1,
  "style_id": 1,
  "image_url": "https://...",
  "thumbnail_url": "https://...",
  "note": null,
  "created_at": "2025-08-08T..."
}
```

### 4.5 分享管理

| 方法 | 路径 | 说明 | 参数 |
|------|------|------|------|
| POST | `/shares` | 创建分享 | Body: `{ style_id, expired_days }` |
| GET | `/shares` | 获取所有分享 | - |
| DELETE | `/shares/:id` | 取消分享 | Path: `id` |
| GET | `/shares/:token` | 获取分享内容（公开） | Path: `token` |

**分享过期选项**：`1天` / `7天` / `30天` / `永久`

---

## 五、页面设计

### 5.1 导航结构

```
底部 Tab Bar（3个Tab）
├── 首页 (home)
│   └── 分类列表页
│       └── 风格列表页 (styles-list)
│           └── 照片详情页 (photos)
│               ├── 全屏查看页 (photo-fullscreen)
│               └── 分享面板 (Modal)
├── 分享 (share)
│   └── 分享中心页
│       └── 分享查看页 (share-view, 只读)
└── 我的 (profile)
    └── 个人中心页
        └── 分享管理列表
```

### 5.2 各页面功能

| 页面 | 文件路径 | 功能 |
|------|---------|------|
| 首页 | `screens/home/index.tsx` | 分类卡片展示、添加/删除分类 |
| 风格列表 | `screens/styles-list/index.tsx` | 风格列表、添加/删除风格 |
| 照片详情 | `screens/photos/index.tsx` | 3列网格、上传/删除/备注、编辑模式 |
| 全屏查看 | `screens/photo-fullscreen/index.tsx` | 全屏查看照片 |
| 分享中心 | `screens/share-center/index.tsx` | 输入分享链接查看 |
| 分享查看 | `screens/share-view/index.tsx` | 只读模式查看分享 |
| 个人中心 | `screens/profile/index.tsx` | 分享管理列表 |

### 5.3 交互规范

#### 删除交互

| 页面 | 交互方式 | 说明 |
|------|---------|------|
| 首页分类 | 右上角小图标 | 点击显示确认弹窗 |
| 风格列表 | 右上角小图标 | 点击显示确认弹窗 |
| 照片详情 | 编辑模式 | 点击「编辑」→ 选择照片 → 点击删除 |

#### 照片编辑模式（类似苹果相册）
1. 点击头部「编辑」按钮进入编辑模式
2. 每张照片右上角显示选择圆圈
3. 点击照片切换选中状态
4. 头部显示「已选 X 张」
5. 右侧显示红色删除按钮
6. 左上角「取消」退出编辑模式

#### 返回按钮
- 所有子页面都有自定义返回按钮
- 使用 `router.canGoBack()` 检查，fallback 到首页

---

## 六、UI 设计规范

### 6.1 风格定义：暖白居家风

**关键词**：温润、自然、居家、安心

**意象**：午后三点的阳光透过亚麻纱帘，洒在浅橡木色的地板上。空气中有淡淡的木质香气，手边是一杯温热的拿铁。

### 6.2 配色方案

| 用途 | 色值 | 意象来源 |
|------|------|----------|
| 主色 | `#8B6F47` | 胡桃木的温润棕 |
| 背景色 | `#FAF7F2` | 晨光中的米白墙面 |
| 卡片背景 | `#FFFFFF` | 干净的白纸 |
| 主要文字 | `#2C2420` | 深焙咖啡豆 |
| 次要文字 | `#8B8680` | 亚麻布的灰 |
| 边框/分割 | `#E8E0D8` | 原木的浅纹 |
| 强调/操作 | `#C4956A` | 拿铁奶泡的驼色 |
| 危险/删除 | `#D45D5D` | 陶土红（柔和警示） |
| 成功 | `#6B9E78` | 室内绿植的叶色 |

### 6.3 样式规范

| 元素 | 规范 |
|------|------|
| 卡片 | 白色背景 + 16px 圆角 + 暖色弥散阴影 |
| 阴影 | `rgba(139, 111, 71, 0.08)` 主色调阴影 |
| 按钮 | 主色填充 + 9999 圆角（胶囊形） |
| 输入框 | 浅灰背景 `#F5F0EB` + 12px 圆角，无边框 |
| 备注框 | 细边框 `#333` + 深灰文字 `#666` |
| 图标容器 | 主色 10% 透明度背景 + 12px 圆角 |

### 6.4 布局规范

| 场景 | 规范 |
|------|------|
| 首页分类 | 2列大卡片网格 |
| 照片网格 | 3列等宽不等高 |
| 容器内边距 | 16~20px |
| 卡片间距 | 12~16px |

### 6.5 设计禁忌

- ❌ 冷蓝色系（与居家温馨感冲突）
- ❌ 纯黑阴影（在暖色背景上显脏）
- ❌ 直角设计（破坏包裹感）
- ❌ 高饱和度颜色（打破安静氛围）
- ❌ 给照片加边框或装饰（照片是主角）

---

## 七、技术实现要点

### 7.1 图片处理

| 环节 | 实现 |
|------|------|
| 上传 | 前端选择 → 后端接收 → sharp 压缩 → 生成缩略图 → 上传 S3 |
| 存储 | 数据库存 `image_key` 和 `thumbnail_key`，动态生成签名 URL |
| 展示 | 网格用缩略图（~30KB），全屏用原图 |
| 缓存 | 使用 `expo-image` 组件，`cachePolicy="memory-disk"` |

### 7.2 数据刷新策略

```tsx
const needsRefreshRef = useRef(true);

useFocusEffect(
  useCallback(() => {
    if (needsRefreshRef.current) {
      fetchPhotos();
      needsRefreshRef.current = false;
    }
  }, [fetchPhotos])
);

// 上传/删除后标记需要刷新
needsRefreshRef.current = true;
```

### 7.3 Web 端兼容

| 问题 | 解决方案 |
|------|---------|
| Alert.alert 不工作 | 用自定义 Modal 替代确认弹窗 |
| onLongPress 不可靠 | 改用显式按钮触发 |
| 事件冒泡 | 添加 `e.stopPropagation()` |
| 触摸区域小 | 增大 `hitSlop` 和明确宽高 |

### 7.4 分享系统

| 功能 | 实现 |
|------|------|
| 生成链接 | UUID token + 过期时间 |
| 访问控制 | 公开接口，无需登录 |
| 只读模式 | 隐藏编辑按钮、备注输入 |
| 过期处理 | 检查 `expired_at`，过期显示提示 |

---

## 八、已知问题与待优化

### 8.1 当前已知问题

| 问题 | 状态 | 说明 |
|------|------|------|
| expo-image Web 兼容性 | 待观察 | 部分图片可能不显示 |
| 图片反复刷新 | 已优化 | 添加刷新标记，减少不必要的请求 |

### 8.2 后续优化建议

| 优先级 | 功能 | 说明 |
|--------|------|------|
| P1 | 图片懒加载 | 滚动到可视区域才加载 |
| P1 | 分页加载 | 照片数量多时分页 |
| P2 | 离线缓存 | PWA 离线访问支持 |
| P2 | 图片排序 | 支持拖拽排序 |
| P3 | 搜索功能 | 按备注内容搜索 |
| P3 | 批量操作 | 批量移动/复制照片 |

---

## 九、开发规范

### 9.1 代码规范

- 前端：TypeScript + ESLint
- 后端：TypeScript + ES Module
- 样式：Tailwind CSS (Uniwind)
- 路由：Expo Router

### 9.2 提交规范

```
feat: 新功能
fix: 修复 bug
perf: 性能优化
refactor: 重构
docs: 文档更新
```

### 9.3 测试检查

```bash
# 静态检查
pnpm -w lint:all

# 路由检查
app_check (工具)

# API 测试
curl http://localhost:9091/api/v1/health
```

---

## 十、附录

### 10.1 环境变量

| 变量 | 说明 |
|------|------|
| `EXPO_PUBLIC_BACKEND_BASE_URL` | 后端 API 地址 |
| `COZE_PROJECT_DOMAIN_DEFAULT` | 项目域名 |

### 10.2 依赖版本

| 依赖 | 版本 |
|------|------|
| Expo | 54 |
| React Native | 0.79 |
| Express | ^5 |
| Drizzle ORM | ^0.44 |
| sharp | ^0.34 |

### 10.3 相关文档

- [PRD.md](./PRD.md) - 需求文档
- [DESIGN.md](./DESIGN.md) - 设计风格文档
- [README.md](./README.md) - 项目说明
