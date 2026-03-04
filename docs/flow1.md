# KMM (Knowledge Memory Management) 场景代码运行流程文档

## 文档信息

| 项目名称 | KMM - 知识记忆管理系统 |
|---------|----------------------|
| 文档版本 | v1.0 |
| 创建日期 | 2026-03-03 |
| 项目类型 | 混合语言项目 (C++ + Python) |
| 当前分支 | br_AISF_master_For1220POC |

---

## 目录

1. [项目概述](#1-项目概述)
2. [系统架构](#2-系统架构)
3. [场景1：用户记忆添加和存储](#场景1用户记忆添加和存储)
4. [场景2：用户记忆查询和检索](#场景2用户记忆查询和检索)
5. [场景3：对话历史管理](#场景3对话历史管理)
6. [场景4：情景记忆提取](#场景4情景记忆提取)
7. [场景5：用户画像构建](#场景5用户画像构建)
8. [场景6：问答短记忆管理](#场景6问答短记忆管理)
9. [场景7：智能检索（向量搜索+重排序）](#场景7智能检索向量搜索重排序)
10. [场景8：记忆老化清理](#场景8记忆老化清理)
11. [场景9：记忆总结优化](#场景9记忆总结优化)
12. [场景10：用户输入处理](#场景10用户输入处理)
13. [定时任务调度](#定时任务调度)
14. [策略模式应用](#策略模式应用)
15. [并发和性能优化](#并发和性能优化)
16. [外部服务调用](#外部服务调用)

---

## 1. 项目概述

### 1.1 项目简介

KMM (Knowledge Memory Management) 是一个基于AI的记忆管理系统，采用混合语言架构（C++核心服务 + Python AI服务），为个性化AI助手提供全面的记忆存储、管理和检索能力。

### 1.2 技术栈

| 类别 | 技术 | 版本 |
|------|------|------|
| 编程语言 | C++ | C++17 |
| 编程语言 | Python | 3.8+ |
| HTTP框架 | UMOMFrmCpp | - |
| 构建工具 | CMake | 3.17+ |
| 数据库 | GaussDB | 8.0+ |
| 向量搜索 | DiskANN | - |
| AI模型 | Qwen3-32B | - |

---

## 2. 系统架构

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                    客户端层                               │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│              Controller层 (路由管理)                      │
│  RouteMgr (route_mgr.cpp)                                │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│              Business层 (业务逻辑)                        │
│  - UserMemoryService (用户记忆服务)                       │
│  - MemoryCommonManager (通用记忆管理)                     │
│  - MemoryMovieManager (观影场景管理)                      │
│  - MemoryTravelManager (出行场景管理)                     │
│  - MemoryAgentManager (Agent场景管理)                     │
│  - QAShortMemoryService (问答短记忆服务)                  │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│              Domain层 (领域模型)                           │
│  - UserMemory, CoreMemory, EpisodicMemory                │
│  - SemanticMemory, UserConversation                      │
│  - *Tbl (数据库表操作类)                                 │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│          Infrastructure层 (基础设施)                       │
│  - ModelInfer (模型推理)                                  │
│  - RagMgr (向量数据库管理)                                │
│  - EmbeddingService (嵌入服务)                           │
│  - Python Memory Service (记忆提取Agent)                  │
└─────────────────────────────────────────────────────────┘
```

### 2.2 服务端口

| 服务 | 端口 | 说明 |
|------|------|------|
| KMMService | 27203 | C++ HTTP服务端口 |
| MemoryService | 8888 | Python服务端口 |

---

## 场景1：用户记忆添加和存储

### 1.1 场景描述

用户通过对话或直接输入方式添加记忆信息，系统需要智能提取、分类并存储到相应的记忆表中。

### 1.2 完整流程图

```
用户请求
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Controller层: RouteMgr::AddUserMemory                      │
│  文件: service/src/controller/route_mgr.cpp:84-90          │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  Business层: UserMemoryService::OnReceiveUserMemoryAddMsg   │
│  文件: service/src/business/memory/user_memory_service.cpp  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 参数解析 (第84-111行)
                     │    - 从请求头获取userId
                     │    - 解析请求体（content, sessionId）
                     │    - 创建UserMemory对象
                     │
                     ▼
                     │
                     │ 2. 消息验证 (第116-120行)
                     │    - 调用VerifyUserMemoryMessage()
                     │    - 检查字段长度限制
                     │
                     ▼
                     │
                     │ 3. 代理类型处理 (第122-132行)
                     │    - 提取agentType字段
                     │    - 根据agentType配置label策略
                     │
                     ▼
                     │
                     │ 4. 临时存储 (第144行)
                     │    - 调用UserTmpEpisodicTbl::Insert()
                     │    - 将记忆存入临时表
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  定时任务: UserMemoryManager::Executor                      │
│  文件: service/src/business/memory/user_memory_manager.cpp  │
│  周期: 5秒                                                 │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 查询临时记忆 (第61行)
                     │    - 从临时表查询待处理的用户记忆
                     │
                     ▼
                     │
                     │ 2. 策略分发 (第68-71行)
                     │    - 根据label分发到不同策略处理器
                     │
                     ▼
                     │
                     │ 3. 删除临时记录 (第74行)
                     │    - 处理完成后删除临时表记录
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  策略分发机制                                                │
│  文件: service/src/business/memory/user_memory_manager.cpp  │
│  策略映射 (第16-20行):                                       │
│  {                                                           │
│    "policy_movie" → MemoryMovieManager::ExtractFact,        │
│    "policy_travel" → MemoryTravelManager::ExtractFact,      │
│    "policy_agent" → MemoryAgentManager::ExtractFact,        │
│    "policy_common" → MemoryCommonManager::ExtractFact       │
│  }                                                           │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  场景策略处理 (以观影场景为例)                                │
│  MemoryMovieManager::ExtractFact                            │
│  文件: service/src/business/memory/memory_movie_manager.cpp │
│  1. 解析电影信息（导演、演员、类型等）                        │
│  2. 查询影视知识库补充信息                                    │
│  3. 插入事实记忆表 UserFactTbl::InsertUserFactTbl            │
│  4. 提取主题记忆 ExtractThemeMemory                          │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 关键代码路径

#### 1.3.1 Controller层

**文件**: `service/src/controller/route_mgr.cpp`

**方法**: `RouteMgr::AddUserMemory()` (第84-90行)

```cpp
CMFrm::COM::ServerRespHandleMode RouteMgr::AddUserMemory(
    const std::shared_ptr<CMFrm::ServiceRouter::Context> &context)
{
    LOG_INFO("AddUserMemory enter");
    UserMemoryService::GetInstance()->OnReceiveUserMemoryAddMessage(context);
    return CMFrm::COM::SYNC;
}
```

#### 1.3.2 Business层

**文件**: `service/src/business/memory/user_memory_service.cpp`

**方法**: `UserMemoryService::OnReceiveUserMemoryAddMessage()` (第82-149行)

**流程详解**:

1. **参数解析** (第84-111行):
```cpp
// 从请求头获取userId
userId = context->GetReqHeader()->GetHeader("user_id");

// 解析请求体
rapidjson::Document document;
document.Parse(body.c_str());

// 提取content和sessionId字段
const char *content = document["content"].GetString();
const char *sessionId = document["sessionId"].GetString();

// 创建UserMemory对象
auto userMemory = std::make_shared<UserMemory>(userId, sessionId, content);
```

2. **消息验证** (第116-120行):
```cpp
// 验证消息格式和长度限制
bool valid = VerifyUserMemoryMessage(userMemory);
if (!valid) {
    LOG_ERROR("Invalid user memory message");
    return;
}
```

3. **代理类型处理** (第122-132行):
```cpp
// 提取agentType字段
std::string agentType = document["agent"].GetString();

// 根据agentType配置label策略
std::string label = "";
if (agentType == "policy_movie") {
    label = "preference";
} else if (agentType == "policy_travel") {
    label = "travel";
}

userMemory->SetAgentType(agentType);
userMemory->SetLabel(label);
```

4. **临时存储** (第144行):
```cpp
// 插入临时表
UserTmpEpisodicTbl::Insert(userMemory);
```

#### 1.3.3 异步处理层

**文件**: `service/src/business/memory/user_memory_manager.cpp`

**方法**: `UserMemoryManager::Executor()` (第59-77行)

```cpp
void UserMemoryManager::Excutor()
{
    // 查询临时记忆
    auto userMemoryVec = UserTmpEpisodicTbl::QueryTempUserMemory();
    
    for (auto userMemory : userMemoryVec) {
        m_threadPool->Submit([userMemory, this]() {
            auto uuid = userMemory->GetUuid();
            
            // 策略分发
            this->DispatchUserMemoryByPolicy(userMemory);
            
            // 恢复uuid
            userMemory->SetUuid(uuid);
            
            // 删除临时记录
            UserTmpEpisodicTbl::Delete(userMemory);
        });
    }
}
```

#### 1.3.4 策略分发机制

**文件**: `service/src/business/memory/user_memory_manager.cpp`

**策略映射** (第16-20行):
```cpp
const std::unordered_map<std::string, std::function<void(std::shared_ptr<UserMemory> &)>> 
UserMemoryManager::policyMap = {
    {"policy_movie", MemoryMovieManager::ExtractFact},
    {"policy_travel", MemoryTravelManager::ExtractFact},
    {"policy_agent", MemoryAgentManager::ExtractFact},
    {"policy_common", MemoryCommonManager::ExtractFact}
};
```

**策略分发方法** (第23-35行):
```cpp
void UserMemoryManager::DispatchUserMemoryByPolicy(
    std::shared_ptr<UserMemory> &userMemory)
{
    std::string label = userMemory->GetLabel();
    
    // 根据label查找对应的策略
    auto it = policyMap.find(label);
    if (it != policyMap.end()) {
        // 调用对应的策略函数
        it->second(userMemory);
    } else {
        LOG_ERROR("Unknown policy: %s", label.c_str());
    }
}
```

#### 1.3.5 观影场景示例

**文件**: `service/src/business/memory/memory_movie_manager.cpp`

**方法**: `MemoryMovieManager::ExtractFact()` (第39-71行)

```cpp
void MemoryMovieManager::ExtractFact(std::shared_ptr<UserMemory> &userMemory)
{
    // 1. 解析电影信息
    std::string movieName = ExtractMovieName(userMemory->GetContent());
    std::string director = ExtractDirector(userMemory->GetContent());
    std::string actors = ExtractActors(userMemory->GetContent());
    std::string genre = ExtractGenre(userMemory->GetContent());
    
    // 2. 查询影视知识库补充信息
    auto movieInfo = QueryMovieKnowledgeBase(movieName);
    
    // 3. 插入事实记忆表
    auto userFact = std::make_shared<UserFact>(
        userMemory->GetUserId(),
        movieName,
        director,
        actors,
        genre
    );
    UserFactTbl::InsertUserFactTbl(userFact);
    
    // 4. 提取主题记忆
    ExtractThemeMemory(userMemory, movieInfo);
}
```

### 1.4 数据流转

```
用户输入
    │
    ▼
UserMemory对象
    │
    ▼
UserTmpEpisodicTbl (临时表)
    │
    ▼
策略分发
    │
    ▼
UserFactTbl (事实记忆表)
    │
    ▼
CoreMemoryTbl (核心记忆表)
```

### 1.5 数据库表

| 表名 | 说明 | 文件路径 |
|------|------|---------|
| tbl_user_tmp_episodic | 用户临时记忆表 | user_tmp_episodic_tbl.cpp |
| tbl_user_fact | 用户事实记忆表 | user_fact_tbl.cpp |
| tbl_core_memory | 核心记忆表 | core_memory_tbl.cpp |

---

## 场景2：用户记忆查询和检索

### 2.1 场景描述

用户发起查询请求，系统通过多种检索方式（向量搜索、文本搜索、主题检索等）召回相关记忆，并进行重排序和总结。

### 2.2 完整流程图

```
用户查询
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Controller层: RouteMgr::QueryUserMemory                    │
│  文件: service/src/controller/route_mgr.cpp:92-101         │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  Business层: UserMemoryService::OnReceiveUserMemoryQueryMsg │
│  文件: service/src/business/memory/user_memory_service.cpp  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 请求参数解析 (第205-211行)
                     │    - 调用ParseMemoryQueryRequest()
                     │    - 解析userId, query, filters, topK等
                     │
                     ▼
                     │
                     │ 2. 策略分发 (第214行)
                     │    - 调用DispatchParamByPolicy()
                     │    - 根据label分发查询策略
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  策略分发机制                                                │
│  文件: service/src/business/memory/user_memory_service.cpp  │
│  策略映射 (第61-67行):                                       │
│  {                                                           │
│    "policy_travel" → MemoryTravelManager::OnReceiveTravelQueryMsg,│
│    "policy_movie" → MemoryMovieManager::OnReceiveQueryMsg,  │
│    "policy_agent" → MemoryAgentManager::OnReceiveAgentQueryMsg,│
│    "policy_common" → MemoryCommonManager::OnReceiveCommonQueryMsg,│
│    "userInput" → MemoryCommonManager::OnReceiveCommonQueryMsg│
│  }                                                           │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  通用记忆查询: MemoryCommonManager::OnReceiveCommonQueryMsg │
│  文件: service/src/business/memory/memory_common_manager.cpp │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 查询改写 (第632-642行)
                     │    - 调用ModelInferExecutor::RewriteQuery()
                     │    - 使用LLM改写查询，提升检索效果
                     │
                     ▼
                     │
                     │ 2. 并发检索 (第187-377行)
                     │    - 调用QueryCommonMemoryInConcurrency()
                     │    - 并发执行多个检索任务
                     │
                     ▼
                     │
                     │ 3. 重排序 (第268-376行)
                     │    - 调用Rerank服务
                     │    - 对检索结果进行重排序
                     │
                     ▼
                     │
                     │ 4. 总结提炼 (第378-420行)
                     │    - 调用ModelInferExecutor::ExtractCommonContent()
                     │    - 使用LLM总结记忆内容
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  并发检索: QueryCommonMemoryInConcurrency                    │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 并发执行多个检索任务
                     │    ├─ GetMemoryByTopic (主题检索)
                     │    └─ GetMemoryByQuery (向量+文本检索)
                     │
                     ▼
                     │
                     │ 2. GetMemoryByQuery内部并发
                     │    ├─ GetMemoryByCoreVectorSearch
                     │    ├─ GetMemoryByCoreTextSearch
                     │    ├─ GetMemoryByEpisodicVectorSearch
                     │    ├─ GetMemoryByEpisodicTextSearch
                     │    ├─ GetMemoryBySemanticVectorSearch
                     │    └─ GetMemoryBySemanticTextSearch
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  向量检索示例: GetMemoryByCoreVectorSearch                  │
│  文件: service/src/business/memory/memory_common_manager.cpp │
│  第1070-1098行                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 关键代码路径

#### 2.3.1 Controller层

**文件**: `service/src/controller/route_mgr.cpp`

**方法**: `RouteMgr::QueryUserMemory()` (第92-101行)

```cpp
CMFrm::COM::ServerRespHandleMode RouteMgr::QueryUserMemory(
    const std::shared_ptr<CMFrm::ServiceRouter::Context> &context)
{
    LOG_INFO("QueryUserMemory thread enter");
    // 提交到查询线程池异步处理
    RouteMgr::GetInstance()->GetQueryThreadPool()->Submit([context]() {
        UserMemoryService::GetInstance()->OnReceiveUserMemoryQueryMessage(context);
    });
    return CMFrm::COM::ASYNC;
}
```

#### 2.3.2 Business层

**文件**: `service/src/business/memory/user_memory_service.cpp`

**方法**: `UserMemoryService::OnReceiveUserMemoryQueryMessage()` (第199-215行)

```cpp
void UserMemoryService::OnReceiveUserMemoryQueryMessage(
    const std::shared_ptr<CMFrm::ServiceRouter::Context> &context)
{
    // 解析请求参数
    UserMemoryReqParam reqParam;
    ParseMemoryQueryRequest(context->GetReqBody()->GetBody(), reqParam);
    
    // 策略分发
    DispatchParamByPolicy(reqParam, context);
}
```

#### 2.3.3 查询改写

**文件**: `service/src/business/memory/memory_common_manager.cpp`

**方法**: `ModelInferExecutor::RewriteQuery()` (第632-642行)

```cpp
bool rewriteRes = ModelInferExecutor::RewriteQuery(rewriteQueryInput, rewriteQuery);
PacketManager::EventMarking(EventLog("info", reqParam.userId, reqParam.taskId, 
    "CommonQuery.EndRewriteCommonQuery", rewriteRes ? "success" : "fail",
    "End rewrite user query, after rewrite, query is: " + rewriteQuery));
```

#### 2.3.4 并发检索

**文件**: `service/src/business/memory/memory_common_manager.cpp`

**方法**: `QueryCommonMemoryInConcurrency()` (第187-377行)

```cpp
void MemoryCommonManager::QueryCommonMemoryInConcurrency(
    const UserMemoryReqParam &reqParam, const std::string &rewriteQuery,
    std::map<std::string, std::shared_ptr<CommonMemory>> &memoryContent, 
    std::set<std::string> &memByDiffWay)
{
    ConcurrentExecutor executor;
    std::map<std::string, std::shared_ptr<CommonMemory>> memoryByTopics = {};
    std::map<std::string, std::shared_ptr<CommonMemory>> memoryByQuery = {};
    
    // 添加并发任务
    executor.AddTask(GetMemoryByTopic, reqParam, rewriteQuery, memoryByTopics, memByDiffWay1);
    executor.AddTask(GetMemoryByQuery, reqParam, rewriteQuery, memoryByQuery, memByDiffWay2);
    
    // 等待所有任务完成（超时3秒）
    bool success = executor.WaitForAll(std::chrono::milliseconds(3000));
    
    // Rerank处理
    std::vector<std::string> contentList;
    for (const auto& mem : memoryContentRerankSets) {
        contentList.push_back(mem);
    }
    std::vector<std::pair<int, double>> rerankRsp;
    ModelInfer::RerankModelThird(rewriteQuery, contentList, rerankSize, rerankRsp);
}
```

#### 2.3.5 向量检索

**文件**: `service/src/domain/datatable/core_memory_tbl.cpp`

**方法**: `CoreMemoryTbl::QueryCoreMemoryByUserTopicVector()` (第57-120行)

```cpp
std::vector<std::shared_ptr<CoreMemory>> CoreMemoryTbl::QueryCoreMemoryByUserTopicVector(
    std::shared_ptr<CoreMemory> coreMemory)
{
    // 获取检索器
    auto retriever = RagMgr::GetInstance()->GetRetriever();
    
    // 向量化查询
    auto embeddingData = Embedding(coreMemory->GetQuery());
    
    // 构建查询选项
    std::shared_ptr<QueryOption> queryOption = QueryOption::Builder(TableQueryType::VECTOR)
        .WithVectorField("details_emb")
        .WithVectorDistanceAlgorithm(VectorDistanceAlgorithm::L2)
        .WithVectorIndex("tbl_core_memory_details_emb_index")
        .WithVectorTopK(5)
        .WithVectorValue(*embeddingData.get())
        .WithBaseFilters(filterBuilder.Build())
        .Build();
    
    // 执行向量检索
    retriever->VectorSearch("tbl_core_memory", queryOption, queryResult);
    
    return coreMemoryItems;
}
```

#### 2.3.6 重排序

**文件**: `service/src/business/memory/memory_common_manager.cpp`

**方法**: (第268-376行)

```cpp
std::vector<std::string> mergedResults2Rerank = 
    std::vector<std::string>(memoryContentRerankSets.begin(), memoryContentRerankSets.end());

std::shared_ptr<DM::RAG::ReRankOption> option = 
    DM::RAG::ReRankOption::Builder()
        .WithTimeOut(ConfigMgr::GetInstance()->GetCommonMemoryRerankTimeout())
        .Build();
auto rerank = RagMgr::GetInstance()->GetRerank();
std::vector<DM::RAG::ReRankResult> rerankResults;
DM::RAG::ReRankOperateCode retCode = rerank->ReRank(
    rewriteQuery, mergedResults2Rerank, option, rerankResults);

// 按得分排序
std::sort(rerankResults.begin(), rerankResults.end(),
    [](const DM::RAG::ReRankResult& a, const DM::RAG::ReRankResult& b) {
        return a.relevanceScore > b.relevanceScore;
    });

// 过滤和选择
for (; i < rerankResults.size(); i++) {
    if (rerankResults[i].relevanceScore >= paasScore && i < maxMemories) {
        memoryContent.insert({rerankResults[i].document, allContentMap[rerankResults[i].document]});
    } else {
        break;
    }
}
```

### 2.4 数据流转

```
用户查询
    │
    ▼
查询改写 (LLM)
    │
    ▼
并发检索
    │
    ├─ 向量检索 (CoreMemory, EpisodicMemory, SemanticMemory)
    ├─ 文本检索 (CoreMemory, EpisodicMemory, SemanticMemory)
    └─ 主题检索
    │
    ▼
结果合并
    │
    ▼
重排序 (Rerank)
    │
    ▼
总结提炼 (LLM)
    │
    ▼
返回结果
```

---

## 场景3：对话历史管理

### 3.1 场景描述

管理用户与AI的对话历史，包括对话存储、批量添加、查询等功能。

### 3.2 完整流程图

```
用户对话
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Controller层: RouteMgr::AddUserConversation                │
│  文件: service/src/controller/route_mgr.cpp:103-110        │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  Business层: UserConversationService::OnReceiveUserConvAddMsg│
│  文件: service/src/business/conversation/user_conversation_service.cpp│
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 解析对话参数
                     │    - userId, sessionId, role, content
                     │
                     ▼
                     │
                     │ 2. 插入对话表
                     │    - UserConversationTbl::Insert()
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  定时任务: MemoryCommonManager::ExtractConvMemory           │
│  文件: service/src/business/memory/memory_common_manager.cpp │
│  第1286-1299行                                              │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 查询待提取对话
                     │    - UserTmpConversationTbl::QueryConversationsOrderedByDate()
                     │
                     ▼
                     │
                     │ 2. 调用Python记忆提取服务
                     │    - SendConversationExtractReq()
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  Python服务调用: SendConversationExtractReq                  │
│  文件: service/src/business/memory/memory_common_manager.cpp │
│  第1238-1284行                                              │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 获取Python服务URL
                     │    - PyMemServiceMgr::GetInstance()->RequestSrvUrl()
                     │
                     ▼
                     │
                     │ 2. 发送HTTP POST请求
                     │    - URL: /mirix/v1/user/memories/add
                     │    - 请求体: conversations数组
                     │
                     ▼
                     │
                     │ 3. 释放服务URL
                     │    - PyMemServiceMgr::GetInstance()->ReleaseSrvUrl()
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  Python Agent处理                                            │
│  文件: MemoryService/server/route.py:29-69                  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 接收请求
                     │    - add_memory()函数
                     │
                     ▼
                     │
                     │ 2. 获取Agent列表
                     │    - agent_manager.get_add_memory_agents()
                     │
                     ▼
                     │
                     │ 3. 并发执行Agent
                     │    ├─ CoreMemoryAgent (画像记忆)
                     │    ├─ EpisodicMemoryAgent (情景记忆)
                     │    └─ SemanticMemoryAgent (语义记忆)
                     │
                     ▼
                     │
                     │ 4. 返回结果
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  Agent基类: BaseMemoryAgent::handle()                        │
│  文件: MemoryService/agent/base_memory_agent.py:70-107      │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 关键代码路径

#### 3.3.1 对话添加

**文件**: `service/src/controller/route_mgr.cpp`

**方法**: `RouteMgr::AddUserConversation()` (第103-110行)

```cpp
CMFrm::COM::ServerRespHandleMode RouteMgr::AddUserConversation(
    const std::shared_ptr<CMFrm::ServiceRouter::Context> &context)
{
    LOG_INFO("AddUserConversation enter");
    UserConversationService::GetInstance()->OnReceiveUserConversationAddMessage(context);
    return CMFrm::COM::SYNC;
}
```

#### 3.3.2 对话提取

**文件**: `service/src/business/memory/memory_common_manager.cpp`

**方法**: `MemoryCommonManager::ExtractConvMemory()` (第1286-1299行)

```cpp
void MemoryCommonManager::ExtractConvMemory(const std::string &userId, 
    std::shared_ptr<ExtractStatusMgr> statusMgr, int expireTimeSec, int limit, int maxLength)
{
    // 查询待提取对话
    auto userConversationInfos = UserTmpConversationTbl::QueryConversationsOrderedByDate(
        userId, "", expireTimeSec, limit);
    
    for (const auto& convInfo : userConversationInfos) {
        // 调用Python服务提取记忆
        auto rsp = SendConversationExtractReq(userId, convInfos, flag, timeOut, source);
    }
}
```

#### 3.3.3 Python服务调用

**文件**: `service/src/business/memory/memory_common_manager.cpp`

**方法**: `MemoryCommonManager::SendConversationExtractReq()` (第1238-1284行)

```cpp
MemoryCommonManager::ConvExtractRsp MemoryCommonManager::SendConversationExtractReq(
    const std::string &userId, std::vector<UserConversationInfo> &convInfos, 
    int flag, int timeOut, const std::string &source)
{
    // 获取Python服务URL
    std::string url = PyMemServiceMgr::GetInstance()->RequestSrvUrl();
    
    // 构建请求体
    rapidjson::Document document;
    document.SetObject();
    rapidjson::Value convArray(rapidjson::kArrayType);
    for (const auto& conv : convInfos) {
        rapidjson::Value convObj(rapidjson::kObjectType);
        convObj.AddMember("role", rapidjson::StringRef(conv.role.c_str()), document.GetAllocator());
        convObj.AddMember("content", rapidjson::StringRef(conv.content.c_str()), document.GetAllocator());
        convArray.PushBack(convObj, document.GetAllocator());
    }
    document.AddMember("conversations", convArray, document.GetAllocator());
    document.AddMember("source", rapidjson::StringRef(source.c_str()), document.GetAllocator());
    
    // 发送HTTP POST请求
    rapidjson::StringBuffer buffer;
    rapidjson::Writer<rapidjson::StringBuffer> writer(buffer);
    document.Accept(writer);
    
    auto [rspbody, execCode] = HttpHelper::InnerCurlService(
        url + "/mirix/v1/user/memories/add", buffer.GetString(), userId);
    
    // 释放服务URL
    PyMemServiceMgr::GetInstance()->ReleaseSrvUrl(url);
    
    return rspResult;
}
```

#### 3.3.4 Python Agent处理

**文件**: `MemoryService/server/route.py`

**方法**: `add_memory()` (第29-69行)

```python
def add_memory(agent_manager: AgentManager, user_id: str, body: Dict):
    logger.set_user_id(user_id=user_id)
    content = body.get("conversations")
    source = body.get("source", "")
    
    event = Event(user_id=user_id, content=content, source=source)
    agents = agent_manager.get_add_memory_agents()
    
    # 并发执行多个Agent
    with concurrent.futures.ThreadPoolExecutor(max_workers=len(agents)) as executor:
        futures = {agent_name: executor.submit(agent.handle, event) 
                   for agent_name, agent in agents.items()}
        
        for agent_name, future in futures.items():
            try:
                future.result(timeout=10000)
            except Exception as e:
                exceptions.append((agent_name, e))
```

#### 3.3.5 Agent基类

**文件**: `MemoryService/agent/base_memory_agent.py`

**方法**: `BaseMemoryAgent::handle()` (第70-107行)

```python
def handle(self, event: Event):
    try:
        logger.info(f"start to handle add memory")
        original_data = event.content
        user_id = event.user_id
        
        # 提取独立内容
        details = self.extract_independent_content(original_data)
        
        # 提取主题和摘要
        extracted_memory_data_list = self._extract_topic_tree_path_summary(
            details, user_id, self.extract_topic_and_summary_prompt)
        
        # 处理每个memory_data
        for memory_data in extracted_memory_data_list:
            memory_data_list_with_action = self._decide_action(memory_data)
            for action_data in memory_data_list_with_action:
                self._perform_action(action_data)
```

### 3.4 数据流转

```
用户对话
    │
    ▼
UserConversationTbl (对话表)
    │
    ▼
UserTmpConversationTbl (临时对话表)
    │
    ▼
Python服务 (Agent处理)
    │
    ├─ CoreMemoryAgent → CoreMemoryTbl
    ├─ EpisodicMemoryAgent → EpisodicMemoryTbl
    └─ SemanticMemoryAgent → SemanticMemoryTbl
```

---

## 场景4：情景记忆提取

### 4.1 场景描述

从用户对话中自动提取情景记忆，包括事件的时间、地点、人物等要素。

### 4.2 完整流程图

```
对话数据
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  EpisodicMemoryAgent::handle()                               │
│  文件: MemoryService/agent/episodic_memory_agent.py         │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 提取独立内容
                     │    - extract_independent_content (LLM)
                     │
                     ▼
                     │
                     │ 2. 提取主题和树路径
                     │    - _extract_topic_tree_path_summary (LLM)
                     │
                     ▼
                     │
                     │ 3. 判断操作类型
                     │    - _decide_action (insert/update/keep)
                     │
                     ▼
                     │
                     │ 4. 执行操作
                     │    - _perform_action
                     │    ├─ insert → create_memory → EpisodicMemoryTbl::Insert
                     │    └─ update → update_memory → EpisodicMemoryTbl::Update
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  EpisodicMemoryTbl::Insert()                                │
│  文件: service/src/domain/datatable/episodic_memory_tbl.cpp  │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 关键代码路径

#### 4.3.1 情景记忆Agent

**文件**: `MemoryService/agent/episodic_memory_agent.py`

```python
class EpisodicMemoryAgent(BaseMemoryAgent):
    @property
    def extract_independent_content_prompt(self) -> str:
        return EXTRACT_INDEPENDENT_CONTENT
    
    @property
    def insert_or_merge_prompt(self) -> str:
        return INSERT_OR_MERGE_EPISODIC_MEMORY
    
    @property
    def extract_topic_and_summary_prompt(self) -> str:
        return EXTRACT_EPISODIC_MEMORY_TOPIC_AND_SUMMARY
    
    def create_memory(self, memory_data: MemoryData):
        memory_data.source = self.get_source()
        return self.gaussdb_client.episodic_memory.create_memory(memory_data)
```

### 4.4 数据流转

```
对话数据
    │
    ▼
EpisodicMemoryAgent处理
    │
    ├─ LLM提取独立内容
    ├─ LLM提取主题和树路径
    ├─ 判断操作类型
    └─ 执行操作
    │
    ▼
EpisodicMemoryTbl (情景记忆表)
```

---

## 场景5：用户画像构建

### 5.1 场景描述

从用户对话中提取用户画像信息，包括用户属性、偏好、性格等，并定期进行总结和优化。

### 5.2 完整流程图

```
用户信息
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  CoreMemoryAgent::handle()                                  │
│  文件: MemoryService/agent/core_memory_agent.py            │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 提取独立内容
                     │    - extract_independent_content (LLM)
                     │
                     ▼
                     │
                     │ 2. 提取主题和摘要
                     │    - _extract_topic_tree_path_summary (LLM)
                     │
                     ▼
                     │
                     │ 3. 判断操作类型
                     │    - _decide_action
                     │
                     ▼
                     │
                     │ 4. 执行操作
                     │    - _perform_action
                     │    ├─ insert → create_memory → CoreMemoryTbl::Insert
                     │    └─ update → update_memory → CoreMemoryTbl::Update
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  定时总结: ScheduledCoreSummary::Executor                   │
│  文件: service/include/business/schedule/scheduled_core_summary_memory.h│
│  周期: 1分钟                                               │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 查询需要总结的画像
                     │
                     ▼
                     │
                     │ 2. 调用LLM总结
                     │
                     ▼
                     │
                     │ 3. 更新画像
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  CoreMemoryTbl::Insert()                                    │
│  文件: service/src/domain/datatable/core_memory_tbl.cpp     │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 关键代码路径

#### 5.3.1 画像Agent

**文件**: `MemoryService/agent/core_memory_agent.py`

```python
class CoreMemoryAgent(BaseMemoryAgent):
    @property
    def extract_independent_content_prompt(self) -> str:
        return EXTRACT_INDEPENDENT_CONTENT
    
    @property
    def insert_or_merge_prompt(self) -> str:
        return INSERT_OR_MERGE_CORE_MEMORY
    
    @property
    def extract_topic_and_summary_prompt(self) -> str:
        return EXTRACT_CORE_MEMORY_TOPIC_AND_SUMMARY
    
    def create_memory(self, memory_data: MemoryData):
        return self.gaussdb_client.core_memory.create_memory(memory_data)
```

#### 5.3.2 画像总结定时任务

**文件**: `service/include/business/schedule/scheduled_core_summary_memory.h`

```cpp
const int SCHEDULED_CORE_SUMMARY_MEMORY_CYCLE = 1; // 1分钟周期

class ScheduledCoreSummary {
public:
    static std::shared_ptr<ScheduledCoreSummary> GetInstance();
    void Init();
private:
    ScheduledCoreSummary();
    void Executor();
};
```

### 5.4 数据流转

```
用户信息
    │
    ▼
CoreMemoryAgent处理
    │
    ├─ LLM提取独立内容
    ├─ LLM提取主题和摘要
    ├─ 判断操作类型
    └─ 执行操作
    │
    ▼
CoreMemoryTbl (核心记忆表)
    │
    ▼
定时总结
    │
    ▼
更新CoreMemoryTbl
```

---

## 场景6：问答短记忆管理

### 6.1 场景描述

管理用户的问答对形式的短时记忆，支持批量添加、TopK查询、关系生成和老化清理。

### 6.2 完整流程图

```
QA对添加
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Controller层: RouteMgr::HandleQABatchAdd                    │
│  文件: service/src/controller/route_mgr.cpp:183-188        │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  Business层: QAShortMemoryService::OnReceiveQABatchAddMsg   │
│  文件: service/src/business/qa/qa_short_memory_service.cpp  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 解析QA对
                     │
                     ▼
                     │
                     │ 2. 插入QA表
                     │    - QAShortMemoryTbl::Insert()
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  定时任务: QAShortMemoryMgr::TimedExecute                   │
│  文件: service/include/business/qa/qa_short_memory_mgr.h   │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 查询过期记录
                     │
                     ▼
                     │
                     │ 2. 根据老化策略清理
                     │    ├─ maxRecords: 最大记录数限制
                     │    └─ agingTime: 老化时间(分钟)
                     │
                     ▼
                     │
                     │ 3. 删除过期记录
                     │
                     ▼
                     │
                     │ 4. 更新关系索引
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  关系生成: GenerateRelationAsync                            │
│  文件: service/src/business/qa/qa_short_memory_mgr.cpp      │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 关键代码路径

#### 6.3.1 QA添加入口

**文件**: `service/src/controller/route_mgr.cpp`

**方法**: `RouteMgr::HandleQABatchAdd()` (第183-188行)

```cpp
CMFrm::COM::ServerRespHandleMode RouteMgr::HandleQABatchAdd(
    const std::shared_ptr<CMFrm::ServiceRouter::Context> &context)
{
    LOG_INFO("%s enter", __FUNCTION__);
    QAShortMemoryService::GetInstance()->OnReceiveQABatchAddMsg(context);
    return CMFrm::COM::ASYNC;
}
```

#### 6.3.2 老化策略定义

**文件**: `service/include/common/common_define.h`

```cpp
struct QAMemoryAgingPolicy {
    int maxRecords;
    int agingTime;  // min
    QAMemoryAgingPolicy() : maxRecords(20), agingTime(30) {}
};
```

### 6.4 数据流转

```
QA对
    │
    ▼
QAShortMemoryTbl (问答短记忆表)
    │
    ▼
定时老化清理
    │
    ├─ 检查maxRecords
    ├─ 检查agingTime
    └─ 删除过期记录
    │
    ▼
关系索引更新
```

---

## 场景7：智能检索（向量搜索+重排序）

### 7.1 场景描述

通过向量搜索、文本搜索、重排序等多种方式，智能检索相关记忆。

### 7.2 完整流程图

```
用户查询
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  QueryCommonMemoryInConcurrency                             │
│  文件: service/src/business/memory/memory_common_manager.cpp │
│  第1187-1223行                                              │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 并发执行多个检索任务
                     │    ├─ GetMemoryByCoreVectorSearch
                     │    ├─ GetMemoryByCoreTextSearch
                     │    ├─ GetMemoryByEpisodicVectorSearch
                     │    ├─ GetMemoryByEpisodicTextSearch
                     │    ├─ GetMemoryBySemanticVectorSearch
                     │    └─ GetMemoryBySemanticTextSearch
                     │
                     ▼
                     │
                     │ 2. 合并检索结果
                     │
                     ▼
                     │
                     │ 3. 重排序 Rerank
                     │    ├─ 内部Rerank (GaussDB)
                     │    └─ 外部Rerank (第三方模型)
                     │
                     ▼
                     │
                     │ 4. 过滤优化
                     │    ├─ 拒绝分数过滤 (rejectScore)
                     │    ├─ 通过分数过滤 (paasScore)
                     │    └─ TopK选择
                     │
                     ▼
                     │
                     │ 5. 返回最终结果
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  向量检索示例: GetMemoryByCoreVectorSearch                  │
│  文件: service/src/business/memory/memory_common_manager.cpp │
│  第1070-1098行                                              │
└─────────────────────────────────────────────────────────────┘
```

### 7.3 关键代码路径

#### 7.3.1 统一检索接口

**文件**: `service/src/business/memory/memory_common_manager.cpp`

**方法**: `GetMemoryByQuery()` (第1187-1223行)

```cpp
void MemoryCommonManager::GetMemoryByQuery(
    const UserMemoryReqParam &param, const std::string &query,
    std::map<std::string, std::shared_ptr<CommonMemory>> &memoryByQuery,
    std::set<std::string> &memoryWithQuery)
{
    int taskNum = 6;
    std::vector<std::map<std::string, std::shared_ptr<CommonMemory>>> memoryByQueryInTbl(taskNum);
    std::vector<std::string> memoryByQueryInTblWithWay(taskNum, "");
    ConcurrentExecutor fusionSearchExecutor;
    
    // 添加并发任务
    if (param.useCoreMemory) {
        fusionSearchExecutor.AddTask(GetMemoryByCoreVectorSearch, param, query, 
            memoryByQueryInTbl[0], memoryByQueryInTblWithWay[0]);
        fusionSearchExecutor.AddTask(GetMemoryByCoreTextSearch, param, query, 
            memoryByQueryInTbl[1], memoryByQueryInTblWithWay[1]);
    }
    if (param.useEpisodicMemory) {
        fusionSearchExecutor.AddTask(GetMemoryByEpisodicVectorSearch, param, query, 
            memoryByQueryInTbl[2], memoryByQueryInTblWithWay[2]);
        fusionSearchExecutor.AddTask(GetMemoryByEpisodicTextSearch, param, query, 
            memoryByQueryInTbl[3], memoryByQueryInTblWithWay[3]);
    }
    if (param.useSemanticMemory) {
        fusionSearchExecutor.AddTask(GetMemoryBySemanticVectorSearch, param, query, 
            memoryByQueryInTbl[4], memoryByQueryInTblWithWay[4]);
        fusionSearchExecutor.AddTask(GetMemoryBySemanticTextSearch, param, query, 
            memoryByQueryInTbl[5], memoryByQueryInTblWithWay[5]);
    }
    
    // 等待所有任务完成
    bool success = fusionSearchExecutor.WaitForAll(std::chrono::milliseconds(3000));
}
```

#### 7.3.2 向量检索示例

**文件**: `service/src/business/memory/memory_common_manager.cpp`

**方法**: `GetMemoryByCoreVectorSearch()` (第1070-1098行)

```cpp
void MemoryCommonManager::GetMemoryByCoreVectorSearch(
    const UserMemoryReqParam &param, const std::string &query,
    std::map<std::string, std::shared_ptr<CommonMemory>> &memoryByCoreFusionSearch,
    std::string &memoryByCoreFusionSearchWithWay)
{
    std::string mem = "[MemoryByCoreQueryVector] query: " + query + "\n";
    auto coreMemory = std::make_shared<CoreMemory>(param.userId);
    coreMemory->SetQuery(query);
    
    // 查询向量检索
    auto coreMemoryItems = CoreMemoryTbl::QueryCoreMemoryByQueryVector(coreMemory);
    
    int i = 1;
    for (auto &item : coreMemoryItems) {
        mem += std::to_string(i) + ". " + item->GetDetail() + 
              ", score: " + std::to_string(item->GetDetailDistance()) + "\n";
        memoryByCoreFusionSearch.insert({item->GetDetail(), item});
        i++;
    }
}
```

#### 7.3.3 重排序

**文件**: `service/src/business/memory/memory_common_manager.cpp`

**方法**: (第268-376行)

```cpp
std::vector<std::string> mergedResults2Rerank = 
    std::vector<std::string>(memoryContentRerankSets.begin(), memoryContentRerankSets.end());

std::shared_ptr<DM::RAG::ReRankOption> option = 
    DM::RAG::ReRankOption::Builder()
        .WithTimeOut(ConfigMgr::GetInstance()->GetCommonMemoryRerankTimeout())
        .Build();
auto rerank = RagMgr::GetInstance()->GetRerank();
std::vector<DM::RAG::ReRankResult> rerankResults;
DM::RAG::ReRankOperateCode retCode = rerank->ReRank(
    rewriteQuery, mergedResults2Rerank, option, rerankResults);

// 按得分排序
std::sort(rerankResults.begin(), rerankResults.end(),
    [](const DM::RAG::ReRankResult& a, const DM::RAG::ReRankResult& b) {
        return a.relevanceScore > b.relevanceScore;
    });

// 过滤和选择
for (; i < rerankResults.size(); i++) {
    if (rerankResults[i].relevanceScore >= paasScore && i < maxMemories) {
        memoryContent.insert({rerankResults[i].document, allContentMap[rerankResults[i].document]});
    } else {
        break;
    }
}
```

### 7.4 数据流转

```
用户查询
    │
    ▼
并发检索
    │
    ├─ CoreMemory向量检索
    ├─ CoreMemory文本检索
    ├─ EpisodicMemory向量检索
    ├─ EpisodicMemory文本检索
    ├─ SemanticMemory向量检索
    └─ SemanticMemory文本检索
    │
    ▼
结果合并
    │
    ▼
重排序
    │
    ├─ 内部Rerank
    └─ 外部Rerank
    │
    ▼
过滤优化
    │
    ├─ 拒绝分数过滤
    ├─ 通过分数过滤
    └─ TopK选择
    │
    ▼
返回结果
```

---

## 场景8：记忆老化清理

### 8.1 场景描述

根据老化策略（最大记录数、老化时间）定期清理过期的记忆记录。

### 8.2 完整流程图

```
定时任务触发
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  QAShortMemoryMgr::TimedExecute                              │
│  文件: service/include/business/qa/qa_short_memory_mgr.h   │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 查询过期记录
                     │
                     ▼
                     │
                     │ 2. 根据老化策略清理
                     │    ├─ maxRecords: 最大记录数限制
                     │    └─ agingTime: 老化时间(分钟)
                     │
                     ▼
                     │
                     │ 3. 删除过期记录
                     │
                     ▼
                     │
                     │ 4. 更新关系索引
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  老化策略                                                    │
│  文件: service/include/common/common_define.h:162-166      │
└─────────────────────────────────────────────────────────────┘
```

### 8.3 关键代码路径

#### 8.3.1 老化策略定义

**文件**: `service/include/common/common_define.h`

```cpp
struct QAMemoryAgingPolicy {
    int maxRecords;
    int agingTime;  // min
    QAMemoryAgingPolicy() : maxRecords(20), agingTime(30) {}
};
```

### 8.4 数据流转

```
定时任务
    │
    ▼
查询过期记录
    │
    ▼
应用老化策略
    │
    ├─ 检查maxRecords
    ├─ 检查agingTime
    └─ 标记过期记录
    │
    ▼
删除过期记录
    │
    ▼
更新关系索引
```

---

## 场景9：记忆总结优化

### 9.1 场景描述

定期对记忆进行总结和优化，包括子主题总结、画像分类等。

### 9.2 完整流程图

```
定时任务触发
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  ScheduledMemorySummary::Executor                           │
│  文件: service/src/business/schedule/scheduled_memory_summary.cpp│
│  周期: 30分钟（可配置）                                     │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 查询需要更新的子主题
                     │
                     ▼
                     │
                     │ 2. 调用对应策略进行总结
                     │    ├─ policy_agent → MemoryAgentManager::UpdateSubtopicSummary
                     │    └─ policy_common → MemoryCommonManager::UpdateSubtopicSummary
                     │
                     ▼
                     │
                     │ 3. 更新子主题总结
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  策略分发机制                                                │
│  文件: service/src/business/schedule/scheduled_memory_summary.cpp│
│  第17-19行                                                  │
└─────────────────────────────────────────────────────────────┘
```

### 9.3 关键代码路径

#### 9.3.1 记忆总结定时任务

**文件**: `service/src/business/schedule/scheduled_memory_summary.cpp`

**方法**: `ScheduledMemorySummary::Init()` (第79-88行)

```cpp
void ScheduledMemorySummary::Init()
{
    InitMemSummaryParams();
    std::shared_ptr<CMFrm::Timer::TimerTask> m_timer = std::make_shared<TimerTask>(
        "MemorySummaryTask", 5000, m_summaryCycle * 60 * 1000, 
        [this]() { this->Executor(); }, 1);
    TimerManager::GetTimerManager()->AddTimerTask(m_timer);
    
    m_threadPool = std::make_shared<CMFrm::ThreadPool::ThreadPool>(4, "summary_extract");
}
```

#### 9.3.2 策略分发

**文件**: `service/src/business/schedule/scheduled_memory_summary.cpp`

**策略映射** (第17-19行):

```cpp
const std::unordered_map<std::string, std::function<void(std::shared_ptr<UserSubtopicSummary> &)>>>
ScheduledMemorySummary::policyMap = {
    {"policy_agent", MemoryAgentManager::UpdateSubtopicSummary},
    {"policy_common", MemoryCommonManager::UpdateSubtopicSummary}
};
```

### 9.4 数据流转

```
定时任务
    │
    ▼
查询需要更新的子主题
    │
    ▼
策略分发
    │
    ├─ policy_agent
    └─ policy_common
    │
    ▼
调用对应Manager
    │
    ├─ MemoryAgentManager::UpdateSubtopicSummary
    └─ MemoryCommonManager::UpdateSubtopicSummary
    │
    ▼
更新子主题总结
```

---

## 场景10：用户输入处理

### 10.1 场景描述

处理用户的输入事件，触发记忆提取和存储流程。

### 10.2 完整流程图

```
用户输入
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Controller层: RouteMgr::UserInput                          │
│  文件: service/src/controller/route_mgr.cpp:131-138        │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  Business层: MemoryUserInputManager::OnReceiveUserEventMsg  │
│  文件: service/src/business/memory/memory_user_input_manager.cpp│
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ 1. 解析用户输入
                     │    - userId, sessionId, input, eventType
                     │
                     ▼
                     │
                     │ 2. 创建UserEvent对象
                     │
                     ▼
                     │
                     │ 3. 插入事件表
                     │    - UserEventTbl::Insert()
                     │
                     ▼
                     │
                     │ 4. 触发记忆提取
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  记忆提取流程                                                │
│  文件: service/src/business/memory/memory_user_input_manager.cpp│
└─────────────────────────────────────────────────────────────┘
```

### 10.3 关键代码路径

#### 10.3.1 用户输入入口

**文件**: `service/src/controller/route_mgr.cpp`

**方法**: `RouteMgr::UserInput()` (第131-138行)

```cpp
CMFrm::COM::ServerRespHandleMode RouteMgr::UserInput(
    const std::shared_ptr<CMFrm::ServiceRouter::Context> &context)
{
    LOG_INFO("UserInput enter");
    MemoryUserInputManager::GetInstance()->OnReceiveUserEventMsg(context);
    return CMFrm::COM::ASYNC;
}
```

### 10.4 数据流转

```
用户输入
    │
    ▼
UserEventTbl (事件表)
    │
    ▼
触发记忆提取
    │
    ├─ 情景记忆提取
    ├─ 用户画像更新
    └─ 语义记忆提取
```

---

## 定时任务调度

### 定时任务列表

| 任务类 | 执行周期 | 线程池 | 功能描述 |
|-------|---------|--------|---------|
| UserMemoryManager | 5秒 | SceneExtractThreadPool (8线程) | 用户记忆提取 |
| ScheduledMemorySummary | 30分钟 | SummaryExtractThreadPool (4线程) | 记忆总结优化 |
| ScheduledCoreSummary | 1分钟 | - | 画像总结优化 |
| ScheduledClassifyPersona | - | - | 画像分类 |
| QAShortMemoryMgr | - | - | QA短记忆老化清理 |

### 定时任务初始化

#### UserMemoryManager

**文件**: `service/src/business/memory/user_memory_manager.cpp`

```cpp
void UserMemoryManager::Init()
{
    auto m_timer = std::make_shared<TimerTask>(
        "UserMemoryTask", 5000, 5000, [this]() { this->Excutor(); }, 1);
    TimerManager::GetTimerManager()->AddTimerTask(m_timer);
    
    m_threadPool = std::make_shared<CMFrm::ThreadPool::ThreadPool>(8, "scene_extract");
}
```

#### ScheduledMemorySummary

**文件**: `service/src/business/schedule/scheduled_memory_summary.cpp`

```cpp
void ScheduledMemorySummary::Init()
{
    InitMemSummaryParams();
    std::shared_ptr<CMFrm::Timer::TimerTask> m_timer = std::make_shared<TimerTask>(
        "MemorySummaryTask", 5000, m_summaryCycle * 60 * 1000, 
        [this]() { this->Executor(); }, 1);
    TimerManager::GetTimerManager()->AddTimerTask(m_timer);
    
    m_threadPool = std::make_shared<CMFrm::ThreadPool::ThreadPool>(4, "summary_extract");
}
```

---

## 策略模式应用

### 策略映射列表

#### 1. 记忆提取策略

**文件**: `service/src/business/memory/user_memory_manager.cpp`

**策略映射** (第16-20行):

```cpp
const std::unordered_map<std::string, std::function<void(std::shared_ptr<UserMemory> &)>> 
UserMemoryManager::policyMap = {
    {"policy_movie", MemoryMovieManager::ExtractFact},
    {"policy_travel", MemoryTravelManager::ExtractFact},
    {"policy_agent", MemoryAgentManager::ExtractFact},
    {"policy_common", MemoryCommonManager::ExtractFact}
};
```

#### 2. 查询路由策略

**文件**: `service/src/business/memory/user_memory_service.cpp`

**策略映射** (第61-67行):

```cpp
const std::unordered_map<std::string, UserMemoryService::PolicyFunction> 
UserMemoryService::policyMap = {
    {"policy_travel", &MemoryTravelManager::OnReceiveTravelQueryMessage},
    {"policy_movie", &MemoryMovieManager::OnReceiveQueryMessage},
    {"policy_agent", &MemoryAgentManager::OnReceiveAgentQueryMessage},
    {"policy_common", &MemoryCommonManager::OnReceiveCommonQueryMessage},
    {"userInput", &MemoryCommonManager::OnReceiveCommonQueryMessage}
};
```

#### 3. 记忆总结策略

**文件**: `service/src/business/schedule/scheduled_memory_summary.cpp`

**策略映射** (第17-19行):

```cpp
const std::unordered_map<std::string, std::function<void(std::shared_ptr<UserSubtopicSummary> &)>>>
ScheduledMemorySummary::policyMap = {
    {"policy_agent", MemoryAgentManager::UpdateSubtopicSummary},
    {"policy_common", MemoryCommonManager::UpdateSubtopicSummary}
};
```

#### 4. 短期记忆管理策略

**文件**: `service/src/business/memory/user_memory_service.cpp`

**策略映射** (第69-73行):

```cpp
const std::unordered_map<std::string, UserMemoryService::ShortMemoryManagerFunction> 
UserMemoryService::shortMemoryMap = {
    {"add", &ShortMemoryManager::AddShortMemory},
    {"delete", &ShortMemoryManager::DeleteShortMemory},
    {"refresh", &ShortMemoryManager::RefreshShortMemory},
    {"get", &ShortMemoryManager::GetShortMemory}
};
```

---

## 并发和性能优化

### 线程池配置

| 线程池名称 | 线程数 | 用途 | 文件位置 |
|-----------|-------|------|---------|
| QueryThreadPool | 32 | 查询请求处理 | route_mgr.cpp:70 |
| NormalThreadPool | 4 | 普通请求处理 | route_mgr.cpp:71 |
| SceneExtractThreadPool | 8 | 场景提取 | user_memory_manager.cpp:41 |
| SummaryExtractThreadPool | 4 | 记忆总结 | scheduled_memory_summary.cpp:87 |

### 并发执行器

**类名**: `ConcurrentExecutor`

**用途**: 并发执行多个任务，提高检索效率

**使用示例**:

**文件**: `service/src/business/memory/memory_common_manager.cpp` (第193-202行)

```cpp
ConcurrentExecutor executor;
std::map<std::string, std::shared_ptr<CommonMemory>> memoryByTopics = {};
std::map<std::string, std::shared_ptr<CommonMemory>> memoryByQuery = {};
std::set<std::string> memByDiffWay1 = {};
std::set<std::string> memByDiffWay2 = {};

executor.AddTask(GetMemoryByTopic, reqParam, rewriteQuery, memoryByTopics, memByDiffWay1);
executor.AddTask(GetMemoryByQuery, reqParam, rewriteQuery, memoryByQuery, memByDiffWay2);

bool success = executor.WaitForAll(std::chrono::milliseconds(3000));
```

### Python服务并发

**并发方式**: 线程池

**线程数**: 与Agent数量一致

**使用示例**:

**文件**: `MemoryService/server/route.py` (第47-69行)

```python
with concurrent.futures.ThreadPoolExecutor(max_workers=len(agents)) as executor:
    futures = {agent_name: executor.submit(agent.handle, event) 
               for agent_name, agent in agents.items()}
    
    for agent_name, future in futures.items():
        try:
            future.result(timeout=10000)
        except Exception as e:
            exceptions.append((agent_name, e))
```

---

## 外部服务调用

### LLM服务调用

**调用位置**: `service/src/infrastructure/model/model_infer.cpp`

**方法**: `ModelInfer::LLMModelInferSync` (第47-78行)

**使用场景**:
- 查询改写: `ModelInferExecutor::RewriteQuery`
- 记忆总结: `ModelInferExecutor::ExtractCommonContent`
- 主题提取: `ModelInferExecutor::ExtractQueryTopicsForCoreMemory`
- 树路径提取: `ModelInferExecutor::ExtractQueryTreePathForEpisodicMemory`

```cpp
bool ModelInfer::LLMModelInferSync(const std::string &query, 
    const std::string &promptName, std::string &outRsp,
    std::string model, std::string modelService, int timeoutSeconds)
{
    std::string modelName = ConfigMgr::GetInstance()->GetEnv(model);
    std::string modelServiceName = ConfigMgr::GetInstance()->GetEnv(modelService);
    
    MMUserIntention intention = {.query = query + " /no_think", 
                                  .modaltype = MMLLM_MODAL_TYPE_TEXT};
    MMReasonParam reasonParam = {
        .modelServiceName = modelServiceName, 
        .modelName = modelName, 
        .temperature = 0.001, 
        .maxToken = 10246};
    
    auto prompt = mmsdk::MMPromptTemplate(mmsdk::MMPromptTmpltKey{
        .name = promptName,
        .owner = SERVICE_NAME,
    });
    prompt.SetContent(ModelMgr::GetInstance()->GetPrompt(promptName));
    prompt.RenderPrompt({});
    
    outRsp = LLMModelInfer(reasonParam, intention, prompt, timeoutSeconds);
    return !outRsp.empty();
}
```

### Embedding服务调用

**调用位置**: `service/src/domain/datatable/data_management_base.cpp`

**方法**: `DataManagementBase::Embedding`

**使用场景**:
- 向量检索前对查询文本进行向量化
- 插入记忆前对内容进行向量化

### Rerank服务调用

**调用位置**: `service/src/business/memory/memory_common_manager.cpp` (第272-274行)

**方法**: `RagMgr::GetInstance()->GetRerank()->ReRank()`

**使用场景**:
- 检索结果重排序
- 提升相关性

```cpp
auto rerank = RagMgr::GetInstance()->GetRerank();
std::vector<DM::RAG::ReRankResult> rerankResults;
DM::RAG::ReRankOperateCode retCode = rerank->ReRank(
    rewriteQuery, mergedResults2Rerank, option, rerankResults);
```

### 数据库操作

**数据库类型**: GaussDB (支持向量和标量检索)

**主要操作**:
- 向量检索: `VectorSearch`
- 标量检索: `ScalarSearch`
- 混合检索: `FusionSearch`
- 插入: `Insert`
- 更新: `Update`
- 删除: `Delete`

**RAG管理器**: `service/include/database/rag/rag_mgr.h`

---

## 附录

### A. 核心类和方法列表

| 类名 | 职责 | 关键方法 | 文件路径 |
|-----|------|---------|---------|
| RouteMgr | 路由管理 | InitRestRouter, AddUserMemory, QueryUserMemory | route_mgr.h |
| UserMemoryService | 用户记忆服务 | OnReceiveUserMemoryAddMessage, OnReceiveUserMemoryQueryMessage | user_memory_service.h |
| UserMemoryManager | 用户记忆管理器 | Init, Excutor, DispatchUserMemoryByPolicy | user_memory_manager.h |
| MemoryCommonManager | 通用记忆管理 | ExtractFact, OnReceiveCommonQueryMessage, RetrieveMemory | memory_common_manager.h |
| MemoryMovieManager | 观影场景管理 | ExtractFact, OnReceiveQueryMessage | memory_movie_manager.h |
| MemoryTravelManager | 出行场景管理 | ExtractFact, OnReceiveTravelQueryMessage | memory_travel_manager.h |
| MemoryAgentManager | Agent场景管理 | ExtractFact, OnReceiveAgentQueryMessage | memory_agent_manager.h |
| QAShortMemoryService | 问答短记忆服务 | OnReceiveQABatchAddMsg, OnReceiveQAGetTopKMsg | qa_short_memory_service.h |

### B. 数据模型类

| 类名 | 职责 | 关键属性 | 文件路径 |
|-----|------|---------|---------|
| UserMemory | 用户记忆 | userId, content, topic, subTopic, label | user_memory.h |
| CoreMemory | 核心记忆 | userId, detail, topic, subTopic | core_memory.h |
| EpisodicMemory | 情景记忆 | userId, summary, treePath, source | episodic_memory.h |
| SemanticMemory | 语义记忆 | userId, summary, topic | semantic_memory.h |
| CommonMemory | 通用记忆基类 | userId, topic, detail, relevanceScore | common_memory.h |
| UserConversation | 用户对话 | userId, role, content, createDate | user_conversation.h |

### C. 数据库表操作类

| 类名 | 数据表 | 关键方法 | 文件路径 |
|-----|-------|---------|---------|
| UserTmpEpisodicTbl | tbl_user_tmp_episodic | Insert, QueryTempUserMemory, Delete | user_tmp_episodic_tbl.h |
| UserFactTbl | tbl_user_fact | InsertUserFactTbl, QueryCommonUserFact | user_fact_tbl.h |
| CoreMemoryTbl | tbl_core_memory | InsertCoreMemoryTbl, QueryCoreMemoryByUserTopicVector | core_memory_tbl.h |
| EpisodicMemoryTbl | tbl_episodic_memory | InsertEpisodicMemoryTbl, QueryEpisodicMemoryByTreePathVector | episodic_memory_tbl.h |
| SemanticMemoryTbl | tbl_semantic_memory | InsertSemanticMemoryTbl, QuerySemanticMemoryByTopicVector | semantic_memory_tbl.h |
| UserConversationTbl | tbl_user_conversation | Insert, GetLatestConversation | user_conversation_tbl.h |

### D. 配置管理

**配置管理器**: `ConfigMgr`

**文件路径**: `service/include/domain/config/config_mgr.h`

**主要配置项**:
- 表名映射
- Agent策略列表
- 数据库Schema
- 知识库映射
- 模型配置
- 记忆总结参数
- Rerank配置
- 黑名单配置

### E. 环境变量

| 变量名 | 说明 |
|--------|------|
| LLM_MODEL | LLM模型名称 |
| LLM_MODEL_SERVICE_NAME | LLM服务名称 |
| RERANK_MODEL | Rerank模型名称 |
| REMOTE_EMBEDDING_BASE_URL | 远程Embedding服务URL |

---

## 总结

### 系统特点

1. **分层清晰**: Controller → Business → Domain → Infrastructure 四层架构
2. **策略灵活**: 基于label的策略模式，支持多场景扩展
3. **并发高效**: 多线程池 + 并发执行器，提升处理能力
4. **混合架构**: C++主服务 + Python Agent，发挥各自优势
5. **智能检索**: 向量检索 + 文本检索 + 重排序，提升召回质量

### 核心流程

1. **记忆添加**: 接收 → 验证 → 临时存储 → 定时提取 → 策略分发 → 持久化
2. **记忆查询**: 接收 → 改写 → 并发检索 → 重排序 → 总结 → 返回
3. **对话管理**: 接收 → 存储 → 定时提取 → Python Agent → 多类型记忆
4. **画像构建**: 对话 → CoreMemoryAgent → 提取 → 存储 → 定时总结

### 关键技术

1. **向量检索**: GaussDB向量索引，支持L2距离算法
2. **重排序**: 内部Rerank + 外部Rerank，提升相关性
3. **LLM推理**: 查询改写、主题提取、记忆总结
4. **Agent架构**: Python Agent处理记忆提取，支持扩展
5. **定时任务**: 多种定时任务，支持记忆老化清理和总结

### 扩展点

1. **新增场景**: 在策略映射中添加新的label和Manager
2. **新增记忆类型**: 创建新的Memory类和Tbl类
3. **新增Agent**: 继承BaseMemoryAgent，实现抽象方法
4. **新增检索方式**: 在GetMemoryByQuery中添加新的检索任务
5. **新增定时任务**: 创建新的Scheduled类，注册定时器

---

**文档结束**

*本文档详细描述了KMM项目所有场景的代码运行流程，包含完整的流程图、关键代码路径、数据流转等信息。*
