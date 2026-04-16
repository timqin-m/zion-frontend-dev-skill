---
name: zion-frontend-dev-skill
description: "Complete integration guide for building custom frontend applications with Zion.app backend-as-a-service. Use when: (1) Building React/TypeScript web apps with Zion backend, (2) Developing WeChat Mini Programs with Zion backend, (3) Setting up GraphQL clients (Apollo Client), (4) Working with database operations, Actionflows, AI Agents, payments, file uploads, (5) Following UI design rules (Organic/Natural style), (6) Deploying to Zeabur platform"
---

# Zion Frontend Dev Skill

Frontend development guide for building custom applications with Zion.app ([functorz.com](https://www.functorz.com)) as a backend-as-a-service (BaaS).

> **Skill Relationship Note**: This skill works alongside `zion-baas-skill`.
> - Use `zion-baas-skill` for: developer authentication, CLI tools, Meta API queries, database CRUD rules, Actionflow protocols, AI Agent protocols, third-party API rules, and binary asset upload details.
> - Use this skill for: frontend framework integration (React, Apollo, WeChat Mini Program), UI design rules, deployment guides, payment processing, and **practical pitfalls** that require extra clarity to avoid AI errors.

## Table of Contents

1. [Zion Frontend Architecture](#1-zion-frontend-architecture)
2. [Payment Processing](#2-payment-processing)
3. [Development Best Practices](#3-development-best-practices)
4. [UI Design Rules](#4-ui-design-rules)
5. [Zeabur Deployment](#5-zeabur-deployment)
6. [WeChat Mini Program](#6-wechat-mini-program)
7. [WeChat Mini Program Payment](#7-wechat-mini-program-payment)

---

# 1. Zion Frontend Architecture

Zion is a full-stack no-code development platform, but its backend architecture is designed to be used headlessly. This allows building completely custom frontend applications while leveraging Zion as a pure backend-as-a-service (BaaS).

## Core Architecture

* **Database**: A powerful, enterprise-grade relational database built on PostgreSQL.
* **Actionflow**: For building custom workflows and automations.
* **Third-party API**: Imported third-party HTTP API definitions, the server acts as a relay.
* **AI Agent**: AI agent builder / runtime capable of RAG, tool use, multi-modal input/output, structured JSON output.
* **GraphQL**: All backend interactions are exposed through a single, unified GraphQL API.
  - **HTTP URL**: `https://zion-app.functorz.com/zero/{projectExId}/api/graphql-v2`
  - **WebSocket URL**: `wss://zion-app.functorz.com/zero/{projectExId}/api/graphql-subscription`

> For detailed backend protocols, refer to `zion-baas-skill`.

## Communicating with the backend

When using Typescript, use Apollo GraphQL + subscriptions-transport-ws.

### Reference Implementation
```typescript
import { ApolloClient, InMemoryCache, HttpLink, split } from '@apollo/client';
import { getMainDefinition } from '@apollo/client/utilities';
import { WebSocketLink } from '@apollo/client/link/ws';
import { SubscriptionClient } from 'subscriptions-transport-ws';

const httpUrl = 'https://zion-app.functorz.com/zero/{projectExId}/api/graphql-v2';
const wssUrl = 'wss://zion-app.functorz.com/zero/{projectExId}/api/graphql-subscription';

export const createApolloClient = (token?: string) => {
  const wsClient = new SubscriptionClient(wssUrl, {
    reconnect: true,
    connectionParams: token ? {
      authToken: token // Anonymous users have no token, connectionParams must be empty.
    } : {},
  });

  const wsLink = new WebSocketLink(wsClient);

  const splitLink = split(
    ({ query }) => {
      const definition = getMainDefinition(query);
      return (
        definition.kind === 'OperationDefinition' &&
        definition.operation === 'subscription'
      );
    },
    wsLink,
    new HttpLink({
      uri: httpUrl,
      headers: token ? { Authorization: `Bearer ${token}` } : {},
    })
  );

  return new ApolloClient({
    link: splitLink,
    cache: new InMemoryCache(),
  });
};
```

## Authentication

All requests to the GraphQL endpoint are either authenticated, or they will be assigned an anonymous user role.
**When user changes authentication status (logging in/out), WebSocket connection should be re-established.**

### Email with verification
1. Send verification code first:
```graphql
mutation SendVerificationCodeToEmail(
    $email: String!
    $verificationEnumType: verificationEnumType!
) {
    sendVerificationCodeToEmail(
    email: $email
    verificationEnumType: $verificationEnumType
    )
}
```
2. Register/login:
```graphql
mutation AuthenticateWithEmail(
    $email: String!
    $password: String!
    $verificationCode: String
    $register: Boolean!
) {
    authenticateWithEmail(
    email: $email
    password: $password
    verificationCode: $verificationCode
    register: $register
    ) {
        account {
            id
            permissionRoles
        }
        jwt {
            token
        }
    }
}
```

### Username and password
```graphql
mutation AuthenticateWithUsername(
  $username: String!
  $password: String!
  $register: Boolean!
) {
  authenticateWithUsername(
    username: $username
    password: $password
    register: $register
  ) {
    account {
      id
      permissionRoles
    }
    jwt {
      token
    }
  }
}
```

N.B. both authentication mutations return `FZ_Account` type, which only has: email, id, permissionRoles, phoneNumber, profileImageUrl, roles, username.

## Interacting with the GraphQL API

The GraphQL API is automatically generated by Zion.app depending on the structure of the backend. Long and bigint are sometimes used to represent corresponding fields of similar types, but the exact type must be chosen depending on the type involved in the query. **Inputs of Json type must be passed in as whole in variables.**

```graphql
mutation CreateOrder($args: Json!) {
  fz_invoke_action_flow(
    actionFlowId: "7e93e65e-7730-470c-b2fd-9ff608cb68e8"
    versionId: 5
    args: $args
  )
}
```
```json
{
  "variables": {
    "course_id": 2
  }
}
```
**AVOID assembling args inside the query.**

> For database CRUD rules, Actionflow protocols, AI Agent protocols, and third-party API rules, refer to `zion-baas-skill`.

### Subscriptions

For real-time functionality, the GraphQL API supports subscriptions.

Once Websocket is established, the initial `connection_init` message will be acknowledged:
```json
{
  "id": null,
  "type": "connection_ack",
  "payload": null
}
```

Example subscription message:
```json
{
  "id": "some id, unique per websocket",
  "type": "start",
  "payload": {
    "operationName": "OperationName",
    "query": "subscription OperationName($arg0: String!) { account(where: { username: { _eq: $arg0 } }) { __typename id username } }",
    "variables": { "arg0": "someName" }
  }
}
```

## File / Binary Asset Handling

Media and other files are not stored in the PostgreSQL database but in a dedicated Object Storage system.
- When using them as input / parameter in other parts of the system, **always use the corresponding id instead**. URLs cannot be used as inputs in place of a file / image / video.
- When using a media file on the frontend, make sure to fetch its `url` subfield.

> For detailed upload protocol, refer to `zion-baas-skill`.

## Permission

All GraphQL fields have permission control based on the current logged in user's role(s). If an attempted access violates permission policies, an error will be given within the GraphQL response. The key to look for here is the **403 error code**.

```json
{
  "errors": [
    {
      "errorCode": 403,
      "extensions": { "classification": "ACTION_FLOW" },
      "message": "Anonymous user has no permission on invocation of action flow: 63734821-319d-4f00-a5cf-69f134b42b9c",
      "operation": "fz_invoke_action_flow"
    }
  ]
}
```

---

# 2. Payment Processing

Zion 支持原生支付集成，使得基于 Zion 构建的项目的最终用户可以使用支付宝、微信支付等方式支付订单。

## 支付宝支付

### 前置条件
1. **选择订单表**：在 Zion 编辑器内将某个表绑定为项目的"订单表"设置。每个支付都必须关联一个订单 ID。
2. **配置支付密钥和证书**：在 Zion 编辑器内配置支付宝的支付密钥和证书。

### 订单创建
每个项目的概念订单表都有自己的结构，不一定需要命名为 "order"。通常应包含：订单属于哪个账户、订单金额、订单包含哪些商品（通常通过 1:n 关系）。

**重要说明**：
- 必须使用创建订单 ActionFlow 输出的订单 ID 和 `goods_name`（不要有自定义订单逻辑）
- 调用 Zion 提供的 `alipayTrade` 接口时，确保请求包含 Bearer token
- `returnUrl` **必须**使用 `window.location.origin`（根域名），不要使用固定路径（如 `/payment/callback`）
- `totalAmount` 在前端以 `Float` 类型传递，后端会将其转换为字符串格式传给支付宝（支付宝要求 `total_amount` 必须是字符串类型，如 `"88.88"`）

### 创建支付宝支付

```gql
mutation AliPayTrade(
  $outTradeNo: String!
  $productCode: String!
  $subject: String!
  $totalAmount: Float!
  $returnUrl: String!
) {
  alipayTrade(
    bizContent: {
      out_trade_no: $outTradeNo
      product_code: $productCode
      subject: $subject
      total_amount: $totalAmount
    }
    returnUrl: $returnUrl
  )
}
```

变量示例：
```json
{
  "outTradeNo": "{orderId}",
  "productCode": "FAST_INSTANT_TRADE_PAY",
  "subject": "{orderSubject}",
  "totalAmount": 0.01,
  "returnUrl": "{window.location.origin}"
}
```

输出示例：
```json
{
  "data": {
    "alipayTrade": [
      "<form>...</form>",
      "{tradeNo}"
    ]
  }
}
```

该 mutation 返回一个包含两个元素的数组：
- 第一个元素：支付宝支付表单 HTML 字符串，需要提交该表单以跳转到支付宝支付页面
- 第二个元素：支付宝交易号（可选，仅供参考，不应依赖）
  - 根据支付宝规范，`alipay.trade.page.pay` 首次响应不应包含支付宝交易号
  - 实际的支付宝交易号应通过异步通知或主动查询接口获取

### 跳转到支付宝支付页面

**⚠️ 关键警告：表单提交的正确方式**

支付宝返回的 HTML 表单包含已经格式化好的字段值，特别是 `biz_content` 字段包含 JSON 字符串。**绝对不要重新构建表单**，否则会导致字段值被错误转义，从而引发签名验证失败。

**正确做法：直接使用原始 HTML 表单（在当前页面打开）**

```typescript
const formMatch = alipayFormHtml.match(/<form[^>]*>[\s\S]*?<\/form>/i);
const scriptMatch = alipayFormHtml.match(/<script[^>]*>[\s\S]*?<\/script>/i);

const extractedFormHtml = formMatch?.[0] || '';
const extractedScriptHtml = scriptMatch?.[0] || '';

const completeHtml = `
  <html>
    <head>
      <meta charset="utf-8">
      <title>正在跳转到支付宝...</title>
    </head>
    <body>
      ${extractedFormHtml}
      ${extractedScriptHtml}
    </body>
  </html>
`;

document.open();
document.write(completeHtml);
document.close();
```

### 支付返回处理

当用户完成支付后，支付宝会将用户重定向到 `returnUrl`。在该页面中需要：
1. 检查 URL 参数中是否包含 `out_trade_no`
2. 查询订单状态，确认支付是否完成
3. 更新用户权限
4. 显示相应提示信息
5. 页面跳转

**实现示例**：

```typescript
useEffect(() => {
  const params = new URLSearchParams(window.location.search);
  const outTradeNo = params.get('out_trade_no');
  
  if (outTradeNo) {
    checkOrderAndUpdateAccess(outTradeNo);
  }
}, []);

const checkOrderAndUpdateAccess = async (orderId: string) => {
  // 等待几秒让后端 webhook 处理完成
  await new Promise(resolve => setTimeout(resolve, 2000));
  
  const { data } = await apolloClient.mutate({
    mutation: QUERY_ORDER_STATUS,
    variables: {
      args: { order_id: parseInt(orderId) },
    },
  });

  const status = data?.fz_invoke_action_flow?.status;
  
  if (status === '已支付') {
    const userId = localStorage.getItem('userId');
    if (userId) {
      const { data: userData } = await apolloClient.query({
        query: GET_USER,
        variables: { userId: parseInt(userId) },
        fetchPolicy: 'network-only',
      });
      
      if (userData?.account_by_pk) {
        useAuthStore.getState().setUser(userData.account_by_pk);
        
        if (userData.account_by_pk.can_use_ai) {
          setTimeout(() => navigate('/'), 2000);
        }
      }
    }
  }
};
```

### Webhook 处理

支付宝可能会通过 webhook 向 Zion 项目的后端发送支付通知。由于 webhook 是异步过程，在支付返回页面查询订单状态前，**应该等待 2 秒左右**，确保后端 webhook 有足够时间处理支付通知并更新订单状态。

---

# 3. Development Best Practices

本文档包含两个部分：
1. **项目开发要求**：技术栈、工具配置、代码质量等具体规范
2. **架构决策和安全实践**：GraphQL CRUD vs Actionflow 的选择原则和安全建议

## 项目开发要求

### 依赖管理

- 使用 npm 安装依赖时：
  - 添加 `--verbose` 标志查看详细安装日志
  - 在 package.json 中指定精确版本号（如 "18.2.0"），不使用 ^ 或 ~ 前缀
  - 避免使用 `npm update` 自动升级依赖
  - 在 `package-lock.json` 中锁定依赖版本

### TypeScript 配置要求 [MUST]

- **严格遵循** TypeScript 编码标准，必须使用 TypeScript 4.9.5
- **模块解析配置**：必须使用 `moduleResolution: "node"`（TypeScript 4.9.5 不支持 `bundler`）
- **必需配置项**：
  - `esModuleInterop: true`
  - `allowSyntheticDefaultImports: true`
  - `strict: true`
  - `noUnusedLocals: true`
  - `noUnusedParameters: true`
  - `noFallthroughCasesInSwitch: true`

**标准 tsconfig.json 配置**：
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

**必需配置文件**：
- `tsconfig.json`
- `tsconfig.node.json`
- `vite.config.ts`

### 代码规范

- **状态管理**: 遵循React状态提升原则，避免多个组件重复调用同一Hook
- 错误处理：使用多种方式显示错误信息（message + 页面内错误提示）
- 页面 title 基于系统场景进行设置，页面的favicon 使用表情符号进行完善
- **Zion 水印按钮**：在页面右下角添加悬浮的"后端由 Zion 驱动"按钮
  - 按钮链接：https://www.functorz.com/vibe-architect-cursor?utm_source=showcase&utm_medium=referral&utm_campaign=cursor&utm_content=watermark
  - **使用持久 URL**：使用持久的 SVG 图片 URL，通过 `<img>` 标签引用

```tsx
export const ZionWatermarkButton: React.FC = () => {
  return (
    <a
      href="https://www.functorz.com/vibe-architect-cursor?utm_source=showcase&utm_medium=referral&utm_campaign=cursor&utm_content=watermark"
      target="_blank"
      rel="noopener noreferrer"
      style={{
        position: 'fixed',
        bottom: '20px',
        right: '20px',
        zIndex: 9999,
      }}
    >
      <img
        src="https://zion-static-public.functorz.com/powered_by_zion.svg"
        alt="Powered by Zion"
        width="181"
        height="48"
      />
    </a>
  );
};
```

### 代码质量要求 [MUST]

**构建前检查清单**：
- ✅ 所有导入的变量和函数必须被使用
- ✅ 所有 TypeScript 类型错误必须修复
- ✅ 不能有未使用的 styled-components 定义
- ✅ 不能有未使用的状态变量
- ✅ 不能使用 `process.env` 而不安装 `@types/node`（推荐直接移除相关代码）
- ✅ 处理 Apollo Client 类型兼容性问题，使用正确的类型定义

**常见错误修复**：
1. **未使用的导入**：移除或使用下划线前缀（如 `const [, setState] = useState()`）
2. **未使用的 styled-components**：删除未使用的样式定义
3. **类型错误**：确保所有函数参数都有类型注解
4. **`process.env` 未定义**：移除相关代码或使用 Vite 的 `import.meta.env` 替代

**导入路径规范**：
```typescript
// ❌ 错误
import App from './App.tsx'

// ✅ 正确
import App from './App'
```

**Styled Components 规范**：
```typescript
// 定义
const SkeletonBase = styled.div<{ $width?: string; $height?: string }>`
  width: ${props => props.$width || '100%'};
`;

// 使用
<Skeleton width="48px" height="48px" />  // ✅ 正确
<Skeleton $width="48px" $height="48px" />  // ❌ 错误
```

**环境变量处理**：
```typescript
// ❌ 避免
if (process.env.NODE_ENV === 'development') { }

// ✅ 推荐
if (import.meta.env.DEV) { }
```

- **可访问性**：iframe 有 title 属性，图片有 alt 属性，交互元素有清晰说明
- **代码规范**：通过 ESLint 检查（`npm run lint`），无警告和错误
- **类型安全**：通过 TypeScript 编译检查（`tsc -b` 或 `npm run build`），无类型错误

### 构建配置

**构建脚本**：
```json
{
  "scripts": {
    "build": "tsc && vite build"
  }
}
```

### Apollo Client 配置

- 必须使用Apollo Client 3.14.0（避免版本兼容性问题）
- 使用 `subscriptions-transport-ws@0.11.0` 库创建 WebSocket 客户端
- 使用 `@apollo/client/link/ws` 的 `WebSocketLink` 创建 WebSocket 链接
- 通过 `split` 函数合并 HTTP 和 WebSocket 链接
- 为 Apollo Client 的 HTTP 链接配置动态 Authorization header
- 确保 Apollo Provider 包装所有使用 GraphQL hooks 的组件

### 手机号认证规范 [MUST]

```graphql
mutation SendVerificationCodeToPhone(
    $telephone: String!
    $verificationEnumType: verificationEnumType!
) {
    sendVerificationCodeToPhone(
    telephone: $telephone
    verificationEnumType: $verificationEnumType
    )
}
```

```graphql
mutation AuthenticateWithPhoneNumber(
    $telephone: String!
    $verificationCode: String
    $password: String
    $register: Boolean!
) {
    authenticateWithPhoneNumber(
    telephone: $telephone
    verificationCode: $verificationCode
    password: $password
    register: $register
    ) {
        account {
            id
            permissionRoles
        }
        jwt {
            token
        }
    }
}
```

**注意事项**：
- `password` 和 `verificationCode` 至少需要提供其中一个
- 注册时，通常两者都需要
- 登录时，可以使用密码或验证码中的任意一个

### 常见问题排查 [REFERENCE]

**问题 1: `Cannot find name 'process'`**
- **原因**：使用了 `process.env` 但没有类型定义
- **解决**：移除相关代码（推荐）或安装 `@types/node`

**问题 2: `Module resolution error`**
- **原因**：TypeScript 配置中的 `moduleResolution` 不兼容
- **解决**：使用 `"moduleResolution": "node"` 而不是 `"bundler"`（TypeScript 4.9.5 要求）

**问题 3: `Unused variable/import`**
- **解决**：移除未使用的变量，或使用下划线前缀

---

## 架构决策和安全实践

### 默认使用 GraphQL CRUD

考虑到开发效率和时间成本，**默认情况下可以使用 GraphQL CRUD 操作**进行快速开发。适合：
- 简单的数据创建、查询、更新、删除
- 不涉及金额、敏感数据或复杂业务逻辑的操作
- 原型开发和快速迭代

### 安全建议：使用 Actionflow 处理敏感操作

对于涉及以下场景的操作，**建议在 Zion 编辑器中配置 Actionflow**：

- **订单创建**：防止价格篡改、库存检查、折扣验证
- **支付状态更新**：防止恶意修改支付状态
- **库存扣减**：确保原子性和一致性
- **用户余额变更**：防止余额被恶意修改
- **优惠券使用**：验证有效性，防止重复使用
- **积分计算和发放**：确保准确性和防刷机制
- **敏感数据的查询和修改**

### 实现方式

订单创建应该通过调用 `fz_invoke_action_flow` mutation 来完成：

```gql
mutation CreateOrder($args: Json!) {
  fz_invoke_action_flow(
    actionFlowId: "{actionFlowId}"
    versionId: {versionId}
    args: $args
  )
}
```

**重要坑点**：`fz_invoke_action_flow` 返回的是 `Json` 类型（叶子类型），**不能进行子选择**。必须直接返回 Json，然后在代码中解析。

---

# 4. UI Design Rules

## Overview

本文档为AI代码生成工具提供UI设计规则，所有规则必须严格遵循。**本规则专为React项目设计**，设计风格采用**有机/自然风格（Organic/Natural）**，强调wabi-sabi美学——接受短暂性和不完美，追求**温暖、柔软和自然连接**。

> **设计哲学**:
> - **wabi-sabi美学**：接受不完美，追求真实和自然
> - **有机形状**：软质blob形状，拒绝90度直角
> - **自然纹理**：全局颗粒纹理叠加（3-4%不透明度，multiply混合模式）
> - **大地色调**：来自森林、粘土、未漂白纸张的调色板
> - **彩色阴影**：柔和阴影带自然色彩色调（苔藓绿、粘土橙），禁止纯黑色

## React技术栈 [MUST]

- **框架**: React（函数组件 + Hooks）
- **样式方案**: Tailwind CSS（使用3.4.0）或 styled-components
- **组件库**: shadcn/ui（复杂组件，需覆盖样式以符合有机风格）
- **规则**: 优先使用Tailwind CSS工具类，有机形状使用内联样式

## 规则执行优先级

- **MUST**: 必须遵循，无例外
- **SHOULD**: 应该遵循，除非有特殊原因
- **MAY**: 可以选择遵循

---

## 1. 颜色系统 [MUST]

### 调色板

```css
--background: #FDFCF8;        /* 米白色，米纸 */
--foreground: #2C2C24;        /* 深壤土/木炭 */
--primary: #5D7052;           /* 苔藓绿 */
--primary-foreground: #F3F4F1; /* 淡雾色 */
--secondary: #C18C5D;         /* 赤陶土/粘土 */
--secondary-foreground: #FFFFFF;
--accent: #E6DCCD;            /* 沙色/米色 */
--accent-foreground: #4A4A40;  /* 树皮色 */
--muted: #F0EBE5;             /* 石头色 */
--muted-foreground: #78786C;   /* 干草色 */
--border: #DED8CF;            /* 原木色 */
--destructive: #A85448;       /* 烧焦的赭石色 */
```

**规则**:
- 页面背景必须使用米白色（#FDFCF8），卡片背景使用极浅米色（#FEFEFA）
- 必须添加全局纹理叠加（3-4%不透明度，multiply混合模式）
- 所有颜色来自自然材料，保持温暖和真实感

### 对比度要求

- 主要文字（#2C2C24）在背景上：14.5:1（AAA）
- 苔藓绿（#5D7052）在背景上：6.2:1（AA）
- 静音文字（#78786C）在背景上：4.8:1（AA）

---

## 2. 排版系统 [MUST]

### 字体

- **标题**: 'Fraunces' (Google Font)，字重600-800
- **正文**: 'Nunito' 或 'Quicksand' (Google Font)

### 字体大小

```css
h1: text-5xl md:text-7xl (移动端48px，桌面端72px)
h2: text-4xl md:text-5xl (移动端36px，桌面端48px)
h3: text-3xl (30px)
h4: text-2xl (24px)
body: text-base (16px)
small: text-sm (14px)
```

---

## 3. 圆角与形状 [MUST]

### 标准圆角

- 按钮: `rounded-full` (完全圆形)
- 标准元素: `rounded-2xl` (16px) 或 `rounded-3xl` (24px)
- 卡片: `rounded-[2rem]` (32px) 作为基础

### 有机形状 [MUST]

**核心特征**: 使用复杂的百分比border-radius创建blob形状。

```css
/* 有机blob形状示例 */
border-radius: 60% 40% 30% 70% / 60% 30% 70% 40%;
border-radius: 30% 60% 70% 40% / 50% 60% 30% 60%;
border-radius: 40% 60% 50% 50% / 40% 50% 60% 50%;

/* 不对称卡片圆角 */
rounded-tl-[4rem] rounded-tr-[2rem] rounded-bl-[2rem] rounded-br-[5rem]
```

**规则**:
- 重要装饰元素（blob背景、图片遮罩）必须使用有机形状
- 卡片可以循环使用不同的border-radius模式
- **禁止**90度直角

---

## 4. 阴影系统 [MUST]

**核心原则**: 使用柔和的、扩散的阴影，带有自然色彩色调（苔藓绿、粘土橙），**禁止纯黑色**。

```css
/* 柔和阴影 - 苔藓色调 */
shadow-[0_4px_20px_-2px_rgba(93,112,82,0.15)]

/* 浮动阴影 - 粘土色调 */
shadow-[0_10px_40px_-10px_rgba(193,140,93,0.2)]

/* 悬停阴影加深 */
hover:shadow-[0_6px_24px_-4px_rgba(93,112,82,0.25)]
hover:shadow-[0_20px_40px_-10px_rgba(93,112,82,0.15)]
```

---

## 5. 组件设计规则

### 5.1 卡片 [MUST]

```tsx
<div className="bg-[#FEFEFA] border border-[#DED8CF]/50 rounded-[2rem] shadow-[0_4px_20px_-2px_rgba(93,112,82,0.15)] p-8 transition-all duration-300 hover:-translate-y-1 hover:shadow-[0_20px_40px_-10px_rgba(93,112,82,0.15)]">
  {children}
</div>
```

### 5.2 按钮 [MUST]

```tsx
/* 主要按钮 */
<button className="bg-[#5D7052] text-[#F3F4F1] rounded-full px-8 py-3 font-bold text-base shadow-[0_4px_20px_-2px_rgba(93,112,82,0.15)] transition-all duration-300 hover:scale-105 hover:shadow-[0_6px_24px_-4px_rgba(93,112,82,0.25)] active:scale-95">
  按钮文字
</button>

/* 轮廓按钮 */
<button className="bg-transparent text-[#C18C5D] border-2 border-[#C18C5D] rounded-full px-8 py-3 font-bold transition-all duration-300 hover:bg-[#C18C5D]/10">
  按钮文字
</button>
```

### 5.3 输入框 [MUST]

```tsx
<input className="bg-white/50 border border-[#DED8CF] rounded-full px-6 py-3 text-sm text-[#2C2C24] font-nunito h-12 transition-all duration-300 focus-visible:ring-2 focus-visible:ring-[#5D7052]/30 focus-visible:ring-offset-2 focus-visible:outline-none" />
```

### 5.4 导航 [SHOULD]

```tsx
<nav className="sticky top-4 z-100 bg-white/70 backdrop-blur-md border border-[#DED8CF]/50 rounded-full px-8 py-3 shadow-[0_4px_20px_-2px_rgba(93,112,82,0.15)]">
  {/* 导航内容 */}
</nav>
```

---

## 6. 布局与间距 [MUST]

### 容器宽度

- Hero/Features/Blog/Pricing: `max-w-7xl` (1280px)
- How It Works/FAQ: `max-w-6xl` (1152px)
- Final CTA: `max-w-5xl` (1024px)
- 文本密集: `max-w-4xl` (896px) 或 `max-w-2xl` (512px)

### 部分内边距

- 垂直: `py-32` (128px)
- 水平: `px-4` (移动端) → `sm:px-6` → `lg:px-8`

### 网格模式

- 统计: `grid-cols-2 md:grid-cols-4`
- Features/Blog/Testimonials: `md:grid-cols-2 lg:grid-cols-3`
- 两列布局: `lg:grid-cols-2`
- 网格间距: `gap-8` (32px)，可选 `md:gap-12` (48px)

---

## 7. 动画规则 [MUST]

### 过渡

```css
transition: all 0.3s ease;  /* duration-300 */
transition: all 0.5s ease;  /* duration-500 */
```

### 悬停动画

- 按钮: `hover:scale-105` 并加深阴影
- 卡片: `hover:-translate-y-1` (提升) 或 `hover:rotate-1` (轻微倾斜)
- 统计数字: `group-hover:scale-110`
- 图片: `hover:scale-105`，700ms持续时间
- 图标容器: 背景色填充过渡

### 激活状态

- 按钮: `active:scale-95` 提供触觉反馈

**规则**: 禁止突然变化，所有过渡使用ease缓动

---

## 8. 非通用元素 [SHOULD]

### Blob背景

```tsx
<div className="absolute inset-0 blur-3xl opacity-30 -z-10" style={{ borderRadius: '60% 40% 30% 70% / 60% 30% 70% 40%', background: '#5D7052' }} />
```

### 旋转图片框架

```css
transform: rotate(-2deg);
border: 4px solid white;
```

### 有机图片遮罩

```css
border-radius: 30% 70% 70% 30% / 30% 30% 70% 70%;
```

### 曲线SVG连接器

**使用场景**: How It Works部分，使用手绘风格的曲线虚线SVG路径

### 悬停微旋转

```css
.testimonial-card:hover { transform: rotate(1deg); }
```

---

## 9. 图标系统 [MUST]

**使用**: Lucide React图标库

**规则**:
- 默认stroke-width: 2px
- 颜色: 苔藓绿（#5D7052）作为默认，深色背景上使用白色
- 容器: `h-14 w-14` (56px)，`rounded-2xl`，`bg-[#5D7052]/10`
- 悬停: 容器完全填充为实心苔藓绿，图标切换为白色
- 尺寸: 功能图标28px，收益检查标记24px

---

## 10. 响应式策略 [MUST]

### 移动优先

- 所有样式从移动端开始
- 使用Tailwind的移动优先断点系统

### 断点

- `sm:` (640px): 水平内边距增加，一些flex-row布局
- `md:` (768px): 主要网格转换（2-3列），导航显示桌面版本
- `lg:` (1024px): 3列网格，2列hero/收益布局

---

## 11. 可访问性 [MUST]

### 焦点状态

```css
focus-visible:ring-2 focus-visible:ring-[#5D7052]/30 focus-visible:ring-offset-2
```

### 触摸目标

- 所有交互元素必须满足44px最小尺寸（按钮h-12 = 48px）

### 语义HTML

- 使用适当的标题层次、导航地标、图片alt文本、需要时使用aria-labels

### 键盘导航

- 所有交互元素键盘可访问

---

## 12. 代码生成检查清单 [MUST]

### 颜色系统
- [ ] 使用定义的调色板（苔藓绿、赤陶土、米白色等）
- [ ] 背景使用米白色（#FDFCF8），卡片使用极浅米色（#FEFEFA）
- [ ] 必须添加全局纹理叠加（3-4%不透明度，multiply混合模式）
- [ ] 文字颜色符合层次规则，对比度符合WCAG标准

### 圆角与形状
- [ ] 按钮使用rounded-full（完全圆形）
- [ ] 标准元素使用rounded-2xl或rounded-3xl
- [ ] 卡片使用rounded-[2rem]作为基础
- [ ] 重要装饰元素使用有机blob形状
- [ ] **禁止**90度直角

### 阴影系统
- [ ] **禁止**使用纯黑色阴影
- [ ] 必须使用彩色阴影（苔藓绿或粘土橙色调）

### 动画
- [ ] 使用transition-all duration-300或duration-500
- [ ] 所有过渡使用ease缓动，持续时间300-700ms
- [ ] 禁止突然变化

---

## 13. 禁止事项

1. ❌ 使用纯白色 `#ffffff` 作为页面背景（应使用米白色 #FDFCF8）
2. ❌ 使用纯黑色阴影（应使用彩色阴影，苔藓绿或粘土橙色调）
3. ❌ 使用90度直角（应使用有机圆角或blob形状）
4. ❌ 缺少全局纹理叠加（必须添加3-4%不透明度，multiply混合模式）
5. ❌ 使用非有机字体（标题必须使用Fraunces，正文必须使用Nunito或Quicksand）
6. ❌ 缺少悬停和焦点状态
7. ❌ 使用linear或突然的snap效果（应使用ease缓动，300-700ms）
8. ❌ 按钮不使用rounded-full（完全圆形）

---

# 5. Zeabur Deployment

本文档总结了在 Zeabur 平台上部署 React + TypeScript + Vite 项目的最佳实践和注意事项。

## 核心要求

### 1. 必需配置文件 [MUST]

- **`tsconfig.json`**: TypeScript 编译配置
- **`tsconfig.node.json`**: Node.js 环境的 TypeScript 配置（用于 `vite.config.ts`）
- **`vite.config.ts`**: Vite 构建工具配置
- **`package.json`**: 项目依赖和脚本配置

### 2. TypeScript 配置要求 [MUST]

**兼容性要求**：
- 如果使用 TypeScript 4.9.5，必须使用 `moduleResolution: "node"`（不支持 `bundler`）
- 必须包含 `esModuleInterop: true` 和 `allowSyntheticDefaultImports: true`

### 3. 构建脚本配置 [MUST]

```json
{
  "scripts": {
    "build": "tsc && vite build"
  }
}
```

### 4. SPA 部署配置（React Router 等）[MUST]

**对于使用 React Router 的单页应用，必须在部署前配置**：

1. **配置 `vite.config.ts`**：
   ```typescript
   export default defineConfig({
     publicDir: 'public',
     preview: {
       host: '0.0.0.0',
       port: 8080, // Zeabur 使用的端口
       strictPort: false,
       allowedHosts: true,
     },
   })
   ```

2. **配置 `package.json`**：
   ```json
   {
     "scripts": {
       "start": "vite preview --host 0.0.0.0 --port ${PORT:-8080}"
     }
   }
   ```

3. **在 Zeabur 中配置**：
   - 服务类型：**Web Service**（不是 Static Site）
   - 启动命令：`npm start`
   - 构建输出目录：`dist`

**为什么需要这些配置**：
- `allowedHosts: true`：防止 "Blocked request. This host is not allowed" 错误
- `vite preview`：自动处理 SPA 路由重定向，防止 502 错误
- Web Service 类型：提供服务器支持，处理客户端路由

### 5. 502 错误和主机访问限制排查 [REFERENCE]

**问题描述**：部署显示成功，但访问页面时出现 `502: SERVICE_UNAVAILABLE` 错误，或显示 "Blocked request. This host is not allowed" 错误。

**根本原因**：
1. SPA 路由需要服务器支持：React Router 等单页应用需要服务器将所有路由请求重定向到 `index.html`
2. Vite preview 的主机访问限制：默认只允许 localhost 访问，需要配置允许外部域名

**解决方案（推荐）**：

#### 使用 Web Service + Vite Preview

1. **在 Zeabur 控制台中配置**：
   - 服务类型：**Web Service**（不是 Static Site）
   - 启动命令：`npm start`
   - 构建输出目录：`dist`

2. **配置 `vite.config.ts`**：
   ```typescript
   export default defineConfig({
     preview: {
       host: '0.0.0.0',
       port: 8080,
       strictPort: false,
       allowedHosts: true,
     },
   })
   ```

3. **配置 `package.json`**：
   ```json
   {
     "scripts": {
       "start": "vite preview --host 0.0.0.0 --port ${PORT:-8080}"
     }
   }
   ```

**重要注意事项**：
- ✅ `allowedHosts` **只能**在 `vite.config.ts` 配置文件中设置，**不能**作为命令行参数
- ✅ 设置为 `true` 允许所有主机访问（适合生产环境部署）
- ✅ 端口设置为 `8080`（Zeabur 默认端口），或使用 `${PORT:-8080}` 从环境变量读取
- ✅ `vite preview` 会自动处理 SPA 路由重定向
- ❌ **不要**在命令行中使用 `--allowedHosts`，会导致 `CACError: Unknown option '--allowedHosts'` 错误

#### 常见错误排查

1. **`CACError: Unknown option '--allowedHosts'`**：
   - **原因**：`allowedHosts` 不能作为命令行参数
   - **解决**：只在 `vite.config.ts` 中配置 `allowedHosts: true`

2. **`Blocked request. This host is not allowed`**：
   - **原因**：Vite preview 默认只允许 localhost 访问
   - **解决**：在 `vite.config.ts` 中设置 `allowedHosts: true`

3. **502 错误**：
   - 确认服务类型为 **Web Service**（不是 Static Site）
   - 确认启动命令为 `npm start`
   - 确认 `vite.config.ts` 中 `preview` 配置正确
   - 确认端口设置为 `8080`
   - 查看 Zeabur 部署日志，检查是否有运行时错误

#### 方案 2：SPA 路由重定向配置

1. **创建 `public/_redirects` 文件**：
   ```
   /*    /index.html   200
   ```

2. **确保 `vite.config.ts` 包含 `publicDir` 配置**：
   ```typescript
   export default defineConfig({
     plugins: [react()],
     publicDir: 'public',
   })
   ```

3. **验证构建输出**：
   - 构建后，`dist/_redirects` 文件应该存在

#### 方案 3：使用 serve 作为备用

```json
{
  "scripts": {
    "start": "npx serve -s dist -l $PORT"
  }
}
```

- 在 Zeabur 中配置为 **Web Service**，启动命令设置为 `npm start`
- `serve -s` 会自动处理 SPA 路由重定向

## 部署前检查清单

- [ ] 本地 `npm run build` 成功执行
- [ ] 所有必需的配置文件都存在
- [ ] `package.json` 中的依赖版本已锁定
- [ ] `vite.config.ts` 中已配置 `preview.allowedHosts: true`
- [ ] `vite.config.ts` 中已配置 `preview.port: 8080`
- [ ] `vite.config.ts` 中已配置 `preview.host: '0.0.0.0'`
- [ ] `package.json` 中已配置 `start` 脚本
- [ ] 在 Zeabur 中已配置服务类型为 **Web Service**
- [ ] 在 Zeabur 中已设置启动命令为 `npm start`
- [ ] 在 Zeabur 中已设置构建输出目录为 `dist`

---

# 6. WeChat Mini Program

## Overview

本文档规定了微信小程序与 Zion.app 后端集成时的开发规则和最佳实践。

## 模块系统

### 使用 CommonJS [MUST]

```javascript
// ✅ 正确
const { functionName } = require('./utils/file.js')
module.exports = { functionName }

// ❌ 错误
import { functionName } from './utils/file.js'
```

## 网络请求

### 使用 wx.request [MUST]

```javascript
function graphqlRequest(query, variables = {}, token = null) {
  return new Promise((resolve, reject) => {
    wx.request({
      url: 'https://zion-app.functorz.com/zero/{projectExId}/api/graphql-v2',
      method: 'POST',
      data: { query, variables },
      header: {
        'Content-Type': 'application/json',
        ...(token ? { 'Authorization': `Bearer ${token}` } : {})
      },
      success: (res) => {
        if (res.data.errors) {
          reject(new Error(res.data.errors[0].message))
        } else {
          resolve(res.data.data)
        }
      },
      fail: reject
    })
  })
}
```

## 用户认证

### 静默登录实现 [MUST]

```javascript
// 1. 获取微信登录 code
const loginRes = await new Promise((resolve, reject) => {
  wx.login({ success: resolve, fail: reject })
})

// 2. 调用 Zion GraphQL mutation 进行静默登录
const query = `
  mutation LoginWithWechatMiniApp($code: String!) {
    loginWithWechatMiniApp(code: $code) {
      account {
        id
        username
        profileImageUrl  // 必须包含
      }
      jwt { token }
    }
  }
`

const result = await graphqlRequest(query, { code: loginRes.code })
const { account, jwt } = result.loginWithWechatMiniApp

// 3. 保存 token 和账户信息
wx.setStorageSync('token', jwt.token)
wx.setStorageSync('account', account)
```

**开发要求**：
- **必须请求 `profileImageUrl` 字段**：用于后续头像显示
- `wx.login` 返回的 `code` 只能使用一次，5 分钟内有效

## 文件上传到 OSS

### 上传流程 [MUST]

#### Step 1: 计算文件的 MD5 Base64

**必须使用 `wx.getFileInfo` API 计算 MD5**：

```javascript
const fileInfo = await new Promise((resolve, reject) => {
  wx.getFileInfo({ filePath: filePath, digestAlgorithm: 'md5', success: resolve, fail: reject })
})
// fileInfo.digest 是十六进制字符串，需要转换为 Base64
```

#### Step 2: 获取 Presigned Upload URL

```javascript
const query = `mutation GetImagePresignedUrl($md5: String!, $suffix: MediaFormat!) {
  imagePresignedUrl(imgMd5Base64: $md5, imageSuffix: $suffix, acl: PRIVATE) {
    imageId uploadUrl uploadHeaders
  }
}`
const result = await graphqlRequest(query, { md5: md5Base64, suffix: 'JPEG' }, token)
const { imageId, uploadUrl, uploadHeaders } = result.imagePresignedUrl
```

#### Step 3: 上传文件

**必须遵循以下规则**：
1. **必须保留所有 uploadHeaders**：presigned URL 的签名是基于这些 headers 计算的，移除任何 header 都会导致 403 错误
2. **必须确保上传的数据和计算 MD5 时使用的数据完全一致**
3. **必须使用真正的 ArrayBuffer**：不能依赖 `instanceof ArrayBuffer` 检查

```javascript
// 读取文件数据并转换为 ArrayBuffer
const fileManager = wx.getFileSystemManager()
let fileData = fileManager.readFileSync(filePath)
let originalFileData = fileData instanceof ArrayBuffer ? fileData : 
  (() => {
    const uint8Array = new Uint8Array(fileData)
    const buffer = new ArrayBuffer(uint8Array.length)
    new Uint8Array(buffer).set(uint8Array)
    return buffer
  })()

// 上传文件（必须保留所有 uploadHeaders）
await new Promise((resolve, reject) => {
  wx.request({
    url: uploadUrl,
    method: 'PUT',
    data: originalFileData,
    header: { 'Content-Type': 'image/jpeg', ...uploadHeaders },
    responseType: 'text',
    success: (res) => {
      if (res.statusCode === 200 || res.statusCode === 204) {
        resolve(imageId)
      } else {
        reject(new Error(`上传失败: ${res.statusCode}`))
      }
    },
    fail: reject
  })
})
```

### 常见错误处理

- **`InvalidDigest`**：必须使用 `wx.getFileInfo` API 计算 MD5，确保上传数据与计算 MD5 时使用的数据完全一致
- **`SignatureDoesNotMatch`**：必须保留所有 uploadHeaders，不要移除任何 header（特别是 Content-MD5）
- **`文件数据格式不正确`**：使用属性检查而不是 `instanceof`，强制转换为真正的 ArrayBuffer

### 文件下载（处理网络 URL）[MUST]

**处理网络资源时必须先下载到本地**：

```javascript
if (avatarUrl.startsWith('http://') || avatarUrl.startsWith('https://')) {
  const downloadRes = await new Promise((resolve, reject) => {
    wx.downloadFile({
      url: avatarUrl,
      success: (res) => {
        if (res.statusCode === 200 && res.tempFilePath) {
          resolve(res)
        } else {
          reject(new Error('下载失败'))
        }
      },
      fail: reject
    })
  })
  localFilePath = downloadRes.tempFilePath
}
```

## 用户信息编辑

### 头像选择实现 [MUST]

```xml
<button open-type="chooseAvatar" bindchooseavatar="onChooseAvatar">
  <view class="avatar-wrapper">
    <image class="avatar" src="{{userInfo.avatar}}" mode="aspectFill" wx:if="{{userInfo.avatar}}"></image>
    <view class="avatar-placeholder" wx:else>
      <text>{{userInfo.nickname ? userInfo.nickname.charAt(0) : '👤'}}</text>
    </view>
  </view>
</button>
```

```javascript
async onChooseAvatar(e) {
  let localFilePath = e.detail.avatarUrl
  // 处理网络 URL（微信默认头像需要先下载）
  if (localFilePath.startsWith('http://') || localFilePath.startsWith('https://')) {
    const downloadRes = await new Promise((resolve, reject) => {
      wx.downloadFile({ url: localFilePath, success: resolve, fail: reject })
    })
    localFilePath = downloadRes.tempFilePath
  }
  const imageId = await uploadImage(localFilePath, token)
  await updateAccount(accountId, imageId, null, token)
}
```

### 昵称编辑实现 [MUST]

```xml
<input type="nickname" value="{{userInfo.nickname}}" bindblur="onNicknameBlur" />
```

```javascript
async onNicknameBlur(e) {
  const nickname = e.detail.value.trim()
  if (nickname && nickname !== this.data.userInfo.nickname) {
    await updateAccount(accountId, null, nickname, token)
  }
}
```

## 数据模型操作

### 创建记录 [MUST]

**当表不支持直接设置外键字段时，必须使用两步操作**：

```javascript
// Step 1: 先创建空记录
const createResult = await graphqlRequest(`
  mutation CreateRecord {
    insert_record_one(object: {}) {
      id
    }
  }
`, {}, token)

// Step 2: 更新记录，设置关联字段
await graphqlRequest(`
  mutation UpdateRecord($id: bigint!, $img_id: bigint!, $user_id: bigint) {
    update_record_by_pk(
      pk_columns: {id: $id}
      _set: {img_id: $img_id, user_id: $user_id}
    ) { id }
  }
`, { id: createResult.insert_record_one.id, img_id: imageId, user_id: userId }, token)
```

## 开发要求

### 头像上传和读取 [MUST]

- **必须使用 `wx.getFileInfo` API 计算 MD5**，不要手动实现或使用第三方库
- **必须保留所有 uploadHeaders**（特别是 Content-MD5），presigned URL 的签名依赖所有 headers
- **必须确保上传的数据和计算 MD5 时使用的数据完全一致**
- **必须处理网络 URL**：微信默认头像等网络资源需要先使用 `wx.downloadFile` 下载到本地
- **登录时必须请求 `profileImageUrl` 字段**
- **必须从服务器获取最新信息**：页面加载时优先从服务器获取账户信息，不能只依赖本地存储

### 自定义 TabBar [MUST]

- **必须在 `app.json` 中设置 `"custom": true`**
- **必须在页面 JSON 配置中引用组件**：`"usingComponents": { "custom-tab-bar": "/custom-tab-bar/index" }`
- **必须在页面 `onLoad` 中更新选中状态**：使用 `this.getTabBar().setData({ selected: index })`
- **必须为页面添加底部 padding**：避免内容被 tabBar 遮挡，建议 `padding-bottom: calc(160rpx + env(safe-area-inset-bottom))`

```javascript
// app.json
"tabBar": { "custom": true, "list": [...] }

// 页面 JSON
{ "usingComponents": { "custom-tab-bar": "/custom-tab-bar/index" } }

// 页面 JS
onLoad() {
  if (typeof this.getTabBar === 'function' && this.getTabBar()) {
    this.getTabBar().setData({ selected: 0 })
  }
}
```

### 变量作用域 [MUST]

**循环内使用的变量必须在循环外声明**：

```javascript
// ❌ 错误：在循环内声明，循环外使用
while (attempts < maxAttempts) {
  let imageId = null
}
this.saveToHistory(imageUrl, imageId)  // 会报错

// ✅ 正确：在循环外声明
let imageId = null
while (attempts < maxAttempts) {
  if (content.image.id) {
    imageId = content.image.id
  }
}
this.saveToHistory(imageUrl, imageId)  // 正常使用
```

### ArrayBuffer 处理 [MUST]

- **不能依赖 `instanceof ArrayBuffer` 检查**：小程序环境中可能返回 `false`
- **必须使用属性检查**：检查 `byteLength`、`buffer`、`byteOffset` 等属性
- **必须转换为真正的 ArrayBuffer**：确保上传时使用真正的 ArrayBuffer

### 页面布局 [MUST]

- **自定义 tabBar 页面必须添加底部 padding**：避免内容被遮挡
- **必须考虑安全区域**：使用 `env(safe-area-inset-bottom)` 适配不同设备

```css
.container {
  padding-bottom: calc(160rpx + env(safe-area-inset-bottom));
}
```

### 自定义组件中加载远程 Icon（SVG/图片）[MUST]

- **自定义组件中不能直接 require 其他模块**：会导致 `can not find module` 错误
- **必须直接使用 `wx.request` 调用 GraphQL API**：避免 require 依赖问题
- **必须提供 fallback（emoji 或占位符）**：确保即使 icon 加载失败，组件也能正常显示
- **必须在 `ready` 生命周期中加载**：不阻塞组件初始化

```javascript
const app = getApp()
const ZION_GRAPHQL_URL = 'https://zion-app.functorz.com/zero/{projectExId}/api/graphql-v2'

Component({
  data: { iconUrl: '' },
  lifetimes: {
    attached() {}, // 立即显示组件（使用 fallback）
    ready() {
      this.loadIcon().catch(err => console.error('加载 icon 失败:', err))
    }
  },
  methods: {
    loadIcon() {
      return new Promise((resolve, reject) => {
        const token = app.getToken() || null
        wx.request({
          url: ZION_GRAPHQL_URL,
          method: 'POST',
          header: {
            'Content-Type': 'application/json',
            ...(token ? { 'Authorization': `Bearer ${token}` } : {})
          },
          data: {
            query: `query GetImageById($imageId: bigint!) {
              getImageById(imageId: $imageId) { id url }
            }`,
            variables: { imageId: 7000000007117943 }
          },
          success: (res) => {
            if (res.statusCode === 200 && res.data.data?.getImageById?.url) {
              this.setData({ iconUrl: res.data.data.getImageById.url })
              resolve(res.data.data.getImageById.url)
            } else {
              reject(new Error('获取 icon 失败'))
            }
          },
          fail: reject
        })
      })
    }
  }
})
```

```xml
<view class="icon-wrapper">
  <image class="icon-image" src="{{iconUrl}}" mode="aspectFit" wx:if="{{iconUrl}}"></image>
  <text class="icon-emoji" wx:if="{{!iconUrl}}">🎨</text>
</view>
```

## Lottie 动画集成

### 库文件引入 [MUST]

**从 npm/unpkg 下载的 `lottie-miniprogram` 是 webpack 打包的 UMD 格式，必须进行以下处理**：

1. **初始化 exports 对象**：在文件开头添加 `var exports = typeof exports !== 'undefined' ? exports : {};`
2. **添加 module.exports**：在文件末尾添加 CommonJS 导出

```javascript
if (typeof module !== 'undefined' && module.exports) {
  module.exports = {
    setup: exports.setup,
    loadAnimation: exports.loadAnimation,
    freeze: exports.freeze,
    unfreeze: exports.unfreeze
  }
}
```

### Canvas 2D 使用 [MUST]

```javascript
const query = wx.createSelectorQuery().in(this)
query.select('#lottie-canvas').fields({ node: true, size: true }).exec((res) => {
  const canvas = res[0].node
  const ctx = canvas.getContext('2d')
  const dpr = wx.getSystemInfoSync().pixelRatio
  const width = res[0].width || 600
  const height = res[0].height || 600
  
  canvas.width = width * dpr
  canvas.height = height * dpr
  ctx.scale(dpr, dpr)
  
  lottie.setup(canvas)
  lottie.loadAnimation({
    loop: true,
    autoplay: true,
    animationData: lottieData,
    rendererSettings: { context: ctx, clearCanvas: true }
  })
})
```

**必须遵循的规则**：
- Canvas 必须使用 `type="2d"` 和 `id` 属性
- 必须设置 `canvas.width` 和 `canvas.height`（考虑 DPR）
- Lottie JSON 数据通过 `getFileById` 从 Zion OSS 获取，然后使用 `wx.request` 下载内容

---

# 7. WeChat Mini Program Payment

## 概述

Zion 支持微信小程序原生支付集成，使得基于 Zion 构建的微信小程序可以使用微信支付完成订单支付。

## 前置条件

1. **选择订单表**：在 Zion 编辑器内将某个表绑定为项目的"订单表"设置。每个支付都必须关联一个订单 ID。
2. **配置微信支付密钥和证书**：在 Zion 编辑器内配置微信支付的 AppID、商户号、API 密钥等。

## 订单创建

每个项目的概念订单表都有自己的结构，不一定需要命名为 "order"。通常应包含：订单属于哪个账户、订单金额、订单包含哪些商品（通常通过 1:n 关系）。

### 创建订单示例（使用 Actionflow）

**重要说明**：
- `fz_invoke_action_flow` 返回的是 `Json` 类型（叶子类型），**不能进行子选择**
- 必须直接返回 Json，然后在代码中解析

```javascript
const query = `
  mutation CreateOrder($args: Json!) {
    fz_invoke_action_flow(
      actionFlowId: "0e4de7f6-2ef2-49ee-8b2f-2b16afc6767d"
      versionId: -1
      args: $args
    )
  }
`

const variables = {
  args: {
    amount: 100  // 订单金额（单位：分）
  }
}

const result = await graphqlRequest(query, variables, token)
// fz_invoke_action_flow 返回的是 Json 类型，直接解析
const orderData = result.fz_invoke_action_flow
if (!orderData) {
  throw new Error('创建订单失败：返回数据为空')
}
const orderId = orderData.order_id
const amount = orderData.amount
```

## 创建微信支付

前置条件：订单 ID、商品描述、订单总金额（单位：分）和认证 token。

**重要说明**：
- 必须使用创建订单 ActionFlow 输出的订单 ID 和金额（不要有自定义订单逻辑）
- 调用 Zion 提供的 `createWechatPayment` 接口时，确保请求包含 Bearer token
- `amount` 参数类型是 `BigDecimal!`，**单位是元**，需要将分转换为元（除以 100）
- `type` 参数必须设置为 `WECHATPAY_MINIPROGRAM`
- 返回的 `SignResult.message` 字段包含 JSON 格式的支付参数，需要解析

```gql
mutation CreateWechatPayment(
  $orderId: Long!
  $amount: BigDecimal!
  $description: String!
  $type: PaymentType!
) {
  createWechatPayment(
    orderId: $orderId
    amount: $amount
    description: $description
    type: $type
  ) {
    message
    status
  }
}
```

变量示例：
```json
{
  "orderId": 1234567890,
  "amount": 0.01,
  "description": "积分充值",
  "type": "WECHATPAY_MINIPROGRAM"
}
```

输出示例：
```json
{
  "data": {
    "createWechatPayment": {
      "status": "SUCCESS",
      "message": "{\"appId\":\"wx1234567890abcdef\",\"timeStamp\":\"1234567890\",...}"
    }
  }
}
```

**关键点**：
- `status` 字段：`SUCCESS` 表示成功，`FAIL` 表示失败
- `message` 字段：包含 JSON 字符串格式的支付参数，需要 `JSON.parse()` 解析
- 解析后的 JSON 包含：`appId`、`timeStamp`、`nonceStr`、`package`、`signType`、`paySign`

## 调用微信支付

**必须使用微信小程序原生 API `wx.requestPayment`**：

```javascript
wx.requestPayment({
  appId: paymentParams.appId,
  timeStamp: paymentParams.timeStamp,
  nonceStr: paymentParams.nonceStr,
  package: paymentParams.package,
  signType: paymentParams.signType,
  paySign: paymentParams.paySign,
  success: (res) => {
    console.log('支付成功', res)
  },
  fail: (err) => {
    console.error('支付失败', err)
  }
})
```

**关键点**：
- 必须使用 `wx.requestPayment` API
- 所有参数必须与后端返回的参数完全一致，不能修改
- 支付成功后，微信会自动调用后端的支付回调接口（webhook）

## 支付返回处理

当用户完成支付后（无论成功或失败），`wx.requestPayment` 的 `success` 或 `fail` 回调会被触发。在该回调中需要：

1. **等待后端处理**：由于 webhook 是异步过程，应该等待 2-3 秒让后端 webhook 有足够时间处理支付通知并更新订单状态
2. **查询订单状态**：使用订单 ID 查询订单状态，确认支付是否完成
3. **更新用户权限**：如果订单状态为"已支付"，更新用户权限（如积分余额）
4. **处理支付结果**：根据订单状态显示相应的提示信息
5. **页面跳转**：如果支付成功，可以跳转到应用主页或其他页面

## Webhook 处理

微信支付会通过 webhook 向 Zion 项目的后端发送支付通知。前端不需要特殊处理 webhook 处理器的逻辑。

由于 webhook 是异步过程，在支付成功后查询订单状态前，**应该等待 2-3 秒**，确保后端 webhook 有足够时间处理支付通知并更新订单状态。

## 完整支付工具函数示例

```javascript
// utils/payment.js
const { graphqlRequest } = require('./graphql.js')

async function createOrder(amount, token) {
  const query = `
    mutation CreateOrder($args: Json!) {
      fz_invoke_action_flow(
        actionFlowId: "0e4de7f6-2ef2-49ee-8b2f-2b16afc6767d"
        versionId: -1
        args: $args
      )
    }
  `
  const variables = { args: { amount: amount } }
  const result = await graphqlRequest(query, variables, token)
  const orderData = result.fz_invoke_action_flow
  if (!orderData) {
    throw new Error('创建订单失败：返回数据为空')
  }
  return {
    order_id: orderData.order_id,
    amount: orderData.amount
  }
}

async function createWechatPay(outTradeNo, description, totalAmount, token) {
  const amountInYuan = totalAmount / 100
  
  const query = `
    mutation CreateWechatPayment(
      $orderId: Long!
      $amount: BigDecimal!
      $description: String!
      $type: PaymentType!
    ) {
      createWechatPayment(
        orderId: $orderId
        amount: $amount
        description: $description
        type: $type
      ) {
        message
        status
      }
    }
  `
  
  const variables = {
    orderId: parseInt(outTradeNo),
    amount: amountInYuan,
    description: description,
    type: 'WECHATPAY_MINIPROGRAM'
  }
  
  const result = await graphqlRequest(query, variables, token)
  const signResult = result.createWechatPayment
  
  if (signResult.status !== 'SUCCESS') {
    throw new Error('创建微信支付失败：' + signResult.message)
  }
  
  try {
    const paymentParams = JSON.parse(signResult.message)
    return paymentParams
  } catch (e) {
    throw new Error('解析支付参数失败：' + signResult.message)
  }
}

async function processWechatPayment(amount, description, token) {
  try {
    const order = await createOrder(amount, token)
    const orderId = order.order_id
    
    const paymentParams = await createWechatPay(
      orderId.toString(),
      description,
      amount,
      token
    )
    
    return await new Promise((resolve, reject) => {
      wx.requestPayment({
        appId: paymentParams.appId,
        timeStamp: paymentParams.timeStamp,
        nonceStr: paymentParams.nonceStr,
        package: paymentParams.package,
        signType: paymentParams.signType,
        paySign: paymentParams.paySign,
        success: async (res) => {
          await new Promise(resolve => setTimeout(resolve, 2000))
          
          try {
            const orderStatus = await queryOrderStatus(orderId, token)
            resolve({
              orderId: orderId,
              success: orderStatus.status === '已支付',
              orderStatus: orderStatus
            })
          } catch (error) {
            resolve({
              orderId: orderId,
              success: false,
              error: error.message
            })
          }
        },
        fail: (err) => {
          reject(err)
        }
      })
    })
  } catch (error) {
    console.error('微信支付失败:', error)
    throw error
  }
}

module.exports = {
  createOrder,
  createWechatPay,
  processWechatPayment
}
```

## 注意事项

1. **金额单位转换**：
   - 订单金额在前端使用**分**为单位（如 100 表示 1.00 元）
   - 调用 `createWechatPayment` 时，`amount` 参数类型是 `BigDecimal!`，**单位是元**，需要除以 100 转换

2. **订单 ID**：必须使用创建订单 ActionFlow 返回的订单 ID，不要自定义

3. **支付参数解析**：
   - `createWechatPayment` 返回 `SignResult` 类型
   - `status` 字段必须为 `SUCCESS` 才表示成功
   - `message` 字段包含 JSON 字符串，需要 `JSON.parse()` 解析后才能获取支付参数

4. **支付参数**：所有支付参数必须与后端返回的参数完全一致，不能修改

5. **异步处理**：支付成功后需要等待 2-3 秒再查询订单状态，确保后端 webhook 处理完成

6. **Actionflow 返回类型**：
   - `fz_invoke_action_flow` 返回 `Json` 类型（叶子类型），**不能进行子选择**
   - 必须直接返回 Json，然后在代码中解析
