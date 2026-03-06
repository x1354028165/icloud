# iCloud - Vue3云盘系统

一个现代化的Web文件管理系统，基于Vue3 + TypeScript构建，后端使用FileBrowser。

## 🎯 项目背景

为了替代传统的文件服务器界面，提供更好的用户体验和现代化的操作界面。

## 🏗 技术架构

### 前端
- **框架**: Vue 3 + TypeScript
- **构建工具**: Vite
- **样式**: TailwindCSS
- **图标**: Material Icons

### 后端
- **API服务**: FileBrowser (Go)
- **文件存储**: 本地文件系统
- **认证**: JWT Token

### 部署
- **Web服务器**: nginx
- **反向代理**: `/api/` → FileBrowser(8090)
- **前端托管**: `/cloud/` → Vue3静态文件

## ✨ 主要功能

- 🔐 用户登录认证
- 📁 文件/文件夹管理（创建、删除、重命名、移动）
- ⬆️ 文件上传（支持进度显示）
- 🔍 文件搜索
- 👁️ 文件预览（图片、视频、文本）
- 🗑️ 回收站功能
- 📤 文件分享
- 📱 响应式设计

## 🚀 本地开发

```bash
# 安装依赖
npm install

# 启动开发服务器
npm run dev

# 构建生产版本
npm run build
```

## 🌍 生产部署

### 手动部署
```bash
npm run build
cp -r dist/* /var/www/cloud-drive/
```

### 自动化部署（GitHub Actions）
推送到主分支将自动触发构建和部署流程。

## 🔧 配置说明

### 前端配置
- `vite.config.ts` - Vite构建配置
- API基础路径: `/api/`
- 应用基础路径: `/cloud/`

### nginx配置
```nginx
# 前端静态文件
location /cloud/ {
    alias /var/www/cloud-drive/;
    try_files $uri $uri/ /cloud/index.html;
}

# API代理
location /api/ {
    proxy_pass http://127.0.0.1:8090/api/;
}
```

### FileBrowser配置
- 端口: 8090
- 数据目录: `/root/filecloud/`
- 认证: JWT Token

## 📦 项目结构

```
src/
├── api/           # API接口封装
├── components/    # Vue组件
├── router/        # 路由配置
├── types/         # TypeScript类型定义
├── views/         # 页面组件
└── main.ts        # 应用入口
```

## 🔒 安全说明

- 不存储明文密码
- API请求使用JWT Token认证
- 文件访问权限由FileBrowser控制
- 敏感配置已排除在版本控制外

## 📄 许可证

MIT License

## 🤝 贡献

欢迎提交Issue和Pull Request。

---

**部署地址**: http://49.51.194.118/cloud/
**管理员**: Moss (薛鑫海)