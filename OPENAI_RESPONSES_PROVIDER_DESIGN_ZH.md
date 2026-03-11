# NullClaw OpenAI Responses Provider 详细设计

## 一、背景

NullClaw 当前对 OpenAI-compatible provider 的主支持路径位于 `src/providers/compatible.zig`，核心协议是 `/chat/completions`。

这条路径对支持 Chat Completions 的提供方没有问题，但对只支持 `/responses` 的模型中转服务会失败。当前用户遇到的实际现象是：

- provider 配置能加载成功
- 模型名可以正常识别
- 真正发请求时，NullClaw 仍然请求 `/chat/completions`
- 中转服务返回错误，提示客户端应改用 `/responses`

因此，问题本质不是 `gpt-5.4` 不能用，而是 NullClaw 当前版本对 OpenAI Responses API 没有做“一等公民级别”的完整支持。

## 二、问题定义

NullClaw 需要为 OpenAI Responses API 提供正式支持，使其能够接入以下类型的服务：

- 只暴露 `/responses` 的 OpenAI 风格网关
- 通过中转方式接入 OpenAI 新模型的服务
- 未来优先支持 Responses API 的兼容 provider

当前 `src/providers/compatible.zig` 中虽然已经存在一个 `/responses` fallback，但它存在以下限制：

1. `shouldFallbackToResponses` 只识别非常狭窄的 404 错误形态。
2. `chatViaResponses` 只支持单轮文本输入。
3. 当前 Responses 解析只会提取纯文本输出。
4. 工具调用、工具结果、多轮历史、图片输入、reasoning 字段、流式行为都没有完整覆盖。

所以，这个 fallback 只够“临时兜底”，不够支撑 NullClaw 的主 agent 回路。

## 三、设计目标

本设计的目标如下：

1. 为 OpenAI-compatible provider 增加正式的 Responses 模式。
2. 保持现有 Chat Completions provider 的行为不变。
3. 在第一阶段支持完整的非流式 agent 主链路：
   - 多轮上下文
   - tools
   - tool result continuation
   - 图片输入
   - reasoning effort 透传
4. 不改变现有 provider vtable 接口。
5. 为后续 streaming / `previous_response_id` 扩展留好结构。

## 四、非目标

本次不做以下事情：

1. 不把 OpenClaw 的 gateway 子系统整套移植到 NullClaw。
2. 第一阶段不实现 WebSocket transport。
3. 不新增一套完全独立的新 provider 家族；优先扩展现有 compatible provider。
4. 不改动与本问题无关的 channel、memory、runtime 行为。

## 五、NullClaw 当前现状

### 5.1 相关文件

- `src/providers/compatible.zig`
- `src/providers/root.zig`
- `src/providers/factory.zig`
- `src/config.zig`
- `src/config_types.zig`
- `src/providers/runtime_bundle.zig`
- `src/subagent.zig`

### 5.2 当前已有能力

虽然 Responses 路径不完整，但 NullClaw 在 Chat Completions 路径中，实际上已经具备了构造完整请求所需的大部分内部信息：

- 完整 `ChatRequest`
- 消息历史
- 工具定义
- 图文 content parts
- reasoning effort
- max token 限制

其中已经存在的关键能力包括：

- `serializeMessagesInto`：可序列化消息历史
- `root.convertToolsOpenAI`：可生成 OpenAI 风格 tools 定义
- `root.appendGenerationFields`：可附加 temperature / max_tokens / reasoning 等生成参数
- provider 层兼容开关：`merge_system_into_user`、`thinking_param`、`reasoning_split_param`

### 5.3 当前缺口

当前的 Responses 路径过早退化成“单轮文本模式”：

- 不是基于 `ChatRequest` 构造完整请求
- 而是只接收 `(system_prompt, message, model)`

因此它无法表达真实 agent 运行时的：

- 历史 assistant 消息
- tool call
- tool result
- 多模态内容
- stop reason / structured output

## 六、OpenClaw 对照分析结论

OpenClaw 之所以能接入这类中转，并不是因为它“特殊支持 gpt-5.4”，而是因为它把 `openai-responses` 当成正式 API 类型来实现。

最值得参考的文件：

- `openclaw/src/config/types.models.ts`
- `openclaw/src/agents/pi-embedded-runner/model.provider-normalization.ts`
- `openclaw/src/agents/pi-embedded-runner/run/attempt.ts`
- `openclaw/src/agents/openai-ws-connection.ts`
- `openclaw/src/agents/openai-ws-stream.ts`
- `openclaw/src/gateway/open-responses.schema.ts`
- `openclaw/src/gateway/openresponses-prompt.ts`
- `openclaw/src/gateway/openresponses-http.ts`
- `openclaw/src/media/input-files.ts`

