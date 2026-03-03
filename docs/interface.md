# KMM (Knowledge Memory Management) 对外接口文档

## 文档概述

**项目名称**: KMM - 知识记忆管理系统
**版本**: v1.0
**文档生成时间**: 2026-03-03
**维护者**: AISF团队

### 目录

1. [项目简介](#项目简介)
2. [技术架构](#技术架构)
3. [C++服务接口](#c服务接口)
4. [Python服务接口](#python服务接口)
5. [接口规范](#接口规范)
6. [错误码说明](#错误码说明)
7. [附录](#附录)

---

## 项目简介

KMM (Knowledge Memory Management) 是一个为个性化AI助手提供全面记忆存储、管理和检索能力的系统。该系统采用混合语言架构（C++核心服务 + Python AI服务），基于华为云微服务框架构建，支持多种记忆类型的管理和智能检索。

### 核心功能

- **用户记忆管理**: 存储和管理用户的个性化记忆信息
- **对话历史管理**: 记录用户与系统的交互历史
- **情景记忆提取**: 从对话中提取情景化记忆片段
- **问答短记忆**: 管理问答对形式的短时记忆
- **用户画像管理**: 构建和维护用户个性化画像
- **智能检索**: 基于向量和语义的智能记忆检索

---

## 技术架构

### 整体架构

```
┌─────────────────────────────────────────┐
│           客户端应用                      │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│         API Gateway / 负载均衡           │
└──────┬───────────────────────┬───────────┘
       │                       │
┌──────▼──────────┐    ┌──────▼──────────┐
│  KMMService     │    │ MemoryService   │
│  (C++ 核心服务)  │    │ (Python AI服务)  │
│  端口: 27203    │    │ 端口: 8888      │
└──────┬──────────┘    └──────┬──────────┘
       │                       │
       └───────────┬───────────┘
                   │
       ┌───────────▼─────────────────────┐
       │      基础设施层                  │
       │  GaussDB (向量数据库)           │
       │  GraphDB (图数据库)             │
       │  LLM Service                   │
       │  Embedding Service             │
       │  Rerank Service                │
       └─────────────────────────────────┘
```

### 技术栈

| 类别 | 技术 | 版本 |
|------|------|------|
| 编程语言 | C++ | C++17 |
| 编程语言 | Python | 3.8+ |
| HTTP框架 | UMOMFrmCpp | - |
| 构建工具 | CMake | 3.17+ |
| 数据库 | GaussDB | 8.0+ |
| 向量搜索 | DiskANN | - |
| 容器化 | Docker | - |
| 编排 | Kubernetes | - |
| AI模型 | Qwen3-32B | - |

### 服务端口

| 服务 | 端口 | 说明 |
|------|------|------|
| KMMService | 27200 | 主服务端口 |
| KMMService | 27201 | 业务健康检查端口 |
| KMMService | 27202 | 健康检查端口 |
| KMMService | 27203 | HTTP服务端口 |
| MemoryService | 8888 | Python服务端口 |
| Embedding Service | 2025 | 嵌入服务端口 |

---

## C++服务接口

> **服务地址**: `http://<host>:27203`
> **协议**: HTTP/HTTPS
> **Content-Type**: `application/json`
> **字符编码**: UTF-8

### 1. 添加用户记忆

**接口描述**: 向系统添加用户记忆信息，支持多种记忆类型。

**请求信息**:

```
POST /kmm/v1/user/memory/add
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| sessionId | String | 否 | 会话ID | "session_123" |
| topic | String | 否 | 主题 | "电影" |
| subTopic | String | 否 | 子主题 | "科幻" |
| content | String | 是 | 记忆内容 | "用户喜欢《星际穿越》" |
| entity | String | 否 | 实体信息 | "星际穿越" |
| label | String | 否 | 标签 | "偏好" |
| metadata | String | 否 | 元数据（JSON格式） | "{\"source\":\"chat\"}" |
| agent | String | 否 | 代理类型 | "policy_movie" |

**请求示例**:

```json
{
  "userId": "user_001",
  "sessionId": "session_123",
  "topic": "电影",
  "subTopic": "科幻",
  "content": "用户喜欢《星际穿越》",
  "entity": "星际穿越",
  "label": "偏好",
  "metadata": "{\"source\":\"chat\"}",
  "agent": "policy_movie"
}
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "uuid": "mem_550e8400-e29b-41d4-a716-446655440000",
    "userId": "user_001",
    "createDate": "2026-03-03 10:30:00"
  }
}
```

**响应模式**: SYNC (同步)

**备注**: 
- `userId` 最大长度: 64
- `topic` 最大长度: 128
- `content` 最大长度: 4096
- 该接口会根据不同的 `agent` 类型应用不同的记忆策略

---

### 2. 查询用户记忆

**接口描述**: 根据查询条件检索用户记忆，支持向量相似度搜索和语义检索。

**请求信息**:

```
POST /kmm/v1/user/memories/get
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| query | String | 是 | 查询内容 | "用户喜欢什么电影" |
| filters | String | 否 | 过滤条件（JSON格式） | "{\"topic\":\"电影\"}" |
| topK | Integer | 否 | 返回结果数量 | 5 |
| agent | String | 否 | 代理类型 | "policy_movie" |
| rewriteQuery | String | 否 | 重写后的查询 | "电影偏好" |
| label | String | 否 | 标签过滤 | "偏好" |
| metadata | String | 否 | 元数据过滤 | "{\"source\":\"chat\"}" |
| taskId | String | 否 | 任务ID | "task_001" |
| useCoreMemory | Boolean | 否 | 是否使用核心记忆 | true |
| useEpisodicMemory | Boolean | 否 | 是否使用情景记忆 | true |
| useSemanticMemory | Boolean | 否 | 是否使用语义记忆 | false |
| overTime | Integer | 否 | 超时时间（毫秒） | 3000 |
| prefThreshold | Double | 否 | 偏好阈值 | 0.5 |

**请求示例**:

```json
{
  "userId": "user_001",
  "query": "用户喜欢什么电影",
  "filters": "{\"topic\":\"电影\"}",
  "topK": 5,
  "agent": "policy_movie",
  "useCoreMemory": true,
  "useEpisodicMemory": true,
  "overTime": 3000
}
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "memories": [
      {
        "uuid": "mem_550e8400-e29b-41d4-a716-446655440000",
        "userId": "user_001",
        "topic": "电影",
        "subTopic": "科幻",
        "content": "用户喜欢《星际穿越》",
        "distance": 0.15,
        "createDate": "2026-03-03 10:30:00"
      },
      {
        "uuid": "mem_660e8400-e29b-41d4-a716-446655440001",
        "userId": "user_001",
        "topic": "电影",
        "subTopic": "科幻",
        "content": "用户推荐《盗梦空间》",
        "distance": 0.25,
        "createDate": "2026-03-02 15:20:00"
      }
    ],
    "total": 2
  }
}
```

**响应模式**: ASYNC (异步)

**备注**:
- 查询结果按相似度排序（distance越小越相似）
- 支持多种记忆类型的混合检索
- `overTime` 参数控制查询超时时间

---

### 3. 添加对话历史

**接口描述**: 记录用户与系统的单条对话历史。

**请求信息**:

```
POST /kmm/v1/user/memory/history/add
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| sessionId | String | 是 | 会话ID | "session_123" |
| role | String | 是 | 角色(user/system) | "user" |
| content | String | 是 | 对话内容 | "帮我推荐一部科幻电影" |
| createDate | String | 否 | 创建时间 | "2026-03-03 10:30:00" |

**请求示例**:

```json
{
  "userId": "user_001",
  "sessionId": "session_123",
  "role": "user",
  "content": "帮我推荐一部科幻电影",
  "createDate": "2026-03-03 10:30:00"
}
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "id": "conv_550e8400-e29b-41d4-a716-446655440000",
    "userId": "user_001",
    "sessionId": "session_123"
  }
}
```

**响应模式**: SYNC (同步)

**备注**:
- `userId` 最大长度: 64
- `sessionId` 最大长度: 128
- `content` 最大长度: 8192
- `role` 只能是 "user" 或 "system"

---

### 4. 批量添加对话历史

**接口描述**: 批量记录多条对话历史，用于导入或批量场景。

**请求信息**:

```
POST /kmm/v1/user/memory/history/batch
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| conversations | Array | 是 | 对话数组 | 见下方示例 |

**conversations数组元素**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| userId | String | 是 | 用户ID |
| sessionId | String | 是 | 会话ID |
| role | String | 是 | 角色(user/system) |
| content | String | 是 | 对话内容 |
| createDate | String | 否 | 创建时间 |

**请求示例**:

```json
{
  "conversations": [
    {
      "userId": "user_001",
      "sessionId": "session_123",
      "role": "user",
      "content": "帮我推荐一部科幻电影",
      "createDate": "2026-03-03 10:30:00"
    },
    {
      "userId": "user_001",
      "sessionId": "session_123",
      "role": "system",
      "content": "我推荐《星际穿越》，这是一部非常优秀的科幻电影",
      "createDate": "2026-03-03 10:30:05"
    }
  ]
}
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "successCount": 2,
    "failedCount": 0,
    "total": 2
  }
}
```

**响应模式**: SYNC (同步)

**备注**: 批量操作建议单次不超过100条

---

### 5. 查询对话历史

**接口描述**: 查询用户的对话历史记录。

**请求信息**:

```
POST /kmm/v1/user/memories/history/get
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| sessionId | String | 否 | 会话ID | "session_123" |
| role | String | 否 | 角色过滤 | "user" |
| startTime | String | 否 | 开始时间 | "2026-03-01 00:00:00" |
| endTime | String | 否 | 结束时间 | "2026-03-03 23:59:59" |
| limit | Integer | 否 | 返回数量限制 | 100 |
| offset | Integer | 否 | 偏移量 | 0 |

**请求示例**:

```json
{
  "userId": "user_001",
  "sessionId": "session_123",
  "limit": 50,
  "offset": 0
}
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "conversations": [
      {
        "id": "conv_550e8400-e29b-41d4-a716-446655440000",
        "userId": "user_001",
        "sessionId": "session_123",
        "role": "user",
        "content": "帮我推荐一部科幻电影",
        "createDate": "2026-03-03 10:30:00",
        "extractCount": 0
      },
      {
        "id": "conv_660e8400-e29b-41d4-a716-446655440001",
        "userId": "user_001",
        "sessionId": "session_123",
        "role": "system",
        "content": "我推荐《星际穿越》",
        "createDate": "2026-03-03 10:30:05",
        "extractCount": 1
      }
    ],
    "total": 2
  }
}
```

**响应模式**: ASYNC (异步)

**备注**: 结果按时间倒序排列

---

### 6. 获取用户输入

**接口描述**: 获取用户的输入事件信息。

**请求信息**:

```
POST /kmm/v1/user/memory/user_input/get
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| eventId | String | 否 | 事件ID | "event_001" |
| startTime | String | 否 | 开始时间 | "2026-03-01 00:00:00" |
| endTime | String | 否 | 结束时间 | "2026-03-03 23:59:59" |
| limit | Integer | 否 | 返回数量限制 | 50 |

**请求示例**:

```json
{
  "userId": "user_001",
  "limit": 50
}
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "events": [
      {
        "eventId": "event_001",
        "userId": "user_001",
        "eventType": "query",
        "content": "查询电影推荐",
        "createDate": "2026-03-03 10:30:00"
      }
    ],
    "total": 1
  }
}
```

**响应模式**: ASYNC (异步)

---

### 7. 处理用户输入

**接口描述**: 处理用户输入事件，触发记忆提取和存储流程。

**请求信息**:

```
POST /kmm/v1/user/memory/user_input
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| sessionId | String | 否 | 会话ID | "session_123" |
| input | String | 是 | 用户输入内容 | "我喜欢科幻电影" |
| eventType | String | 否 | 事件类型 | "query" |
| metadata | String | 否 | 元数据 | "{\"source\":\"voice\"}" |

**请求示例**:

```json
{
  "userId": "user_001",
  "sessionId": "session_123",
  "input": "我喜欢科幻电影",
  "eventType": "query",
  "metadata": "{\"source\":\"voice\"}"
}
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "eventId": "event_550e8400-e29b-41d4-a716-446655440000",
    "status": "processing"
  }
}
```

**响应模式**: ASYNC (异步)

**备注**: 该接口会触发异步的记忆提取和处理流程

---

### 8. 获取记忆列表

**接口描述**: 获取用户的记忆列表，支持分页和过滤。

**请求信息**:

```
POST /kmm/v1/user/memory/query
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| memoryType | String | 否 | 记忆类型 | "user_fact_mem" |
| topic | String | 否 | 主题过滤 | "电影" |
| startTime | String | 否 | 开始时间 | "2026-03-01 00:00:00" |
| endTime | String | 否 | 结束时间 | "2026-03-03 23:59:59" |
| limit | Integer | 否 | 返回数量限制 | 50 |
| offset | Integer | 否 | 偏移量 | 0 |

