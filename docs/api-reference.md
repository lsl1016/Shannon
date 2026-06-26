# API 参考文档（LLM 服务）

## POST /agent/query

请求体（仅展示最相关的字段）：
- `query` (字符串) – 任务或问题。
-`context` (对象，可选) – 附加参数；可能包含 `role`、`model_tier`、`prompt_params`、`history`、`attachments`。
- 'context'（对象，可选） – 附加参数; 可能包含 'role'、 'model_tier'、 'prompt_params'、 'history'， 'attachments'
-'context'（对象，可选） – 附加参数; 可能包含 'role'、 'model_tier'、 'prompt_params'、 'history'， 'attachments'。
 - `attachments` (数组，可选) – 文件附件，格式为 `[{id, media_type, filename, size_bytes}]` 引用（由 agent 从 Redis 解析）。
- `agent_id` (字符串，可选) – 用于可观测性的标识符。
- `allowed_tools` (字符串数组，可选) – 显式工具白名单。
 - 省略或为 `null` → 角色预设可能启用工具。
  - `[]` → 禁用工具。
  - 非空列表 → 仅这些工具可用（名称必须与已注册的工具匹配：内置工具、OpenAPI、MCP）。
-`model_tier` (字符串，可选) – `small|medium|large`。
-'model_tier'（字符串，可选） – 'small|medium|large'。
- `model_override` (字符串，可选) – 特定提供商的模型 ID。
- `max_tokens` (整数，可选) – 响应长度限制。
- `temperature` (浮点数，可选) – 采样温度。

响应体：
- `success` (布尔值)
- `response` (字符串) – 最终回答文本。
- `tokens_used` (整数) – 总 token 数（提示词 + 补全）。
- `model_used` (字符串)
- `provider` (字符串)
- `metadata` (对象) – 可能包含 `allowed_tools`、`role`。

备注：
- GPT‑5 模型会被路由到 Responses API；服务端在有 `output_text` 时会优先使用它。
- 聊天提供商会进行防御性规范化处理，当返回列表时会合并各文本部分。
