# 学习路由器增强

## 概述

学习路由器使用 **epsilon-greedy** 算法，基于历史性能和探索需求智能选择策略，并结合上下文模式匹配。

## 算法设计

### Epsilon-Greedy 选择

路由器使用 10% 探索率的 epsilon-greedy 探索：
- 90% 的时间：基于历史成功率利用表现最佳的策略
- 10% 的时间：通过尝试不同策略进行探索

### 策略得分计算

路由器基于历史表现为每个策略计算得分：

```
得分 = 成功率 + 上下文提升 - 性能惩罚
```

### 组成部分

#### 1. 成功率（0.0 - 1.0）
- 基于历史成功/失败比率的基础成分
- 按近因加权（近期结果更重要）

```go
successRate := float64(successes) / float64(attempts)
```

#### 2. 探索（Epsilon-Greedy）
- 10% 的概率随机选择策略进行探索
- 确保对新模式的持续学习

```go
if rand.Float64() < 0.1 {  // 10% 探索
    return selectRandomStrategy()
}
```

#### 3. 惩罚

**延迟惩罚**（0.0 - 0.2）：
```go
if avgLatency > targetLatency {
    penalty = min(0.2, (avgLatency - targetLatency) / targetLatency * 0.1)
}
```

**令牌效率惩罚**（0.0 - 0.15）：
```go
if avgTokens > budgetTarget {
    penalty = min(0.15, (avgTokens - budgetTarget) / budgetTarget * 0.1)
}
```

#### 4. 关键词提升（0.0 - 0.1）
- 提升匹配查询模式的策略
- 基于 TF-IDF 相似度

```go
if keywordMatch > threshold {
    boost = min(0.1, similarity * 0.2)
}
```

### 最终置信度

置信度被限制在 [0, 1] 范围内：

```go
confidence := max(0.0, min(1.0, ucbScore))
```

## 实现

### RecommendWorkflowStrategy

```go
func RecommendWorkflowStrategy(ctx context.Context,
    input RecommendStrategyInput) (RecommendStrategyOutput, error) {

    // 从监督器内存中获取历史模式
    patterns := SearchDecompositionPatterns(input.SessionID, input.Query)

    // Epsilon-greedy 选择
    if rand.Float64() < 0.1 { // 10% 探索
        return selectRandomStrategy()
    }

    // 为每个策略计算得分
    scores := make(map[StrategyType]float64)
    for strategy, stats := range patterns {
        score := calculateStrategyScore(strategy, stats, input.Query)
        scores[strategy] = score
    }

    // 选择最高得分的策略
    best := selectBestStrategy(scores)

    return RecommendStrategyOutput{
        Strategy:   best.Strategy,
        Confidence: best.Confidence,
        Source:     "epsilon_greedy",
    }, nil
}
```

### 策略得分计算

```go
func calculateStrategyScore(strategy StrategyType,
    stats StrategyStats, query string) float64 {

    // 基于历史数据的基准成功率
    successRate := stats.SuccessRate()

    // 延迟惩罚
    latencyPenalty := 0.0
    if stats.AvgLatencyMs > targetLatencyMs {
        ratio := (stats.AvgLatencyMs - targetLatencyMs) / targetLatencyMs
        latencyPenalty = math.Min(0.2, ratio * 0.1)
    }

    // 令牌效率惩罚
    tokenPenalty := 0.0
    if stats.AvgTokens > targetTokens {
        ratio := (stats.AvgTokens - targetTokens) / targetTokens
        tokenPenalty = math.Min(0.15, ratio * 0.1)
    }

    // 来自查询相似度的上下文提升
    contextBoost := calculateContextualBoost(strategy, query)

    // 综合得分
    score := successRate - latencyPenalty - tokenPenalty + contextBoost

    // 限制在 [0, 1]
    return math.Max(0.0, math.Min(1.0, score))
}
```

## 策略模式

### 模式识别

系统为每个策略学习查询模式：

