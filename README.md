# opencombine

一个将 **New API**、**Resin** 和 **OpenCode Zen** 组合起来的自托管 OpenAI 兼容网关方案。

这个仓库记录了一套已经实际跑通的部署方式，以及在 2026 年 9 月 OpenCode Zen 加强免费层客户端校验后，如何定位并解决 403、流式响应解析、渠道路由、模型鉴权等问题。

>所有示例均使用占位符，请自行替换

## 整体架构

```text
客户端
  |
  | OpenAI 兼容请求
  v
New API
  |
  | 渠道路由 / 请求头与参数覆盖
  v
Resin
  |
  | 可选代理出口 / 多节点
  v
OpenCode Zen
  |
  v
上游模型
```

典型公网部署链路：

```text
客户端
  -> HTTPS 反向代理
  -> New API :3000
  -> Resin :2260
  -> https://opencode.ai/zen
```

## 问题背景

原本可以正常工作的 OpenAI 兼容请求，后来开始返回：

```text
403 FreeTierError
OpenCode's free tier can only be used from within OpenCode
```
检查发现由于opencode更新，不能正常第三方调用上游，官方说法仅在opencode客户端
排查后确认基础设施本身都正常：

- New API 正常运行
- New API 容器可以访问 Resin
- Resin 代理出口正常
- OpenCode Zen 可访问
- 同一个模型在 OpenCode 客户端里仍然可用

最终问题出在上游对请求来源和请求结构的校验。

## 已验证的关键请求条件

通过逐项删减和对比请求，确认了以下条件。

对于测试过的免费 `chat/completions` 模型，成功请求至少需要：

### 1. OpenCode 风格 User-Agent

例如：

```http
User-Agent: opencode/1.18.31
```

完整的：

```text
opencode/1.18.31 ai-sdk/provider-utils/4.0.46 runtime/bun/1.3.14
```

也可以。

测试中已经确认，后面的 `ai-sdk/provider-utils` 和 `runtime/bun` 并不是必要条件。

### 2. 有效的 x-opencode-session

例如：

```http
x-opencode-session: <YOUR_VALID_OPENCODE_SESSION>
```

这里必须是你自己的 OpenCode 客户端产生的有效 Session。

测试结果：

```text
真实有效 session -> 成功
随机伪造 ses_xxx -> 403 FreeTierError
```

所以仅仅伪造一个看起来像 `ses_...` 的值是不够的。

### 3. 客户端必须使用流式请求

请求体中必须有：

```json
"stream": true
```

### 4. tools 中同时存在 bash 和 read

请求体中需要同时包含：

```text
bash
read
```

两个 function tool。

测试结果：

```text
只有 bash          -> 失败
只有 read          -> 失败
bash + dummy       -> 失败
dummy + read       -> 失败
bash + read        -> 成功
```

工具的 `description` 文本不重要，改成简单的 `"x"` 也可以正常使用。

## 已确认不是必要条件的字段

以下字段经过单独测试，确认不是成功请求的必要条件：

- `x-opencode-request`
- `x-opencode-client`
- `x-opencode-project`
- User-Agent 后面的 `ai-sdk/provider-utils ... runtime/bun ...`
- `tool_choice: "auto"`
- `stream_options.include_usage`
- tools 的具体 description 内容

因此当前最核心的组合可以概括为：

```text
OpenCode 风格 User-Agent
+
有效 x-opencode-session
+
stream: true
+
tools 中同时存在 bash 和 read
```

## 直接测试 OpenCode Zen

使用你自己的合法 Session，

先创建请求体：

```bash
cat > /tmp/opencode-test.json <<'EOF'
{"model":"mimo-v2.5-free","stream":true,"messages":[{"role":"user","content":"只回复OK"}],"tools":[{"type":"function","function":{"name":"bash","description":"x","parameters":{"type":"object","properties":{}}}},{"type":"function","function":{"name":"read","description":"x","parameters":{"type":"object","properties":{}}}}]}
EOF
```

然后直接请求 OpenCode Zen：

```bash
curl -N -s https://opencode.ai/zen/v1/chat/completions -H "Content-Type: application/json" -H "User-Agent: opencode/1.18.31" -H "x-opencode-session: <YOUR_VALID_OPENCODE_SESSION>" --data-binary @/tmp/opencode-test.json
```