从 OpenClaw 得出的关键结论：

1. Responses 需要被显式建模，而不是作为 fallback 猜测。
2. 请求输入应以 typed items 表示，而不是把所有历史压扁成一个文本 prompt。
3. 图片输入应按 `input_image` 原生结构传递。
4. 工具续轮应通过 `function_call_output` 回传。
5. `previous_response_id` 是优化项，不是第一阶段必须项。

对 NullClaw 来说，最重要的不是照搬 OpenClaw 的工程结构，而是借鉴它的协议建模方式。

## 七、总体方案

### 7.1 总体思路

在现有 `OpenAiCompatibleProvider` 上增加一个显式 API 模式开关：

```zig
pub const CompatibleApiMode = enum {
    chat_completions,
    responses,
};
```

并增加字段：

```zig
api_mode: CompatibleApiMode = .chat_completions,
```

该字段用于决定 provider 的主请求协议。

### 7.2 为什么要显式模式，而不是继续依赖 fallback

仅靠 fallback 存在以下问题：

1. 不同服务返回的错误码和错误文案不一致。
2. 有些服务对旧协议返回 400/405，而不是 404。
3. 工具调用和 streaming 场景不适合依赖“先失败一次再回退”。
4. 选择主协议本质上是 provider 配置问题，不应靠运行时猜测。

因此，本设计建议：

- 显式配置 `responses` 模式
- fallback 只保留给历史兼容场景

## 八、配置层设计

### 8.1 新增 provider 配置字段

建议在 provider 配置中增加字段：

```json
"api_mode": "responses"
```

允许值：

- `chat_completions`
- `responses`

### 8.2 行为规则

- 未配置时，默认值为 `chat_completions`
- 配置为 `responses` 时，provider 直接请求 `/responses`
- 配置为未知值时，配置加载直接报错

### 8.3 涉及文件

- `src/config_types.zig`
- `src/config.zig`
- `src/providers/runtime_bundle.zig`
- 可能还包括 provider factory 的构造链路

## 九、Provider 结构设计

在 `src/providers/compatible.zig` 中为 `OpenAiCompatibleProvider` 增加：

```zig
pub const CompatibleApiMode = enum {
    chat_completions,
    responses,
};

api_mode: CompatibleApiMode = .chat_completions,
```

这是本设计最核心、也是对现有 vtable 架构影响最小的一步。

## 十、Responses 请求映射设计

### 10.1 顶层字段映射

建议按如下方式从 `ChatRequest` 映射到 Responses 请求：

- `model` -> `model`
- `request.max_tokens` -> `max_output_tokens`
- `request.reasoning_effort` -> `reasoning.effort`
- `request.tools` -> `tools`
- 如果存在 tools -> `tool_choice: "auto"`
- 第一阶段固定 `stream: false`

### 10.2 system / developer 消息处理

Responses 提供了 `instructions` 字段，因此建议：

- 收集所有 `system` 消息
- 以 `"\n\n"` 拼接成一个字符串
- 写入 `instructions`
- 不再把它们重复写入 `input`

原因：

- 简单明确
- 语义上更贴近 system prompt
- 避免重复注入

### 10.3 user / assistant 消息处理

#### user 消息

- 纯文本：
  ```json
  {"type":"message","role":"user","content":"..."}
  ```
- 图文混合：
  ```json
  {"type":"message","role":"user","content":[...]}
  ```

#### assistant 文本消息

- 纯文本 assistant 输出：
  ```json
  {"type":"message","role":"assistant","content":"..."}
  ```
- 多 part assistant 文本：仅保留可安全回传的文本 part

第一阶段不要求把内部 reasoning block 回传给模型；如存在 reasoning 内容，默认忽略即可。

### 10.4 tool call 映射

assistant 的工具调用应映射为 Responses 的 `function_call` item。

必须包含：

- `call_id`
- `name`
- `arguments`（JSON string）

若同一 assistant 消息中既有文本又有 tool call，建议按顺序输出：

1. 先输出 assistant 文本 `message`
2. 再输出一个或多个 `function_call`

这样可最大化保留原始语义顺序。

### 10.5 tool result 映射

tool result 应映射为：

```json
{
  "type": "function_call_output",
  "call_id": "...",
  "output": "..."
}
```

第一阶段统一将 tool result 作为文本输出；如果内部是结构化 JSON，则先 stringify。

### 10.6 图片输入映射

