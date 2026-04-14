---
title: GraphQL基础
date: 2026-04-11
description: GraphQL查询语言与API设计基础
tags:
  - graphql
  - api
  - web
draft: false
permalink:
---

## 概述

GraphQL 是一种用于 API 的查询语言，由 Facebook 在 2012 年开发，2015 年开源。它提供了一种比 REST 更高效、灵活的方式来获取和操作数据。[^1]

### 核心特点

- **精确获取**：客户端可以指定需要哪些字段，避免过度获取（over-fetching）或不足获取（under-fetching）
- **单一端点**：所有请求都发送到同一个端点（如 `/graphql`）
- **强类型模式**：Schema 是强类型的，提供完整的类型系统
- **自描述**：Introspection 允许客户端查询 API 的类型信息

### REST vs GraphQL

| 特性 | REST | GraphQL |
|------|------|---------|
| 数据获取 | 多个端点 | 单一端点 |
| 获取字段 | 固定返回 | 客户端指定 |
| 传输数据量 | 可能过多/不足 | 精确获取 |
| 类型系统 | 无 | 强类型 Schema |

---

## Schema 定义

GraphQL 使用 Schema Definition Language（SDL）来定义 API 的类型和操作。

### 基本类型

```graphql
# 标量类型：String, Int, Float, Boolean, ID

type User {
  id: ID!
  name: String!
  email: String!
  age: Int
  bio: String
}

# ! 表示非空
# ID 是唯一标识符类型
```

### 接口与联合类型

```graphql
# 接口定义
interface Node {
  id: ID!
}

type User implements Node {
  id: ID!
  name: String!
  email: String!
}

# 联合类型
union SearchResult = User | Post | Comment
```

### 输入类型

```graphql
# 用于 mutation 的输入参数
input CreateUserInput {
  name: String!
  email: String!
  age: Int
}

type Mutation {
  createUser(input: CreateUserInput!): User!
}
```

---

## 查询（Query）

Query 用于读取（获取）数据。

### 定义 Query

```graphql
type Query {
  # 获取单个用户
  user(id: ID!): User
  
  # 获取用户列表
  users: [User!]!
  
  # 带分页的查询
  posts(first: Int, after: String): PostConnection!
  
  # 根据条件查询
  postsByAuthor(authorId: ID!): [Post!]!
}
```

### 客户端查询

```graphql
# 查询单个用户及其帖子
query GetUser($id: ID!) {
  user(id: $id) {
    name
    email
    posts(first: 5) {
      title
      content
      createdAt
    }
  }
}

# 变量传递
{
  "id": "123"
}
```

### 片段（Fragments）

```graphql
query GetUsers {
  user1: user(id: "1") {
    ...userFields
  }
  user2: user(id: "2") {
    ...userFields
  }
}

fragment userFields on User {
  name
  email
  age
}
```

---

## 变更（Mutation）

Mutation 用于修改（创建、更新、删除）数据。

### 定义 Mutation

```graphql
type Mutation {
  # 创建用户
  createUser(input: CreateUserInput!): User!
  
  # 更新用户
  updateUser(id: ID!, input: UpdateUserInput!): User
  
  # 删除用户
  deleteUser(id: ID!): Boolean!
  
  # 批量操作
  bookTrips(launchIds: [ID!]!): TripUpdateResponse!
}
```

### 客户端 Mutation

```graphql
mutation CreateNewUser($input: CreateUserInput!) {
  createUser(input: $input) {
    id
    name
    email
  }
}

# 变量
{
  "input": {
    "name": "张三",
    "email": "zhang@example.com",
    "age": 25
  }
}
```

### 变更响应

```graphql
# 好的实践：返回被修改的对象
type TripUpdateResponse {
  success: Boolean!
  message: String
  launches: [Launch]
}
```

---

## 订阅（Subscription）

Subscription 用于实时数据更新，基于 WebSocket。

```graphql
type Subscription {
  # 新用户创建时通知
  userCreated: User!
  
  # 帖子更新时通知
  postUpdated(postId: ID!): Post
}
```

---

## Resolver

Resolver 是负责为 Schema 中每个字段获取数据的函数。

### 基本结构

```typescript
// Apollo Server 示例
const resolvers = {
  Query: {
    user: async (parent, args, context) => {
      // args 包含查询参数
      return await context.db.users.findById(args.id);
    },
    
    users: async (parent, args, context) => {
      return await context.db.users.findAll();
    }
  },
  
  // User 类型的字段 resolver
  User: {
    // 如果不定义，默认使用 parent 对象上同名字段
    posts: async (parent, args, context) => {
      // parent 是 User 对象
      return await context.db.posts.findByAuthorId(parent.id);
    }
  }
};
```

### Resolver 参数

```typescript
// resolver 函数签名
fieldName: (parent, args, context, info) => data;

// parent: 父对象（如果是顶层查询则为 undefined）
// args: 查询参数
// context: 共享上下文（数据库连接、认证信息等）
// info: 包含查询信息的 AST
```