**请求示例**:

```json
{
  "userId": "user_001",
  "memoryType": "user_fact_mem",
  "topic": "电影",
  "limit": 50,
  "offset": 0
}
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "memories": [
      {
        "uuid": "mem_550e8400-e29b-41d4-a716-446655440000",
        "userId": "user_001",
        "topic": "电影",
        "subTopic": "科幻",
        "content": "用户喜欢《星际穿越》",
        "createDate": "2026-03-03 10:30:00"
      }
    ],
    "total": 1,
    "limit": 50,
    "offset": 0
  }
}
```

**响应模式**: SYNC (同步)

---

### 9. 处理自定义输入

**接口描述**: 处理自定义输入，用于特殊场景的记忆管理。

**请求信息**:

```
POST /kmm/v1/user/memory/custom_input
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| category | String | 是 | 自定义分类 | "preference" |
| customInfo | String | 是 | 自定义信息 | "{\"genre\":\"科幻\",\"director\":\"诺兰\"}" |

**请求示例**:

```json
{
  "userId": "user_001",
  "category": "preference",
  "customInfo": "{\"genre\":\"科幻\",\"director\":\"诺兰\"}"
}
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "uuid": "mem_550e8400-e29b-41d4-a716-446655440000",
    "category": "preference"
  }
}
```

**响应模式**: SYNC (同步)

---

### 10. 快速路径检索

**接口描述**: 快速检索路径，用于高频查询场景，优化性能。

**请求信息**:

```
POST /kmm/v1/user/memory/get/fast
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| query | String | 是 | 查询内容 | "电影偏好" |
| type | String | 否 | 检索类型 | "preference" |
| topK | Integer | 否 | 返回数量 | 3 |
| taskId | String | 否 | 任务ID | "task_001" |

