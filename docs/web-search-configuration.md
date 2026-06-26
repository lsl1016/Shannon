# Web 搜索配置

Shannon 平台支持多个 Web 搜索提供商，为 AI agents 提供实时信息。通过环境变量配置你首选的提供商。

## 支持的提供商

### 1. **Google Custom Search**（使用最广泛）

行业标准搜索，覆盖范围全面，结果丰富。

```bash
export WEB_SEARCH_PROVIDER=google
export GOOGLE_SEARCH_API_KEY=your_api_key_here
export GOOGLE_SEARCH_ENGINE_ID=your_search_engine_id_here
```

* 在此获取 API key：[https://console.cloud.google.com/apis/credentials](https://console.cloud.google.com/apis/credentials)
* 在此创建搜索引擎：[https://programmablesearchengine.google.com/](https://programmablesearchengine.google.com/)
* 免费层级：100 次查询/天，之后每 1000 次查询 $5
* 速率限制：每 100 秒 100 次查询

### 2. **Serper**（对开发者最具性价比）

快速、实惠的 Google 搜索结果 API，定价简单。

```bash
export WEB_SEARCH_PROVIDER=serper
export SERPER_API_KEY=your_api_key_here
```

* 在此获取 API key：[https://serper.dev](https://serper.dev)
* 免费层级：注册时赠送 2500 次查询
* 定价：50k 次查询起价 $50（$1.00/1k）
* 速率限制：300 次查询/秒

### 3. **SerpAPI**（稳健的 Google Search API）

可靠的 Google Search、Maps、News 等 API。能有效处理代理和验证码。

```bash
export WEB_SEARCH_PROVIDER=serpapi
export SERPAPI_API_KEY=your_api_key_here
```

* 在此获取 API key：[https://serpapi.com](https://serpapi.com)
* 免费层级：100 次搜索/月
* 定价：5k 次搜索起价 $50
* 速率限制：根据套餐灵活调整

### 4. **Bing Search API**（企业选择）

Microsoft 的搜索 API，集成 Azure。

```bash
export WEB_SEARCH_PROVIDER=bing
export BING_API_KEY=your_api_key_here
```

* 在此获取 API key：[https://azure.microsoft.com/en-us/services/cognitive-services/bing-web-search-api/](https://azure.microsoft.com/en-us/services/cognitive-services/bing-web-search-api/)
* 免费层级：1000 次查询/月
* 定价：S1 层级每 1000 次查询 $3
* 注意：Bing Search APIs 将于 2025 年 8 月 11 日退役，请考虑迁移计划

### 5. **Exa**（面向 AI 的最快且最准确的 Web Search API）

具备语义理解的神经搜索，针对 AI 应用优化。

```bash
export WEB_SEARCH_PROVIDER=exa
export EXA_API_KEY=your_api_key_here
```

* 在此获取 API key：[https://exa.ai](https://exa.ai)
* 特性：语义搜索、自动提示、摘要高亮提取
* 免费层级：1000 次查询/月
* 定价：每次搜索 $0.001

### 6. **Firecrawl**（搜索 + 内容提取）

集成爬取和 markdown 提取的 Web 搜索。

```bash
export WEB_SEARCH_PROVIDER=firecrawl
export FIRECRAWL_API_KEY=your_api_key_here
```

* 在此获取 API key：[https://firecrawl.dev](https://firecrawl.dev)
* 特性：完整内容提取、markdown 格式化
* 免费层级：有限 alpha 访问
* 定价：根据爬取深度变化

## Docker Compose 配置

添加到你的 `deploy/compose/.env` 文件：

```env
# Web Search Configuration (choose one provider)
WEB_SEARCH_PROVIDER=google

# Google Custom Search (recommended)
GOOGLE_SEARCH_API_KEY=your_google_api_key_here
GOOGLE_SEARCH_ENGINE_ID=your_search_engine_id_here

# Or Serper (simple and affordable)
# WEB_SEARCH_PROVIDER=serper
# SERPER_API_KEY=your_serper_api_key_here

# Or SerpAPI (robust scraping)
# WEB_SEARCH_PROVIDER=serpapi
# SERPAPI_API_KEY=your_serpapi_api_key_here

# Or Bing (enterprise)
# WEB_SEARCH_PROVIDER=bing
# BING_API_KEY=your_bing_api_key_here

# Or Exa (semantic AI search)
# WEB_SEARCH_PROVIDER=exa
# EXA_API_KEY=your_exa_api_key_here

# Or Firecrawl (with content extraction)
# WEB_SEARCH_PROVIDER=firecrawl
# FIRECRAWL_API_KEY=your_firecrawl_api_key_here
```

环境变量已经在 `deploy/compose/docker-compose.yml` 中配置：

```yaml
llm-service:
  environment:
    - WEB_SEARCH_PROVIDER=${WEB_SEARCH_PROVIDER:-google}
    - GOOGLE_SEARCH_API_KEY=${GOOGLE_SEARCH_API_KEY}
    - GOOGLE_SEARCH_ENGINE_ID=${GOOGLE_SEARCH_ENGINE_ID}
    - SERPER_API_KEY=${SERPER_API_KEY}
    - BING_API_KEY=${BING_API_KEY}
    - SERPAPI_API_KEY=${SERPAPI_API_KEY}
    - EXA_API_KEY=${EXA_API_KEY}
    - FIRECRAWL_API_KEY=${FIRECRAWL_API_KEY}
```

## 回退行为

如果配置的提供商不可用（缺少 API key 或配置），系统会按以下优先级自动尝试其他提供商：

1. Google Custom Search
2. Serper
3. SerpAPI
4. Bing
5. Exa
6. Firecrawl

如果没有配置任何提供商，Web 搜索将被禁用，但系统会继续运行。

## 响应格式

所有提供商都会返回带有以下字段的标准化结果：

* `title`：结果标题
* `snippet`：短文本预览
* `content`：扩展内容（可用时）
* `url`：结果 URL
* `source`：提供商名称
* 其他提供商特定字段（score、date、highlights 等）

## 测试你的配置

测试你的 Web 搜索配置：

```bash
curl -X POST http://localhost:8000/tools/execute \
  -H "Content-Type: application/json" \
  -d '{
    "tool_name": "web_search",
    "parameters": {
      "query": "latest AI news",
      "max_results": 5
    }
  }'
```

## 提供商对比

| 提供商           | 最适合            | 定价（每 1k 次查询） | 速率限制     | 内容深度           |
| ------------- | -------------- | ------------ | -------- | -------------- |
| **Google**    | 通用搜索、全面结果      | 免费层级后 $5     | 100/100s | 摘要 + 元数据       |
| **Serper**    | 高流量、成本效益高      | $0.30-$1.00  | 300/s    | 摘要 + 知识图谱      |
| **SerpAPI**   | 爬取可靠性和多样性      | $10          | 不同套餐不同   | 摘要 + 丰富结果数据    |
| **Bing**      | 企业、Azure 集成    | $3           | 根据层级变化   | 摘要             |
| **Exa**       | AI agents、语义搜索 | $1           | 标准       | 全文 + 高亮        |
| **Firecrawl** | 内容提取           | 可变           | 有限       | 完整 markdown 内容 |

## 设置 Google Custom Search

1. **创建 Custom Search Engine：**

   * 前往 [https://programmablesearchengine.google.com/](https://programmablesearchengine.google.com/)
   * 点击 "Add" 创建新的搜索引擎
   * 配置为搜索整个 Web
   * 记录你的 Search Engine ID

2. **获取 API Key：**

   * 前往 [https://console.cloud.google.com/](https://console.cloud.google.com/)
   * 创建新项目或选择已有项目
   * 启用 "Custom Search API"
   * 创建凭据（API Key）
   * 将 key 限制为 Custom Search API

3. **配置 Shannon：**

   ```bash
   export GOOGLE_SEARCH_API_KEY=your_key
   export GOOGLE_SEARCH_ENGINE_ID=your_engine_id
   export WEB_SEARCH_PROVIDER=google
   ```

## 最佳实践

1. **从 Google 或 Serper 开始**，以获得广泛兼容性和可靠结果
2. **使用 Exa** 满足 AI 特定的语义搜索需求
3. **配置多个提供商** 以实现冗余
4. **监控用量**，保持在免费层级或预算范围内
5. **尽可能缓存结果**，减少 API 调用

## 故障排查

如果 Web 搜索无法工作：

1. 检查环境变量是否正确设置
2. 验证 API keys 是否有效且有足够配额
3. 检查日志：`docker compose logs llm-service`
4. 使用 curl 直接测试提供商以隔离问题
5. 确保部署环境具备网络连接

对于提供商特定问题，请查阅各自的文档和状态页。