正常情况下会得到 SSE 流式响应，结尾类似：

```text
data: ... "content":"OK" ...
data: [DONE]
```

## New API 部署

示例 Docker 部署：

```bash
docker network create ai
docker run --name new-api -d --restart always --network ai -p 3000:3000 -v /data/new-api:/data calciumion/new-api:latest
```

然后在 New API 中创建一个 OpenAI 类型渠道。

## New API 渠道配置

### Base URL

如果通过 Resin 转发：

```text
http://resin:2260/Default/%2E/https/opencode.ai/zen
```

### 请求头覆盖


示例：

```json
{
  "User-Agent": "opencode/1.18.31 ai-sdk/provider-utils/4.0.46 runtime/bun/1.3.14",
  "x-opencode-client": "cli",
  "x-opencode-project": "global",
  "x-opencode-session": "<YOUR_VALID_OPENCODE_SESSION>",
  "x-opencode-request": "msg_placeholder"
}
```

根据逐项测试结果，真正确认必要的是：

```text
User-Agent
x-opencode-session
```

其余几个请求头可以保留，但不是必须项。

### 参数覆盖

New API 界面会提示：

```text
无法覆盖 stream 参数
```

因此不要依赖参数覆盖强制设置 `stream:true`。

真正的客户端请求本身应该主动发送：

```json
"stream": true
```

参数覆盖里最重要的是给请求补上 `bash` 和 `read` 两个 tools。

示例：

```json
{
  "operations": [
    {
      "path": "stream_options",
      "mode": "set",
      "value": {
        "include_usage": true
      }
    },
    {
      "path": "tool_choice",
      "mode": "set",
      "value": "auto"
    },
    {
      "path": "tools",
      "mode": "set",
      "value": [
        {
          "type": "function",
          "function": {
            "name": "bash",
            "description": "execute shell command",
            "parameters": {
              "type": "object",
              "properties": {}
            }
          }
        },
        {
          "type": "function",
          "function": {
            "name": "read",
            "description": "read file",
            "parameters": {
              "type": "object",
              "properties": {}
            }
          }
        }
      ]
    }
  ]
}
```

其中 `stream_options` 和 `tool_choice` 在测试中已经确认不是必要条件，保留只是为了贴近完整请求结构。

## New API 后台测试为什么会报 invalid character 'd'

一个非常容易误导的报错：

```text
invalid character 'd' looking for beginning of value
```

实际原因是：

1. New API 的渠道测试器处于非流式模式
2. 参数覆盖或上游请求实际变成了流式
3. OpenCode Zen 正常返回 SSE：
   ```text
   data: {...}
   ```
4. New API 测试器却把它当普通 JSON 解析
5. 第一个字符就是 `d`
6. 因此出现：
   ```text
   invalid character 'd' looking for beginning of value
   ```

解决方法：

**在 New API 的渠道测试窗口右上角开启“流式模式”。**

开启之后再测试，就能正确处理 SSE 响应。

因此：

```text
invalid character 'd'
```

并不一定代表上游失败，反而可能说明上游已经正常返回了 `data:` 流。

## 真实 New API 调用测试

真实客户端必须明确使用流式请求。

例如：

```bash
curl -N -s https://api.example.com/v1/chat/completions -H "Authorization: Bearer <YOUR_NEW_API_KEY>" -H "Content-Type: application/json" -d '{"model":"mimo-v2.5-free","stream":true,"messages":[{"role":"user","content":"只回复OK"}]}'
```

成功时结尾类似：

```text
data: ... "content":"OK" ...
data: [DONE]
```

## 一个非常隐蔽的问题：改错渠道

如果 New API 中存在两个支持同一模型的渠道，例如：

```text
#1 zen-free
#2 zen-free_copy
```

你可能修改的是 #2，但真实 API 请求却被 New API 路由到了 #1。

这会导致：

```text
后台测试 #2 看起来已经接近成功
真实 API 仍然 403
```

排查时查看 New API 日志：

```text
channel error (channel #1, status code: 403)
```

如果日志显示真实请求走了旧渠道，应：

- 临时禁用旧渠道
- 或把两个渠道配置同步
- 再重新测试

真实请求成功后，可以确认整个链路已经打通。