**请求示例**:

```json
{
  "userId": "user_001",
  "query": "电影偏好",
  "type": "preference",
  "topK": 3
}
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "memories": [
      {
        "uuid": "mem_550e8400-e29b-41d4-a716-446655440000",
        "content": "用户喜欢科幻电影",
        "distance": 0.1
      }
    ],
    "total": 1
  }
}
```

**响应模式**: SYNC (同步)

**备注**: 该接口针对高频查询优化，响应速度快

---

### 11. 短记忆管理

**接口描述**: 管理用户的短时记忆，支持添加、更新、删除等操作。

**请求信息**:

```
POST /kmm/v1/user/memory/short/manager
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| operation | String | 是 | 操作类型(add/update/delete) | "add" |
| memoryId | String | 否 | 记忆ID（update/delete时必填） | "mem_001" |
| content | String | 否 | 记忆内容（add/update时必填） | "用户正在询问天气" |
| ttl | Integer | 否 | 生存时间（秒） | 3600 |

**请求示例**:

```json
{
  "userId": "user_001",
  "operation": "add",
  "content": "用户正在询问天气",
  "ttl": 3600
}
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "memoryId": "mem_550e8400-e29b-41d4-a716-446655440000",
    "operation": "add",
    "ttl": 3600
  }
}
```

**响应模式**: ASYNC (异步)

---

### 12. 查询分类记忆

**接口描述**: 查询用户分类后的画像记忆。

**请求信息**:

```
GET /kmm/v1/user/memory/persona/classify/query
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| personaType | String | 否 | 画像类型 | "movie_lover" |
| limit | Integer | 否 | 返回数量限制 | 20 |

