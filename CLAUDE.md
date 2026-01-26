# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

VulnArk 是一个现代化的漏洞管理平台，采用前后端分离架构：
- **后端**: Go (Gin 框架) + MySQL/GORM
- **前端**: Vue 3 + Element Plus + ECharts
- **部署**: Docker Compose

该平台提供漏洞管理、资产管理、知识库、漏洞库、自动扫描、CI/CD 集成、AI 风险评估等核心功能。

## 常用命令

### Docker 部署（推荐）

```bash
# 一键部署（交互式配置）
chmod +x deploy.sh
./deploy.sh

# 手动启动服务
docker-compose up -d --build

# 查看容器状态
docker-compose ps

# 查看日志
docker-compose logs -f backend   # 后端日志
docker-compose logs -f frontend  # 前端日志
docker-compose logs -f mysql     # 数据库日志

# 重启服务
docker-compose restart

# 停止服务
docker-compose stop

# 完全删除（包括数据卷）
docker-compose down -v
```

### 后端开发

```bash
cd backend

# 安装依赖
go mod download

# 开发环境运行（使用 config.yaml）
go run main.go

# 构建
go build -o vulnark main.go

# 创建管理员用户（如需手动创建）
mysql -u vulnark -p vulnark < create_admin.sql

# 清理测试数据
chmod +x clean_test_data.sh
./clean_test_data.sh
# 或直接使用 SQL
mysql -u vulnark -p vulnark < clean_test_data.sql
```

### 前端开发

```bash
cd frontend

# 安装依赖
npm install

# 开发服务器（热重载）
npm run serve

# 生产构建
npm run build

# 代码检查
npm run lint
```

## 架构设计

### 后端架构

后端采用标准的 MVC 分层架构：

```
backend/
├── cmd/                      # 命令行工具（如创建管理员）
├── config/                   # 配置文件
│   ├── config.yaml          # 本地开发配置
│   └── config.docker.yaml   # Docker 环境配置
├── controllers/             # 控制器层（处理 HTTP 请求）
│   ├── user_controller.go
│   ├── vulnerability_controller.go
│   ├── asset_controller.go
│   ├── vulnerability_assignment_controller.go
│   ├── knowledge_controller.go
│   ├── vulndb_controller.go
│   ├── scan_controller.go
│   ├── integration_controller.go
│   ├── dashboard_controller.go
│   ├── settings_controller.go
│   └── ai_controller.go
├── models/                  # 数据模型层（ORM 模型）
│   ├── user.go
│   ├── vulnerability.go
│   ├── vulnerability_assignment.go
│   ├── asset.go
│   ├── knowledge.go
│   ├── vulndb.go
│   ├── scan.go
│   ├── integration.go
│   └── settings.go
├── middleware/              # 中间件
│   ├── auth.go             # JWT 认证
│   ├── admin_auth.go       # 管理员权限
│   └── cors.go             # CORS 处理
├── routes/                  # 路由配置
│   └── routes.go
├── utils/                   # 工具函数
│   ├── database.go         # 数据库连接管理
│   ├── notification.go     # 通知服务（钉钉/飞书/企业微信/邮件）
│   ├── random.go
│   └── time.go
└── main.go                  # 应用入口
```

**关键设计点：**

1. **配置管理**: 使用 Viper 读取 YAML 配置，环境变量优先级高于配置文件
2. **数据库连接**: `utils/database.go` 实现连接重试机制（5 次重试，每次间隔 5 秒）
3. **路由分组**:
   - `/api/v1` - 公共路由（登录等）
   - `/api/v1` + JWT 中间件 - 需要认证的路由
   - `/api/v1/admin` + 管理员中间件 - 管理员专用路由
4. **自动迁移**: `main.go` 中的 `autoMigrateModels()` 在启动时自动创建/更新数据库表
5. **默认管理员**: 启动时自动创建默认管理员账户（用户名: admin, 密码: admin123）

### 前端架构

前端采用 Vue 3 Composition API + Vuex 状态管理：

