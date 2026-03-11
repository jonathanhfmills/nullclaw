# NullClaw Responses Provider 使用说明

## 适用场景

当你接入的大模型中转服务：

- 不支持 `/chat/completions`
- 只支持 `/responses`

就需要在 NullClaw 的 provider 配置里显式开启：

- `api_mode = "responses"`

这类场景下，问题不是模型名本身，而是协议不同。

## 最小配置示例

下面是最小可用配置示例：

```json
{
  "models": {
    "providers": {
      "sub2api": {
        "base_url": "https://your-gateway.example.com/v1",
        "api_key": "sk-your-key",
        "api_mode": "responses"
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "sub2api/gpt-5.4"
      }
    }
  }
}
```

## 字段说明

- `base_url`
  - 你的中转服务 OpenAI 风格入口
  - 通常形如 `https://xxx/v1`
- `api_key`
  - 中转服务提供的密钥
- `api_mode`
  - 必须写成 `"responses"`
  - 如果不写，默认是 `chat_completions`
- `agents.defaults.model.primary`
  - 格式是 `provider_name/model_name`
  - 例如：`sub2api/gpt-5.4`

## 推荐配置方式

建议把只支持 `/responses` 的中转单独定义成一个新 provider，不要直接覆盖已有的 `openai`、`novacode` 之类 provider。

推荐：

```json
{
  "models": {
    "providers": {
      "openai": {
        "api_key": "sk-xxx"
      },
      "sub2api": {
        "base_url": "https://your-gateway.example.com/v1",
        "api_key": "sk-your-key",
        "api_mode": "responses"
      }
    }
  }
}
```

这样做的好处：

- 不会影响原有 provider
- 更容易切换和回滚
- 后续排查问题更清晰

## 多 agent / 子代理场景

如果你有命名 agent 或子代理单独指定 provider/model，也需要确保它们使用的是 Responses provider。

例如：

```json
{
  "agents": {
    "list": [
      {
        "name": "coder",
        "provider": "sub2api",
        "model": "gpt-5.4"
      }
    ]
  }
}
```

否则默认 agent 能跑，不代表命名 agent 一定走的是同一条 provider 链路。

## 如何验证配置是否生效

### 1. 健康检查

可先运行：

```bash
nullclaw --probe-provider-health --timeout-secs 20
```

如果配置正确，应该返回类似：

```json
{"provider":"sub2api","model":"gpt-5.4","live_ok":true,"status":"ok","reason":"ok","status_code":200}
```

### 2. 常见错误信号

如果仍看到这类报错：

- `Unsupported legacy protocol: /v1/chat/completions is not supported`
- `Please use /v1/responses`

通常说明有以下几种可能：

- 当前运行的不是你新编译的二进制
- 配置文件里对应 provider 没有加 `api_mode: "responses"`
- 实际使用的 provider 不是你以为的那个 provider
- 命名 agent / 子代理仍在走旧 provider

## 当前实现支持范围

当前这版 Responses 支持已经覆盖：

- 非流式文本对话
- 多轮历史
- tools
- tool result continuation
- 图片输入
- reasoning effort 基础透传

当前策略是：

- `chat` 主路径支持 Responses
- `chatWithSystem` 支持 Responses
- `stream_chat` 在 Responses 模式下安全降级为非流式

也就是说，如果 provider 配成 `responses`，功能是可用的，但流式目前走的是安全降级策略，不是完整 SSE Responses 事件流。

## 排障建议

优先按这个顺序排查：

1. 确认正在运行的新二进制
2. 确认 provider 配置里确实有 `"api_mode": "responses"`
3. 确认 `agents.defaults.model.primary` 指向的是正确 provider
4. 运行 `--probe-provider-health`
5. 再测试普通对话和工具调用

## 安全提醒

- 不要把真实 `api_key` 提交到 GitHub
- 如果你之前把 key 写进过仓库、日志或文档，建议后续做一次轮换