**请求示例**:

```
GET /kmm/v1/user/memory/persona/classify/query?userId=user_001&personaType=movie_lover&limit=20
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "personas": [
      {
        "personaId": "persona_001",
        "userId": "user_001",
        "personaType": "movie_lover",
        "tags": ["科幻", "诺兰"],
        "confidence": 0.85,
        "createDate": "2026-03-03 10:30:00"
      }
    ],
    "total": 1
  }
}
```

**响应模式**: SYNC (同步)

---

### 13. QA批量添加

**接口描述**: 批量添加问答对形式的短时记忆。

**请求信息**:

```
POST /kmm/v1/user/memory/short/qa_mgr/operate
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| qaList | Array | 是 | 问答对数组 | 见下方示例 |

**qaList数组元素**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| originalQA | String | 是 | 原始问答对 |
| abstractQA | String | 是 | 抽象问答对 |
| abstract | String | 是 | 摘要 |
| rewriteQuestion | String | 是 | 重写问题 |
| type | String | 是 | 类型 |
| keys | String | 是 | 关键词 |

**请求示例**:

```json
{
  "userId": "user_001",
  "qaList": [
    {
      "originalQA": "Q:你喜欢什么电影?A:我喜欢科幻电影",
      "abstractQA": "Q:电影偏好?A:科幻",
      "abstract": "用户偏好科幻电影",
      "rewriteQuestion": "电影类型偏好",
      "type": "preference",
      "keys": "科幻,电影,偏好"
    }
  ]
}
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "successCount": 1,
    "failedCount": 0,
    "total": 1
  }
}
```

**响应模式**: ASYNC (异步)

---

### 14. QA获取TopK

**接口描述**: 获取与查询最相关的TopK个问答对。

**请求信息**:

```
POST /kmm/v1/user/memory/short/qa_mgr/get
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| query | String | 是 | 查询内容 | "你喜欢什么电影" |
| topK | Integer | 否 | 返回数量 | 3 |