利用现有 compatible provider 的 vision 数据结构，将图片 part 映射为 Responses `input_image`。

URL 图片：

```json
{"type":"input_image","source":{"type":"url","url":"..."}}
```

base64 图片：

```json
{"type":"input_image","source":{"type":"base64","media_type":"image/png","data":"..."}}
```

注意：

- NullClaw 在这里是客户端，不是 gateway
- 第一阶段不要在 provider 层新增远程抓图或文件提取逻辑
- 只需把内部已有图像数据按上游协议正确转发出去

## 十一、Responses 响应解析设计

建议新增一个专用解析函数：

```zig
fn parseResponsesNativeResponse(
    allocator: std.mem.Allocator,
    body: []const u8,
) !ChatResponse
```

### 11.1 文本输出解析

从 `output[*]` 中提取：

- `type == "message"`
- `role == "assistant"`
- `content[*].type == "output_text"`

按顺序拼接非空文本。

### 11.2 工具调用解析

从 `output[*]` 中提取：

- `type == "function_call"`

然后映射成 NullClaw 内部已有的 tool-call 响应结构。

这是实现 agent 工具回路的关键，必须支持。

### 11.3 文本 + 工具调用混合输出

Responses 可能在同一响应中同时返回：

- assistant 文本
- function_call

要求：

- 如果 `ChatResponse` 能同时承载两者，则都保留
- 如果内部 stop reason 只能选一个，则只要存在 tool call，就优先判定为 tool-call turn
- 同时尽量保留文本内容，避免丢失模型的解释性输出

### 11.4 reasoning item 处理

第一阶段中：

- 遇到 `reasoning` item 时不报错
- 默认忽略其内容
- 为后续扩展预留解析位置

### 11.5 usage 映射

若返回体中包含 usage 信息，应尽可能映射到现有 `ChatResponse` usage 字段中。

## 十二、Provider 执行路径改造

### 12.1 当前行为

当前 `chatImpl`：

- 总是构造 Chat Completions body
- 总是 POST 到 `/chat/completions`

### 12.2 目标行为

改造成：

```zig
switch (self.api_mode) {
    .chat_completions => { 现有逻辑 }
    .responses => { 新的 Responses 逻辑 }
}
```

新的 Responses 主路径应完成：

1. 使用 `responsesUrl(...)`
2. 从完整 `ChatRequest` 构造 Responses JSON
3. 复用现有认证头逻辑
4. 用新的 `parseResponsesNativeResponse(...)` 解析响应

## 十三、fallback 策略

在显式 `api_mode` 落地后：

- `responses` 模式不再依赖 fallback
- `chat_completions` 模式仍可保留 fallback 作为历史兼容兜底

建议对 fallback 做轻微增强：

- 继续保留现有 404 检测
- 可适当扩展到明确指向 `/responses` 的 400/405 错误
- 但不能把它作为 Responses-only provider 的主方案

## 十四、Phase 2 流式设计预留

第一阶段可以先不做 streaming，但序列化和解析结构必须可复用。

未来 streaming 需要支持的事件包括：

- `response.created`
- `response.output_item.added`
- `response.output_text.delta`
- `response.output_item.done`
- `response.completed`
- `response.failed`

设计建议：

- 请求序列化逻辑做成可复用 helper
- item 级别的解析逻辑可同时服务于“整包解析”和“SSE 增量解析”

## 十五、Phase 3 可选优化：previous_response_id

`previous_response_id` 第一阶段不必实现。

NullClaw 即使每轮都重发完整历史，只要 Responses input 映射正确，也已经可以正常完成 tool continuation。

后续稳定后可继续优化：

- 按 session 跟踪 response id
- 工具续轮只发送增量 tool result
- 减少请求体体积

## 十六、文件级实施计划

### `src/config_types.zig`

- 增加 provider 的 API mode 配置字段
- 保持默认值向后兼容

### `src/config.zig`

- 解析并校验 `api_mode`
- 非法值尽早报错

### `src/providers/compatible.zig`

- 增加 `CompatibleApiMode`
- 在 `OpenAiCompatibleProvider` 增加 `api_mode`
- 增加基于 `ChatRequest` 的 Responses request builder
- 增加 Responses response parser
- 在 `chatImpl` 中根据 `api_mode` 分支
- 可选增强 fallback 逻辑

### `src/providers/runtime_bundle.zig`

- 确保 provider holder 构造时把配置中的 API mode 传入 compatible provider 实例

### `src/providers/factory.zig` 或相关构造路径

- 检查所有 compatible provider 的构造入口
- 确保不会遗漏 `api_mode` 透传

### 测试

至少补以下测试：

