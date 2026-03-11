# NullClaw OpenAI Responses 开发任务清单

本文档用于指导后续开发执行，要求每完成一项就更新状态，避免重复劳动或遗漏。

## 当前分支

- `codex/openai-responses-design`

## 当前目标

- 为 `src/providers/compatible.zig` 增加 OpenAI Responses 主协议支持
- 让只支持 `/responses` 的中转服务可在 NullClaw 中正常使用
- 第一阶段先完成“非流式完整闭环”

## 状态说明

- `[x]` 已完成
- `[-]` 进行中
- `[ ]` 未开始

## 任务清单

### 1. 设计与方案

- [x] 分析 `openclaw` 的 Responses 实现方式
- [x] 分析 `nullclaw` 当前 compatible provider 的限制
- [x] 输出英文设计文档 `OPENAI_RESPONSES_PROVIDER_DESIGN.md`
- [x] 输出中文设计文档 `OPENAI_RESPONSES_PROVIDER_DESIGN_ZH.md`

### 2. 分支与工作区准备

- [x] 在 `workspace/nullclaw` 建立开发分支 `codex/openai-responses-design`
- [x] 确认本地仓库可用于直接开发

### 3. 配置层改造

- [x] 在 provider 配置模型中增加 `api_mode`
- [x] 支持解析 `models.providers.<name>.api_mode`
- [x] 增加配置读取 getter（`getProviderApiMode`）
- [x] 增加配置保存逻辑
- [x] 增加配置校验：非法 `api_mode` 报错

### 4. Provider 构造链路改造

- [x] 将 `api_mode` 从配置透传到 `ProviderHolder.fromConfig`
- [x] 更新 `runtime_bundle` 透传链路
- [x] 更新 `provider_probe` 透传链路
- [x] 更新 `subagent_runner` 透传链路
- [x] 更新 `tools/delegate.zig` 透传链路
- [x] 更新/修复相关测试签名

### 5. Compatible Provider 主逻辑改造

- [x] 在 `OpenAiCompatibleProvider` 中增加 `api_mode`
- [x] 为 Responses 增加完整请求构造（基于 `ChatRequest`）
- [x] 支持从 system 消息提取 `instructions`
- [x] 支持 user/assistant/tool 历史映射到 `input`
- [x] 支持 tools -> Responses `tools`
- [x] 支持 `reasoning_effort` -> `reasoning.effort`
- [x] 支持 `max_tokens` -> `max_output_tokens`
- [x] 在 `chatImpl` 中按 `api_mode` 分流
- [x] 对 `chat_completions` 模式保留 fallback 兼容
- [x] `chatWithSystemImpl` 支持 Responses 模式
- [x] `streamChatImpl` 在 Responses 模式下安全降级到非流式

### 6. Responses 响应解析

- [x] 解析 assistant 文本输出
- [x] 解析 function_call 输出项
- [x] 解析 usage
- [x] 解析 model 字段
- [x] 对 reasoning 项做兼容处理（至少不报错）
- [x] 支持“文本 + function_call”混合响应

### 7. 图片输入支持

- [x] 将 `content_parts.text` 映射为 `input_text`
- [x] 将 `content_parts.image_url` 映射为 `input_image(url)`
- [x] 将 `content_parts.image_base64` 映射为 `input_image(base64)`

### 8. 测试

- [x] 补配置解析测试：`api_mode = responses`
- [x] 补配置保存测试：能写出 `api_mode`
- [x] 补 factory 测试：`ProviderHolder.fromConfig` 正确设置 `api_mode`
- [x] 补 Responses request builder 测试：system + user
- [x] 补 Responses request builder 测试：tools
- [x] 补 Responses request builder 测试：tool result
- [x] 补 Responses request builder 测试：image parts
- [x] 补 Responses response parser 测试：纯文本
- [x] 补 Responses response parser 测试：纯 function_call
- [x] 补 Responses response parser 测试：文本 + function_call 混合
- [x] 修复 `tools/delegate.zig` 相关测试因签名变化带来的失败

### 9. 验证

- [x] 运行 `zig build`
- [x] 运行 `zig build test --summary all`
- [x] 如时间允许，验证 `ReleaseSmall` 构建
- [x] 用真实 Responses provider 做一次最小验证

## 当前实际进度

截至当前这次开发，已经完成：

- 设计文档
- 分支创建
- 配置层 `api_mode` 基础接入
- provider 构造链路 `api_mode` 透传
- `compatible.zig` 中 `api_mode` 字段接入

当前正在做：

- 准备整理最终交付说明

## 接手提醒

- 当前最关键文件是 `src/providers/compatible.zig`
- 在继续改代码前，先看：
  - `OPENAI_RESPONSES_PROVIDER_DESIGN_ZH.md`
  - `src/providers/compatible.zig`
  - `src/providers/factory.zig`
  - `src/config.zig`
  - `src/config_parse.zig`
  - `src/config_types.zig`

- 每完成一个阶段，请同步更新本文档状态