```go
type PatternSignature struct {
    Keywords     []string
    Complexity   float64
    ToolsNeeded  []string
    OutputFormat string
}

func matchPattern(query string,
    signature PatternSignature) float64 {
    // TF-IDF 相似度
    keywordScore := tfidfSimilarity(query, signature.Keywords)

    // 复杂度匹配
    queryComplexity := estimateComplexity(query)
    complexityMatch := 1.0 - math.Abs(queryComplexity -
                                      signature.Complexity)

    // 工具需求匹配
    requiredTools := detectRequiredTools(query)
    toolMatch := jaccardSimilarity(requiredTools,
                                   signature.ToolsNeeded)

    // 加权组合
    return keywordScore*0.5 + complexityMatch*0.3 + toolMatch*0.2
}
```

### 策略画像

每个策略都有一个学习到的画像：

| 策略 | 关键词 | 复杂度 | 典型工具 | 成功领域 |
|----------|----------|------------|---------------|-----------------|
| Tree of Thoughts | 分析、探索、比较、评估 | 高 | web_search、calculator | 研究、分析 |
| Chain of Thought | 解释、描述、总结 | 中 | web_search | 解释、摘要 |
| ReAct | 查找、获取、检索、计算 | 低 | 所有工具 | 数据检索、计算 |
| Debate | 优缺点、比较、论证 | 高 | web_search | 决策制定 |
| Reflection | 改进、完善、审查 | 中 | 无 | 质量改进 |

## 置信度评分

### 置信度级别

| 置信度 | 含义 | 操作 |
|------------|---------|--------|
| 0.9 - 1.0 | 非常高 | 使用推荐策略 |
| 0.7 - 0.89 | 高 | 允许模式降级时使用 |
| 0.5 - 0.69 | 中 | 考虑替代方案 |
| 0.3 - 0.49 | 低 | 偏好探索 |
| 0.0 - 0.29 | 非常低 | 使用默认策略 |

### 置信度因素

置信度受以下因素影响：

1. **样本量**：更多数据 → 更高置信度
2. **近因**：近期成功 → 更高置信度
3. **一致性**：稳定表现 → 更高置信度
4. **上下文匹配**：相似查询 → 更高置信度

```go
func adjustConfidence(baseConfidence float64,
    stats StrategyStats) float64 {

    // 样本量因子
    sampleFactor := math.Min(1.0, float64(stats.Attempts) / 100.0)

    // 近因因子（指数衰减）
    recencyFactor := calculateRecencyWeight(stats.LastSuccess)

    // 一致性因子（低方差 = 高一致性）
    consistencyFactor := 1.0 - stats.SuccessVariance()

    // 加权调整
    adjusted := baseConfidence *
                (0.4 + sampleFactor*0.2 +
                 recencyFactor*0.2 +
                 consistencyFactor*0.2)

    return math.Max(0.0, math.Min(1.0, adjusted))
}
```

## 存储与检索

### 向量存储集成

模式存储在 Qdrant 中，带有嵌入：

```go
type DecompositionPattern struct {
    ID           string
    Query        string
    QueryVector  []float32  // 嵌入
    Strategy     string
    Success      bool
    TokensUsed   int
    LatencyMs    int64
    Timestamp    time.Time
    Metadata     map[string]interface{}
}
```

### 相似度搜索

查找相似模式：

```go
func findSimilarPatterns(query string, limit int) []Pattern {
    // 生成查询嵌入
    embedding := generateEmbedding(query)

    // 在 Qdrant 中搜索
    results := qdrant.Search(
        collection: "decomposition_patterns",
        vector: embedding,
        limit: limit,
        threshold: 0.75,  // 最小相似度
    )

    return results
}
```

## 指标与监控

### 关键指标