1. 配置解析：`api_mode = responses`
2. 请求序列化：
   - system + user
   - 多轮 user/assistant
   - tools
   - tool result continuation
   - 图片输入
3. 响应解析：
   - 纯文本
   - 纯 function_call
   - 文本 + function_call 混合
4. transport 选择：`api_mode` 是否生效
5. legacy fallback 是否仍可用于旧配置

## 十七、建议新增的内部 helper

以下命名仅供参考：

### 请求侧

- `build_responses_chat_request_body(...)`
- `serialize_responses_input_items_into(...)`
- `serialize_responses_message_item(...)`
- `serialize_responses_function_call_item(...)`
- `serialize_responses_function_call_output_item(...)`
- `serialize_responses_content_part(...)`
- `collect_responses_instructions(...)`

### 响应侧

- `parse_responses_native_response(...)`
- `extract_responses_output_text(...)`
- `extract_responses_tool_calls(...)`
- `extract_responses_usage(...)`

## 十八、兼容性与风险

### 18.1 向后兼容性

如果实现正确，本方案对现有 provider 是安全的，因为：

- 默认模式仍是 `chat_completions`
- 只有显式配置 `responses` 的 provider 才会切新路径
- vtable 接口不变

### 18.2 主要风险点

1. 内部 tool history 映射到 Responses items 的顺序不正确。
2. Responses 的“文本 + function_call 混合输出”与当前 `ChatResponse` 结构不完全对齐。
3. 图片 content part 的序列化与现有 vision 数据结构可能存在边界不一致。
4. 配置透传链路可能漏掉某个 provider 构造点，导致 `api_mode` 被静默忽略。

### 18.3 风险缓解

1. 使用基于真实消息形态的 fixture 测试。
2. 尽量复用现有 native tool provider 的响应结构。
3. 针对图片请求体做单独序列化测试。
4. 同时验证主 agent 与 named subagent 两条链路。

## 十九、验证计划

### 19.1 单元测试

必须覆盖：

1. 配置解析
2. Responses 请求序列化
3. Responses 响应解析
4. `api_mode` 路由行为

### 19.2 集成测试

建议覆盖：

1. 纯文本返回
2. function_call 返回
3. 图片输入请求体形状

### 19.3 人工验证

开发完成后建议按以下步骤验证：

1. 配置一个专门的 `api_mode = responses` provider
2. 运行一条纯文本问答
3. 运行一条必须调用工具的 agent 指令
4. 如果中转支持 vision，再运行一条最小图片输入
5. 确认原有 `/chat/completions` provider 不受影响

## 二十、推荐交付阶段

### Phase 1

- 显式 provider API mode
- 非流式 Responses 主路径
- tools
- tool results
- 图片输入
- 测试

### Phase 2

- SSE streaming
- 增量事件解析

### Phase 3

- `previous_response_id`
- 工具续轮增量优化
- 如有必要再评估 WebSocket transport

## 二十一、具体映射示例

### 21.1 内部历史

- system: `You are helpful.`
- user: `Read package.json`
- assistant tool call: `read({"path":"package.json"})`
- tool result: `{...file text...}`

### 21.2 期望的 Responses 请求

```json
{
  "model": "gpt-5.4",
  "instructions": "You are helpful.",
  "input": [
    {"type": "message", "role": "user", "content": "Read package.json"},
    {
      "type": "function_call",
      "call_id": "call_123",
      "name": "read",
      "arguments": "{\"path\":\"package.json\"}"
    },
    {
      "type": "function_call_output",
      "call_id": "call_123",
      "output": "{...file text...}"
    }
  ],
  "tools": [...],
  "tool_choice": "auto",
  "stream": false
}
```

### 21.3 期望的 tool-call 响应

```json
{
  "id": "resp_abc",
  "status": "incomplete",
  "output": [
    {
      "type": "function_call",
      "id": "fc_1",
      "call_id": "call_456",
      "name": "bash",
      "arguments": "{\"command\":\"ls\"}"
    }
  ]
}
```

NullClaw 必须把这类响应解析成“发起工具调用”，而不是错误地当成空文本响应。

## 二十二、最终建议

建议把这项能力实现为：

- 现有 compatible provider 的扩展
- 而不是一套新 provider 家族

这样能在以下方面取得最佳平衡：

- 对现有架构侵入最小
- 对现有配置系统兼容最好
- 后续支持更多 Responses-only gateway 更容易
- 二进制体积和维护成本更可控

最低可上线范围应至少覆盖 Phase 1。若只做一个文本 fallback，仍然无法支撑真实 agent 使用场景。