## Resin

示例 Docker Compose：

```yaml
services:
  resin:
    image: ghcr.io/resinat/resin:latest
    container_name: resin
    restart: unless-stopped
    environment:
      RESIN_ADMIN_TOKEN: "<YOUR_RANDOM_ADMIN_TOKEN>"
      RESIN_PROXY_TOKEN: ""
    ports:
      - "127.0.0.1:2260:2260"
    volumes:
      - ./cache:/var/cache/resin
      - ./state:/var/lib/resin
      - ./log:/var/log/resin
    networks:
      - ai

networks:
  ai:
    external: true
```

建议只把 Resin 管理端口绑定到：

```text
127.0.0.1:2260
```

New API 和 Resin 需要处于同一个 Docker 网络，例如：

```text
ai
```

这样 New API 才能访问：

```text
http://resin:2260
```

## 常见报错与含义

### 403 FreeTierError

```text
OpenCode's free tier can only be used from within OpenCode
```

优先检查：

1. 客户端是否发送 `stream:true`
2. 上游是否有 OpenCode 风格 User-Agent
3. `x-opencode-session` 是否真实有效
4. 请求体中是否同时存在 `bash` 和 `read`
5. 真实请求是否真的走了你修改过的 New API 渠道

这个错误属于客户端身份/请求结构校验层。

### 401 Missing API key

```text
AuthError: Missing API key.
```

通常意味着：

当前模型需要上游 API Key 或付费鉴权，不属于当前匿名/免费调用路径。

因此不要看到模型存在于 `/v1/models` 就默认它一定免费可用。

### 400 Model is unavailable

例如：

```text
Model is unavailable
```

说明：

- 模型可能已经下线
- 模型 ID 已过期
- 本地 New API 模型列表没有及时更新

建议重新获取实时模型列表并清理旧模型。

### 429 FreeUsageLimitError

通常属于：

```text
免费额度 / 限流层
```

它和 403 是两类完全不同的问题。

## 三种错误要分开处理

```text
403 FreeTierError
  -> 客户端身份 / 请求结构校验

401 Missing API key
  -> 模型需要鉴权或付费 Key

429 FreeUsageLimitError
  -> 免费额度 / 速率限制
```

更换代理 IP 并不能解决 403。

## Resin 多出口与代理池

Resin 可以用于：

- 多代理出口
- 网络故障切换
- 出口 IP 健康检查
- 在上游允许的范围内处理 IP 级限流
- 提高链路可用性



## 如何查看当前模型列表

可以直接请求：

```bash
curl -s https://opencode.ai/zen/v1/models
```

如果只想简单筛选名称中包含 `-free` 的模型，可以：

```bash
curl -s https://opencode.ai/zen/v1/models | python3 -c 'import sys,json; d=json.load(sys.stdin)["data"]; print("\n".join(x["id"] for x in d if x["id"].endswith("-free") or x["id"]=="big-pickle"))'
```

注意：

模型名带 `-free` 只能作为候选判断。

真正是否可用，仍建议通过真实流式请求验证。

## 模型健康状态建议

建议区分以下状态：

```text
成功
401 -> 需要鉴权
403 -> 客户端校验失败
400 -> 模型不可用
429 -> 限流或额度耗尽
```

不要把所有失败都归为“模型坏了”。


## 当前已验证成功的链路

```text
OpenAI 兼容流式客户端
  -> New API
  -> Resin
  -> OpenCode Zen
  -> 免费 chat/completions 模型
  -> SSE 正常返回
```

已经实际验证过的请求结构：

```text
model: mimo-v2.5-free
stream: true
tools: bash + read
有效 OpenCode Session
OpenCode 风格 User-Agent
```

最终可以正常收到：

```text
data: ... "content":"OK" ...
data: [DONE]
```

## 说明

OpenCode Zen 的上游规则可能随时调整。

如果未来再次出现：

```text
403
401
429
Model unavailable
```

建议重新做最小化 A/B 测试，不要直接假设还是旧原因。

本文档记录的是已经实际验证过的一套排查过程和部署经验。

## 免责声明

本项目是独立的社区部署与故障排查记录，与 OpenCode、New API、Resin 官方均无隶属或背书关系。

请在遵守相关服务条款、访问规则和限流政策的前提下使用。
