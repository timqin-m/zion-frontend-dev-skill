# Zion Backend Connect Skill

> **前端集成 AI Coding Skill，用于开发基于 Zion（[functorz.com](https://www.functorz.com)）作为后端即服务（BaaS）的自定义前端应用**

本仓库提供一个**前端集成 AI Coding Skill**，适用于 Cursor、Kimi CLI、Trae 等支持 Skill/MDC 协议的 AI 编码工具。后端协议细节（数据库 CRUD、Actionflow、AI Agent 等）请参考配合使用的 [`zion-baas-skill`](https://github.com/timqin-m/zion-baas-skill)。

> **注意**：本仓库适用于支持 Skill/MDC 协议的 AI 编码工具。不同工具的技能目录结构可能不同（如 `.cursor/skills/`、`.kimi/skills/` 等），请根据所用工具的要求放置。

## 🚀 快速安装

### 通过 skills.sh 安装（推荐）

```bash
npx skills add timqin-m/zion-backend-connect-skill
```

### 手动安装

1. 克隆或下载本仓库
2. 将 `zion-backend-connect-skills` 目录复制到你的项目根目录下的 `.cursor/skills/` 目录中
3. 确保目录结构为：`.cursor/skills/zion-backend-connect-skills/SKILL.md`

## 📦 包含的内容

这个 skill 专注于前端集成和实战避坑：

1. **Zion 后端架构** - GraphQL 端点、Apollo Client 配置、邮箱/用户名认证
2. **支付处理** - 支付宝集成，含实战坑点（表单提交、returnUrl、webhook 异步处理）
3. **开发最佳实践** - TypeScript/Apollo 版本锁定、安全规范（CRUD vs Actionflow）、构建检查清单
4. **UI 设计规则** - 基于有机/自然设计风格（Organic/Natural）的 React UI 设计系统
5. **Zeabur 部署规范** - SPA 部署、502 排查、`allowedHosts` 陷阱
6. **微信小程序开发** - 静默登录、文件上传坑点（MD5、ArrayBuffer、uploadHeaders）、Lottie、自定义 tabBar
7. **微信小程序支付** - 微信支付全流程，含分转元、JSON.parse 支付参数、异步查单

> 后端协议（数据库 CRUD、Actionflow、AI Agent、第三方 API、二进制上传）请参考 `zion-baas-skill`。

## 🎯 优势

- ✅ **专注前端集成**，避免与后端协议 skill 重复
- ✅ **实战坑点密集**，减少 AI 犯错概率
- ✅ **与 `zion-baas-skill` 完美配合**，分工清晰
- ✅ **无需 MCP 配置**，开箱即用

## 📚 使用示例

### 示例 1: 构建 Web 应用

**你**："基于 Zion 项目后端创建一个 WEB 应用，项目 exId 是 xxxx"

**AI 将**：

* 配置 Apollo Client（HTTP + WebSocket）
* 生成类型安全的 GraphQL 查询
* 创建博客/电商/工具类页面
* 实现实时更新（订阅）
* 按有机/自然风格设计 UI

### 示例 2: 构建微信小程序

**你**："基于 Zion 项目后端创建一个微信小程序，项目 exId 是 xxxx"

**AI 将**：

* 创建微信小程序项目结构（app.json、pages、components）
* 使用 `wx.request` 封装 GraphQL 请求
* 实现小程序页面和组件
* 集成微信静默登录
* 实现文件上传功能
* 配置小程序支付流程（如需要）

## 🛠️ 前置要求

* 支持 Skill/MDC 协议的 AI 编码工具（Cursor、Kimi CLI、Trae 等）
* Zion（functorz.com）账号和项目
* 配合使用 [`zion-baas-skill`](https://github.com/timqin-m/zion-baas-skill) 获取后端协议和 CLI 工具支持

## 📖 其他资源

### Zion 文档

* [Zion 官方文档](https://www.functorz.com/)

### GraphQL 资源

* [Apollo Client 文档](https://www.apollographql.com/docs/react/)
* [GraphQL 最佳实践](https://graphql.org/learn/best-practices/)

### AI 编码工具

* [Cursor 官方文档](https://cursor.sh/docs)
* [Kimi CLI](https://kimi.moonshot.cn/)

## 🎯 支持的开发场景

### Web 应用开发

* React + TypeScript + Vite 项目
* Apollo Client 集成
* 实时订阅（WebSocket）
* 支付宝支付集成
* Zeabur 平台部署

### 微信小程序开发

* 微信小程序原生开发
* CommonJS 模块系统
* wx.request 网络请求
* 微信支付集成
* 小程序文件上传

---

### 关于 Zion

Zion（functorz.com）是一款不用写代码也可以开发出微信小程序、WEB、AI Agent 的全栈无代码开发工具。Zion 拥有全栈无代码开发的能力，你可以在 Zion 上通过可视化的配置完成一个包含前后端的完整项目。作为一个高自由度的开发工具，Zion 的服务端接口也拥有标准的规范（包括不限于用户鉴权、支付服务、AI Agent服务、对象存储服务、数据库服务），结合用户在 Zion 上的自定义配置（数据库配置、AI Agent配置、支付配置）以及各项服务的规范，无需在Zion上搭建前端页面即可让 Cursor 结合 Zion 的后端开发一个可商用的完整软件产品！

---

**准备构建你的下一个应用？**

1. ⭐ Star 本仓库
2. 📋 通过 `npx skills add timqin-m/zion-backend-connect-skill` 安装本 skill
3. 📋 通过 `npx skills add timqin-m/zion-baas-skill` 安装后端协议 skill
4. 🚀 开始使用 AI 构建！

**有问题？** 提交 issue 或通过 Zion 平台联系

_最后更新：2026年4月16日_