**请求示例**:

```json
{
  "userId": "user_001",
  "query": "你喜欢什么电影",
  "topK": 3
}
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "qaList": [
      {
        "uuid": "qa_550e8400-e29b-41d4-a716-446655440000",
        "userId": "user_001",
        "originalQA": "Q:你喜欢什么电影?A:我喜欢科幻电影",
        "abstractQA": "Q:电影偏好?A:科幻",
        "distance": 0.1,
        "createDate": "2026-03-03 10:30:00"
      }
    ],
    "total": 1
  }
}
```

**响应模式**: ASYNC (异步)

---

### 15. 生成关联索引

**接口描述**: 为问答对生成关联索引，支持相关问答的发现。

**请求信息**:

```
POST /kmm/v1/user/memory/short/qa_mgr/gen_related_index
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| qaId | String | 是 | 问答ID | "qa_001" |

**请求示例**:

```json
{
  "userId": "user_001",
  "qaId": "qa_550e8400-e29b-41d4-a716-446655440000"
}
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "qaId": "qa_550e8400-e29b-41d4-a716-446655440000",
    "relatedCount": 3,
    "status": "generated"
  }
}
```

**响应模式**: ASYNC (异步)

---

### 16. 查询关联索引

**接口描述**: 查询问答对的关联索引。

**请求信息**:

```
POST /kmm/v1/user/memory/short/qa_mgr/query_related_index
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| qaId | String | 是 | 问答ID | "qa_001" |
| topK | Integer | 否 | 返回数量 | 5 |

**请求示例**:

```json
{
  "userId": "user_001",
  "qaId": "qa_550e8400-e29b-41d4-a716-446655440000",
  "topK": 5
}
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "relatedQAs": [
      {
        "qaId": "qa_660e8400-e29b-41d4-a716-446655440001",
        "similarity": 0.85,
        "originalQA": "Q:推荐一部科幻电影?A:《星际穿越》"
      }
    ],
    "total": 1
  }
}
```

**响应模式**: ASYNC (异步)

---

### 17. 轻量级检索

**接口描述**: 轻量级的记忆检索接口，用于快速获取相关记忆。

**请求信息**:

```
POST /kmm/v1/user/memory/light_retrieval
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| query | String | 是 | 查询内容 | "电影" |
| topK | Integer | 否 | 返回数量 | 3 |

**请求示例**:

```json
{
  "userId": "user_001",
  "query": "电影",
  "topK": 3
}
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "memories": [
      {
        "content": "用户喜欢科幻电影",
        "type": "preference",
        "relevance": 0.9
      }
    ],
    "total": 1
  }
}
```

**响应模式**: ASYNC (异步)

**备注**: 该接口用于快速检索，不进行复杂的语义分析

---

## Python服务接口

> **服务地址**: `http://<host>:8888`
> **协议**: HTTP/HTTPS
> **Content-Type**: `application/json`
> **字符编码**: UTF-8

### 1. 添加用户

**接口描述**: 添加新用户到系统。

**请求信息**:

```
POST /mirix/v1/user/add
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| userId | String | 是 | 用户ID | "user_001" |
| userName | String | 否 | 用户名 | "张三" |
| metadata | Object | 否 | 元数据 | {"age": 30} |

**请求示例**:

```json
{
  "userId": "user_001",
  "userName": "张三",
  "metadata": {
    "age": 30
  }
}
```

**响应示例**:

```json
{
  "retCode": 0,
  "retMsg": "add user success",
  "posId": ""
}
```

**备注**: 该接口功能待完善

---

### 2. 提取情景记忆

**接口描述**: 从用户输入中提取情景记忆，异步处理。

**请求信息**:

```
POST /mirix/v1/user/episodic/memories/add
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| user_id | String | 是 | 用户ID（Header） | "user_001" |
| text | String | 是 | 待提取的文本 | "昨天我和朋友在星巴克喝了咖啡" |
| timestamp | Long | 否 | 时间戳（毫秒） | 1709107200000 |
| metadata | Object | 否 | 元数据 | {"source": "chat"} |

**请求示例**:

```json
{
  "text": "昨天我和朋友在星巴克喝了咖啡",
  "timestamp": 1709107200000,
  "metadata": {
    "source": "chat"
  }
}
```

**请求头**:

```
user_id: user_001
```

**响应示例**:

```json
{
  "code": 202,
  "message": "start extract episodic memory success"
}
```

**备注**: 
- 该接口为异步处理，返回202表示已接受请求
- 实际提取结果需要通过其他接口查询
- 会自动解析相对时间（如"昨天"）

---

### 3. 添加记忆

**接口描述**: 添加记忆到系统，支持多Agent并行处理。

**请求信息**:

```
POST /mirix/v1/user/memories/add
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| user_id | String | 是 | 用户ID（Header） | "user_001" |
| conversations | Array | 是 | 对话数组 | 见下方示例 |
| source | String | 否 | 来源 | "chat" |

**conversations数组元素**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| role | String | 是 | 角色(user/system) |
| content | String | 是 | 对话内容 |
| timestamp | Long | 否 | 时间戳（毫秒） |

**请求示例**:

```json
{
  "conversations": [
    {
      "role": "user",
      "content": "我喜欢科幻电影",
      "timestamp": 1709107200000
    },
    {
      "role": "system",
      "content": "好的，我记住了",
      "timestamp": 1709107201000
    }
  ],
  "source": "chat"
}
```

**请求头**:

```
user_id: user_001
```

**响应示例**:

```json
{
  "retCode": 0,
  "retMsg": "add memory success",
  "posId": ""
}
```

**备注**: 
- 支持多个Agent并行处理（CoreMemoryAgent、EpisodicMemoryAgent、SemanticMemoryAgent等）
- 会自动解析对话中的相对时间
- 单个Agent超时时间为10000秒

---

### 4. 获取相关记忆

**接口描述**: 获取与查询相关的记忆。

**请求信息**:

```
GET /mirix/v1/user/memories/relevant/get
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| user_id | String | 是 | 用户ID（Header） | "user_001" |
| query | String | 是 | 查询内容 | "电影" |
| topK | Integer | 否 | 返回数量 | 5 |

**请求示例**:

```
GET /mirix/v1/user/memories/relevant/get?query=电影&topK=5
```

**请求头**:

```
user_id: user_001
```

**响应示例**:

```json
{
  "retCode": 0,
  "retMsg": "get memory success",
  "data": [
    {
      "content": "用户喜欢科幻电影",
      "type": "preference",
      "relevance": 0.9
    }
  ]
}
```

**备注**: 该接口功能待完善

---

### 5. 查询画像记忆

**接口描述**: 按记忆类型查询用户画像记忆。

**请求信息**:

```
GET /mirix/v1/user/memories/persona/query
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| user_id | String | 是 | 用户ID（Header） | "user_001" |
| memory_type | String | 否 | 记忆类型 | "preference" |

**请求示例**:

```
GET /mirix/v1/user/memories/persona/query?memory_type=preference
```

**请求头**:

```
user_id: user_001
```

**响应示例**:

```json
{
  "retCode": 0,
  "retMsg": "query success",
  "data": [
    {
      "type": "preference",
      "content": "科幻电影",
      "confidence": 0.85
    }
  ]
}
```

**备注**: 该接口功能待完善

---

### 6. 召回记忆

**接口描述**: 根据用户查询召回相关记忆。

**请求信息**:

```
GET /mirix/v1/user/memories/recall
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| user_id | String | 是 | 用户ID（Header） | "user_001" |
| query | String | 是 | 查询内容 | "电影" |
| topK | Integer | 否 | 返回数量 | 5 |

**请求示例**:

```
GET /mirix/v1/user/memories/recall?query=电影&topK=5
```

**请求头**:

```
user_id: user_001
```

**响应示例**:

```json
{
  "retCode": 0,
  "retMsg": "recall success",
  "data": [
    {
      "content": "用户喜欢科幻电影",
      "type": "episodic",
      "score": 0.9
    }
  ]
}
```

**备注**: 该接口功能待完善

---

### 7. 第三方LLM调用

**接口描述**: 调用第三方大语言模型服务。

**请求信息**:

```
POST /mirix/v1/user/llm/third
```

**请求参数**:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| user_id | String | 是 | 用户ID（Header） | "user_001" |
| query | String | 是 | 用户查询 | "推荐一部科幻电影" |
| prompt_content | String | 否 | 提示词内容 | "你是一个电影推荐专家" |

**请求示例**:

```json
{
  "query": "推荐一部科幻电影",
  "prompt_content": "你是一个电影推荐专家"
}
```

**请求头**:

```
user_id: user_001
```

**响应示例**:

```json
{
  "retCode": 0,
  "retMsg": "call llm third success",
  "res": "我推荐《星际穿越》，这是一部优秀的科幻电影"
}
```

**备注**: 
- 默认使用阿里云DashScope的Qwen3-32B模型
- 可配置第三方模型端点和API密钥
- 超时时间为120秒

---

### 8. 动态重载路由

**接口描述**: 动态重载路由配置，无需重启服务。

**请求信息**:

```
POST /mirix/v1/reload-routes
```

**请求参数**: 无

**请求示例**:

```bash
curl -X POST http://localhost:8888/mirix/v1/reload-routes
```

**响应示例**:

```json
{
  "message": "Routes reloaded successfully"
}
```

**备注**: 
- 该接口用于开发调试和生产环境热更新
- 路由配置文件路径: `MemoryService/config/config.ini`

---

## 接口规范

### 通用请求规范

1. **Content-Type**: 所有接口请求和响应均使用 `application/json`
2. **字符编码**: 统一使用 UTF-8 编码
3. **时间格式**: 使用 `YYYY-MM-DD HH:mm:ss` 格式或毫秒时间戳
4. **ID格式**: 使用UUID格式（如 `550e8400-e29b-41d4-a716-446655440000`）

### 通用响应规范

#### 成功响应

```json
{
  "code": 200,
  "message": "success",
  "data": {
    // 具体数据
  }
}
```

#### 错误响应

```json
{
  "code": 400,
  "message": "error message",
  "data": null
}
```

### 线程池配置

| 线程池 | 用途 | 线程数 |
|--------|------|--------|
| router_mem_query | 查询接口 | 32 |
| router_normal | 普通接口 | 4 |
| Python服务 | 所有接口 | 4 |

### 响应模式说明

- **SYNC (同步)**: 请求处理完成后立即返回结果
- **ASYNC (异步)**: 请求提交后立即返回，实际处理在后台进行

---

## 错误码说明

### 通用错误码

| 错误码 | 说明 | 处理建议 |
|--------|------|----------|
| 200 | 成功 | - |
| 400 | 请求参数错误 | 检查请求参数格式和内容 |
| 404 | 资源不存在 | 检查请求路径和资源ID |
| 500 | 服务器内部错误 | 联系技术支持 |
| 503 | 服务不可用 | 稍后重试 |

### 业务错误码

| 错误码 | 说明 | 处理建议 |
|--------|------|----------|
| 1001 | 用户ID为空 | 提供有效的用户ID |
| 1002 | 记忆内容为空 | 提供有效的记忆内容 |
| 1003 | 记忆不存在 | 检查记忆ID或创建新记忆 |
| 1004 | 参数长度超限 | 缩短参数内容 |
| 1005 | 记忆类型错误 | 使用正确的记忆类型 |
| 2001 | 数据库连接失败 | 检查数据库配置 |
| 2002 | 数据库操作失败 | 联系技术支持 |
| 3001 | LLM服务调用失败 | 检查LLM服务配置 |
| 3002 | Embedding服务调用失败 | 检查Embedding服务配置 |
| 3003 | Rerank服务调用失败 | 检查Rerank服务配置 |
| 4001 | 请求超时 | 增加超时时间或优化查询 |
| 4002 | 并发限制 | 降低请求频率 |

### Python服务错误码

| 错误码 | 说明 | 处理建议 |
|--------|------|----------|
| 0 | 成功 | - |
| 10 | LLM调用失败 | 检查LLM服务配置和网络连接 |
| 202 | 异步处理已接受 | 等待处理完成 |

---

## 附录

### A. 数据库表映射