```prometheus
# 策略选择分布
shannon_strategy_selection_total{strategy="tree_of_thoughts"} 142
shannon_strategy_selection_total{strategy="chain_of_thought"} 298
shannon_strategy_selection_total{strategy="react"} 521

# 记录的分解模式
shannon_decomposition_patterns_recorded_total 961

# 学习来源分布
shannon_strategy_selection_total{source="epsilon_greedy"} 865
shannon_strategy_selection_total{source="exploration"} 96

# 模式缓存性能
shannon_decomposition_pattern_cache_hits_total 432
shannon_decomposition_pattern_cache_misses_total 529
```

### 性能跟踪

```sql
-- 随时间变化的策略表现
SELECT
    strategy,
    DATE(timestamp) as date,
    AVG(CASE WHEN success THEN 1.0 ELSE 0.0 END) as success_rate,
    AVG(tokens_used) as avg_tokens,
    AVG(latency_ms) as avg_latency,
    COUNT(*) as attempts
FROM decomposition_patterns
WHERE timestamp > NOW() - INTERVAL '7 days'
GROUP BY strategy, date
ORDER BY date DESC, strategy;
```

## 配置

### 调优参数

```yaml
learning_router:
  # 探索率（epsilon-greedy）
  epsilon: 0.1

  # UCB 参数
  exploration_weight: 2.0  # UCB 公式中的 C

  # 惩罚阈值
  target_latency_ms: 5000
  target_tokens: 3000

  # 置信度阈值
  min_confidence: 0.3
  high_confidence: 0.7

  # 历史窗口
  pattern_retention_days: 30
  max_patterns_per_session: 1000

  # 关键词匹配
  keyword_boost_weight: 0.1
  min_keyword_similarity: 0.6
```

### A/B 测试

通过受控实验比较策略：

```yaml
experiments:
  strategy_comparison:
    enabled: true
    control_strategy: "chain_of_thought"
    test_strategy: "ucb_router"
    traffic_split: 0.5
    metrics:
      - success_rate
      - avg_tokens
      - avg_latency
    duration_hours: 168  # 1 周
```

## 最佳实践

### 1. 冷启动处理

对于没有历史记录的新会话，使用启发式方法：

```go
if len(patterns) == 0 {
    // 探索模式：随机选择以构建初始数据
    if rand.Float64() < 0.3 {  // 冷启动期间 30% 探索
        return selectRandomStrategy()
    }
    // 否则使用查询复杂度启发式
    complexity := estimateComplexity(query)
    if complexity > 0.7 {
        return "tree_of_thoughts", 0.5
    } else if complexity > 0.4 {
        return "chain_of_thought", 0.5
    }
    return "react", 0.5
}
```

### 2. 策略回退

始终有回退链：

```go
strategies := []StrategyType{
    recommendedStrategy,
    "chain_of_thought",  // 回退 1
    "react",             // 回退 2
}
```

### 3. 持续学习

每次执行后更新模式：

```go
defer recordPattern(PatternRecord{
    Query:      input.Query,
    Strategy:   selectedStrategy,
    Success:    result.Success,
    TokensUsed: result.TokensUsed,
    LatencyMs:  elapsed.Milliseconds(),
})
```

### 4. 隐私考虑

- 存储前对模式进行匿名化
- 对聚合数据使用差分隐私
- 实施数据保留策略
- 允许用户选择退出

## 未来增强

1. **Thompson Sampling**：用于探索/利用的贝叶斯方法
2. **上下文赌博机**：考虑额外上下文特征进行选择
3. **深度学习**：基于神经网络的策略预测
4. **迁移学习**：跨相似领域共享模式
5. **动态探索**：基于性能方差调整 epsilon 率
6. **分层策略**：多级策略选择

## 算法说明

当前实现使用 **epsilon-greedy**（而非 UCB），因为：
- 更易于实现和理解
- 无需跟踪每个策略的访问次数
- 探索行为更具可预测性
- 在历史数据有限时表现良好

未来版本可能探索 UCB 变体（UCB1、Thompson Sampling）以实现更复杂的探索策略。