```
frontend/src/
├── api/                     # API 封装层
│   ├── index.js            # 通用 HTTP 方法
│   ├── user.js
│   ├── vulnerability.js
│   ├── asset.js
│   ├── assignment.js
│   ├── knowledge.js
│   ├── vulndb.js
│   ├── scan.js
│   ├── integration.js
│   ├── dashboard.js
│   └── settings.js
├── components/              # 可复用组件
├── layout/                  # 布局组件
├── router/                  # 路由配置
│   └── index.js
├── store/                   # Vuex 状态管理
├── utils/                   # 工具函数
│   ├── auth.js             # 认证工具（token 管理）
│   └── request.js          # Axios 请求封装
├── views/                   # 页面组件
│   ├── Dashboard.vue
│   ├── Login.vue
│   ├── Profile.vue
│   ├── Settings.vue
│   ├── vulnerability/      # 漏洞管理相关页面
│   ├── assignment/         # 漏洞分配相关页面
│   ├── asset/              # 资产管理相关页面
│   ├── knowledge/          # 知识库相关页面
│   ├── vulndb/             # 漏洞库相关页面
│   ├── scan/               # 扫描管理相关页面
│   ├── integration/        # 集成管理相关页面
│   ├── user/               # 用户管理相关页面
│   └── error/              # 错误页面
├── App.vue
└── main.js                  # 应用入口
```

**关键设计点：**

1. **API 基础路径**: 使用 `/api` 相对路径，由 Nginx 反向代理到后端
2. **认证机制**: Token 存储在 Cookie 中，启动时自动恢复用户状态
3. **路由守卫**: 未登录自动跳转到登录页，保留重定向参数
4. **中文固定**: 移除了国际化，固定使用中文

### 数据库设计

核心实体关系：
- **User** (用户) - 系统用户，包含角色（管理员/普通用户）
- **Vulnerability** (漏洞) - 漏洞信息，关联资产和分配记录
- **VulnerabilityAssignment** (漏洞分配) - 漏洞分配给用户的记录
- **VulnerabilityAssignmentHistory** (分配历史) - 漏洞分配状态变更历史
- **Asset** (资产) - 组织资产，关联漏洞
- **Knowledge** (知识库) - 安全知识和修复指南
- **VulnDB** (漏洞库) - 已知漏洞数据库（CVE 等）
- **ScanTask** (扫描任务) - 自动化扫描任务配置
- **ScanResult** (扫描结果) - 扫描任务的结果
- **CIIntegration** (CI/CD 集成) - CI/CD 集成配置
- **IntegrationHistory** (集成历史) - CI/CD 集成调用历史
- **Settings** (系统设置) - 系统配置（JSON 字段存储）

## 配置说明

### 环境变量优先级

后端配置读取顺序：
1. 环境变量（Docker 部署时使用）
2. 配置文件（本地开发时使用）

**关键环境变量:**
- `DB_HOST` - 数据库主机（默认: mysql）
- `DB_PORT` - 数据库端口（默认: 3306）
- `DB_USER` - 数据库用户名
- `DB_PASSWORD` - 数据库密码
- `DB_NAME` - 数据库名称
- `SERVER_HOST` - 后端监听地址（默认: 0.0.0.0）
- `SERVER_PORT` - 后端监听端口（默认: 8080）

### 配置文件位置

- 本地开发: `backend/config/config.yaml`
- Docker 部署: `backend/config/config.docker.yaml`（由 deploy.sh 处理）

## 漏洞复测功能

v0.1.3 版本新增的核心功能：

1. **复测申请**: 被分配人修复漏洞后可以提交复测申请
2. **状态管理**: 漏洞状态包括「待复测」状态
3. **通知系统**: 管理员收到复测申请的实时通知

相关文件：
- `controllers/vulnerability_assignment_controller.go` - 分配和状态更新逻辑
- `models/vulnerability_assignment.go` - 分配模型

## 已知问题修复记录

### v0.1.2 修复的问题
- 漏洞分配时的字段名不匹配（前后端字段命名规范统一）
- 日期格式解析错误（修改为 ISO 标准格式）

### v0.1.1 修复的问题
- Docker 环境下 MySQL 连接失败（添加重试机制）
- 配置文件路径不匹配（环境变量优先级）
- 数据库连接配置读取错误

## 开发注意事项

1. **数据库连接**: Docker 环境中后端需等待 MySQL 完全启动，已实现重试机制
2. **默认管理员**: 首次启动自动创建 admin/admin123，生产环境需立即修改
3. **前后端字段命名**: 遵循统一的命名规范，后端使用下划线，前端驼峰式（通过 JSON tag 映射）
4. **日期格式**: 统一使用 ISO 8601 格式
5. **CORS 配置**: 开发环境允许所有来源，生产环境需要限制
6. **通知服务**: 支持钉钉、飞书、企业微信、邮件多种通知方式，在系统设置中配置
7. **用户密码**: User 模型的 `BeforeCreate` 钩子自动加密密码，不要手动调用加密函数

## 访问地址

Docker 部署后的默认访问地址：
- 前端: http://localhost
- 后端 API: http://localhost/api 或 http://localhost:8080
- MySQL: localhost:3306