### 默认 Resolver

```typescript
// 如果字段名与属性名相同，可以不定义 resolver
const resolvers = {
  User: {
    // name 字段会自动返回 parent.name
    // posts 字段需要自定义 resolver
    posts: (parent) => getPostsByAuthor(parent.id)
  }
};
```

---

## 分页实现

### 游标分页（推荐）

```graphql
type Connection {
  edges: [Edge!]!
  pageInfo: PageInfo!
}

type Edge {
  node: User!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type Query {
  users(first: Int, after: String): UserConnection!
}
```

### 偏移量分页（简单但不推荐）

```graphql
type Query {
  # offset-limit 分页
  posts(offset: Int, limit: Int): [Post!]!
}
```

---

## 错误处理

### GraphQL 错误结构

```json
{
  "errors": [
    {
      "message": "User not found",
      "extensions": {
        "code": "NOT_FOUND",
        "field": "id"
      },
      "path": ["user"]
    }
  ],
  "data": {
    "user": null
  }
}
```

### 自定义错误

```typescript
import { GraphQLError } from 'graphql';

function notFoundError(type: string, id: string) {
  return new GraphQLError(`${type} with id ${id} not found`, {
    extensions: {
      code: 'NOT_FOUND',
      type,
      id
    }
  });
}

// 在 resolver 中使用
const resolvers = {
  Query: {
    user: async (_, { id }) => {
      const user = await db.users.findById(id);
      if (!user) {
        throw notFoundError('User', id);
      }
      return user;
    }
  }
};
```

---

## N+1 问题与 DataLoader

### N+1 问题示例

```typescript
// 问题：查询 10 个用户及其帖子，会执行 11 次数据库查询
const resolvers = {
  Query: {
    users: () => db.users.findAll()  // 1 次查询
  },
  User: {
    posts: (user) => db.posts.findByAuthorId(user.id)  // N 次查询！
  }
};
```

### DataLoader 解决方案

```typescript
import DataLoader from 'dataloader';

// 创建 batch 函数
const createUserLoader = () => new DataLoader(
  async (ids) => {
    const users = await db.users.findByIds(ids);
    // 返回结果顺序必须与输入 ids 顺序一致
    return ids.map(id => users.find(u => u.id === id));
  }
);

// 在 context 中注入
const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: () => ({
    userLoader: createUserLoader()
  })
});

// 在 resolver 中使用
const resolvers = {
  User: {
    posts: async (user, _, { userLoader }) => {
      return userLoader.load(user.id);
    }
  }
};
```

---

## 认证与授权

### 在 Context 中添加认证信息

```typescript
context: ({ req }) => {
  const token = req.headers.authorization;
  const user = verifyToken(token);
  return { user };
}
```

### 在 Resolver 中检查权限

```typescript
const resolvers = {
  Mutation: {
    deletePost: async (_, { id }, { user }) => {
      if (!user) {
        throw new GraphQLError('Authentication required', {
          extensions: { code: 'UNAUTHENTICATED' }
        });
      }
      
      const post = await db.posts.findById(id);
      if (post.authorId !== user.id && !user.isAdmin) {
        throw new GraphQLError('Not authorized', {
          extensions: { code: 'FORBIDDEN' }
        });
      }
      
      return db.posts.delete(id);
    }
  }
};
```

### Schema 级授权

```graphql
# 使用 @auth 指令（需要自定义实现）
type User {
  id: ID!
  email: String! @auth(requires: ADMIN)
  posts: [Post!]!
}
```

---

## 最佳实践

### Schema 设计原则

1. **使用非空类型默认**：只在有实际意义时才使用可空类型
2. **使用 Input 类型**：保持 mutation 参数清晰，可验证
3. **使用枚举**：表示固定集合的值
4. **使用游标分页**：比偏移量分页更高效，避免重复/跳过
5. **废弃字段**：使用 `@deprecated` 而非删除
6. **Mutation 动词前缀**：`createUser`、`updatePost`、`deleteComment`
7. **返回被修改对象**：让客户端获取最新状态

### 安全考虑

1. **输入验证**：验证所有输入参数
2. **限流**：防止滥用（可用 Persisted Queries）
3. **查询复杂度限制**：防止恶意查询
4. **深度限制**：防止嵌套过深的查询

```typescript
// Apollo Server 安全配置
const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    createApollo4QueryComplexityLimitRule(1000)
  ],
  plugins: [
    {
      async requestDidStart({ request, context }) {
        return {
          async didResolveOperation({ operation }) {
            // 检查操作类型和复杂度
          }
        };
      }
    }
  ]
});
```

---

## GraphQL Federation

Federation（联合）是一种分布式GraphQL架构，允许不同团队独立开发、部署各自的服务子图（subgraph），最终组成统一的超图（supergraph）。[^4]

### Federation vs 单体Schema