| 记忆类型 | 表名 | 说明 |
|----------|------|------|
| user_conversion | tbl_user_conversation | 用户对话历史 |
| user_fact_mem | tbl_user_fact | 用户事实记忆 |
| user_theme_mem | tbl_user_theme | 用户主题记忆 |
| user_profile | tbl_user_profile | 用户画像 |
| episodic_memory | tbl_episodic_memory | 情景记忆 |
| semantic_memory | tbl_semantic_memory | 语义记忆 |
| core_memory | tbl_core_memory | 核心记忆 |
| qa_short_memory | tbl_qa_short_memory | 问答短记忆 |
| user_classified_persona | tbl_user_classified_persona | 用户分类画像 |

### B. 记忆策略映射

| 场景 | 策略名称 | 说明 |
|------|----------|------|
| 电影 | policy_movie | 电影推荐策略 |
| 旅行 | policy_travel | 旅行规划策略 |
| 美食 | policy_agent | 美食推荐策略 |
| 购物 | policy_agent | 购物推荐策略 |
| 用户输入 | userInput | 用户输入处理策略 |
| 默认 | policy_common | 通用策略 |

### C. 配置文件路径

| 配置文件 | 路径 | 说明 |
|----------|------|------|
| kmm_config.json | deployment/conf/kmm_config.json | KMM主配置 |
| matemindsdk.json | deployment/conf/matemindsdk.json | MateMind SDK配置 |
| product.json | deployment/conf/product.json | 产品配置 |
| config.ini | MemoryService/config/config.ini | Python服务配置 |
| serviceconfig.yaml | deployment/serviceconfig/serviceconfig.yaml | 服务配置 |

### D. 环境变量

| 变量名 | 说明 |
|--------|------|
| KMM_DEBUG_INFO | 调试信息开关 |
| PYTHON_SERVICE_URL | Python服务URL |
| LLM_MODEL | LLM模型名称 |
| LLM_MODEL_SERVICE_NAME | LLM服务名称 |
| MM_GATE_WAY_SERVICE_NAME | 记忆网关服务名称 |
| EXTERNAL_MODEL_URL | 外部模型URL |
| EXTERNAL_MODEL_API_key | 外部模型API密钥 |
| RERANK_MODEL | 重排序模型 |

### E. 接口调用示例

#### Python示例

```python
import requests
import json

# 添加用户记忆
url = "http://localhost:27203/kmm/v1/user/memory/add"
headers = {"Content-Type": "application/json"}
data = {
    "userId": "user_001",
    "topic": "电影",
    "content": "用户喜欢《星际穿越》",
    "agent": "policy_movie"
}
response = requests.post(url, headers=headers, data=json.dumps(data))
print(response.json())
```

#### cURL示例

```bash
# 查询用户记忆
curl -X POST http://localhost:27203/kmm/v1/user/memories/get \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "user_001",
    "query": "用户喜欢什么电影",
    "topK": 5
  }'
```

### F. 性能建议

1. **批量操作**: 使用批量接口减少网络开销
2. **异步接口**: 对于耗时操作使用异步接口
3. **缓存策略**: 对频繁查询的结果进行缓存
4. **分页查询**: 大数据量查询使用分页
5. **超时设置**: 合理设置超时时间，避免长时间等待

### G. 安全建议

1. **API鉴权**: 建议增加API网关鉴权机制
2. **数据加密**: 敏感数据传输使用HTTPS
3. **限流保护**: 实现接口限流和熔断机制
4. **输入验证**: 严格验证所有输入参数
5. **日志审计**: 记录所有接口调用日志

### H. 监控指标

| 指标 | 说明 | 建议阈值 |
|------|------|----------|
| 接口响应时间 | 99分位响应时间 | < 1000ms |
| 接口成功率 | 成功请求占比 | > 99% |
| 错误率 | 错误请求占比 | < 1% |
| QPS | 每秒查询数 | 根据业务需求 |
| 并发数 | 同时处理的请求数 | < 线程池大小 |

---

## 变更历史

| 版本 | 日期 | 变更内容 | 作者 |
|------|------|----------|------|
| v1.0 | 2026-03-03 | 初始版本，包含所有C++和Python接口 | AISF团队 |

---

## 联系方式

如有问题或建议，请联系：

- **项目维护者**: AISF团队
- **技术支持**: [技术支持邮箱]
- **文档反馈**: [文档反馈邮箱]

---

**文档结束**