| 特性 | 单体Schema | Federation |
|------|-----------|------------|
| 团队协作 | 冲突频繁 | 独立开发 |
| 部署模式 | 统一部署 | 分、子图独立部署 |
| 所有权 | 集中管理 | 域边界清晰 |
| 扩展性 | 受限于单体 | 线性扩展 |

### 核心概念

#### Entity（实体）

Entity是跨子图共享的类型，使用`@key`标识：

```graphql
# Users子图定义User实体
type User @key(fields: "id") {
  id: ID!
  name: String!
  email: String!
}
```

```graphql
# Orders子图引用User实体
type Order @key(fields: "id") {
  id: ID!
  user: User!
  total: Int!
}
```

#### 指令系统

| 指令 | 作用 | 使用场景 |
|------|------|---------|
| `@key` | 声明实体主键 | 子图引用其他子图的实体 |
| `@external` | 标记外部字段 | 跨子图依赖 |
| `@requires` | 声明依赖字段 | 本子图需要但不由本子图解析 |
| `@provides` | 声明提供字段 | 优化查询计划 |

### Apollo Federation 架构

```
┌─────────────────────────────────────────────────────────┐
│                    Router / Gateway                      │
│              (超图组合 + 查询计划生成)                     │
└────────────────┬────────────────────────────────────────┘
                 │
     ┌───────────┼───────────┐
     ▼           ▼           ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│  Users  │ │ Orders  │ │ Products│
│ Subgraph│ │ Subgraph│ │ Subgraph│
└─────────┘ └─────────┘ └─────────┘
```

### 子图设计原则

#### 1. 域驱动边界

```graphql
# Accounts子图 - 用户账户领域
type Query {
  me: User
  user(id: ID!): User
}

type User @key(fields: "id") {
  id: ID!
  email: String!
  passwordHash: String!
  createdAt: DateTime!
}

type Mutation {
  updateProfile(input: UpdateProfileInput!): User
}
```

```graphql
# Reviews子图 - 评论领域
type Query {
  reviews(productId: ID!): [Review!]!
}

type Review @key(fields: "id") {
  id: ID!
  productId: ID!
  authorId: ID!
  rating: Int!
  comment: String!
}
```

#### 2. 避免循环依赖

❌ **错误**：Orders引用Reviews，Reviews也引用Orders

✅ **正确**：通过顶层Query或事件驱动解耦

#### 3. Value Type共享

共享简单类型（枚举、输入类型）：

```graphql
# Orders子图
enum OrderStatus {
  PENDING
  CONFIRMED
  SHIPPED
  DELIVERED
}

# Reviews子图可以直接使用OrderStatus枚举
type Query {
  ordersByStatus(status: OrderStatus!): [Order!]!
}
```

### 查询计划（Query Plan）

Router自动生成的跨子图查询计划：

```graphql
# 客户端查询
query GetUserWithOrders {
  user(id: "1") {
    name
    orders {
      total
      items {
        name
      }
    }
  }
}
```

```json
// 自动生成的查询计划
{
  "queryPlan": {
    "sequence": [
      {
        "serviceName": "users",
        "operation": "query { _entities(...) { name } }"
      },
      {
        "serviceName": "orders", 
        "operation": "query { ordersByUserId(userId: $userId) { total items { name } } }"
      }
    ]
  }
}
```

### Federation 2.0 新特性

| 特性 | 说明 |
|------|------|
| `@shareable` | 允许多个子图解析同一字段 |
| `@inaccessible` | 隐藏内部字段 |
| `@override` | 字段解析权转移 |
| `@link` | 显式声明依赖关系 |

```graphql
# 使用@shareable
extend schema @link(url: "https://specs.apollo.dev/federation/v2.3")

type Product @key(fields: "upc") {
  upc: String!
  name: String @shareable  # 多个子图可解析
  price: Int @external     # 外部子图提供
}
```

### 迁移策略

1. **阶段一**：创建新子图，保留单体Schema
2. **阶段二**：逐步将类型迁移到子图
3. **阶段三**：旧单体仅作代理，最终废弃

---

## 工具链

### 主流 GraphQL 服务器

- **Apollo Server**：最流行的 GraphQL 服务器
- **GraphQL Yoga**：轻量级、功能丰富
- **Prisma**：数据库 GraphQL 映射

### Federation工具

- **Apollo Router**：高性能Router实现
- **Rover CLI**：Schema校验和发布
- **Apollo Studio**：Schema注册与监控

### 客户端库

- **Apollo Client**：功能完整的 GraphQL 客户端
- **urql**：轻量级选择
- **Relay**：Facebook 出品的 React GraphQL 客户端

---

## 参考资料

[^1]: [GraphQL 官网](https://graphql.org/)
[^2]: [Apollo GraphQL 文档](https://www.apollographql.com/docs/)
[^3]: [GraphQL 最佳实践](https://graphql.org/learn/best-practices/)
[^4]: [Apollo Federation 文档](https://www.apollographql.com/docs/federation/)
