# opencombine

A self-hosted OpenAI-compatible gateway that combines **New API**, **Resin**, and **OpenCode Zen**.

This repository documents a working deployment pattern, the request-format changes required after OpenCode Zen tightened free-tier client validation in September 2026, and the troubleshooting process used to distinguish gateway, proxy, streaming, authentication, and upstream-model failures.

> This repository intentionally contains **no real API keys, session IDs, proxy subscriptions, server IPs, domains, access tokens, or other private credentials**. Replace every placeholder with your own value and keep secrets out of Git.

## Architecture

```text
Client
  |
  | OpenAI-compatible request
  v
New API
  |
  | channel routing / request override
  v
Resin
  |
  | optional proxy-pool egress
  v
OpenCode Zen
  |
  v
Upstream model
```

Typical public deployment:

```text
Client
  -> HTTPS reverse proxy
  -> New API :3000
  -> Resin :2260
  -> https://opencode.ai/zen
```

## What changed

A configuration that previously worked with ordinary OpenAI-compatible requests began returning:

```text
403 FreeTierError
OpenCode's free tier can only be used from within OpenCode
```

The infrastructure itself was healthy:

- New API was running.
- Resin was reachable from the New API container.
- Proxy egress was working.
- The OpenCode Zen endpoint was reachable.
- The same model could still work from the official OpenCode client.

The issue was request validation at the upstream layer.

## Verified request requirements

Testing isolated the request fields one by one.

For the tested free chat-completions models, the successful request shape required all of the following:

1. An OpenCode-style `User-Agent`, for example:

```http
User-Agent: opencode/1.18.31
```

2. A **valid session value from your own OpenCode client**:

```http
x-opencode-session: <YOUR_VALID_OPENCODE_SESSION>
```

A random value with the same `ses_...` shape was rejected.

3. Streaming enabled by the client:

```json
"stream": true
```

4. A `tools` array containing both function names:

```text
bash
read
```

The tool descriptions were not important in testing.

The following were tested and were **not required** for the successful request:

- `x-opencode-request`
- `x-opencode-client`
- `x-opencode-project`
- the longer `ai-sdk/provider-utils ... runtime/bun ...` part of the User-Agent
- `tool_choice: "auto"`
- `stream_options.include_usage`
- specific tool descriptions

## Minimal direct upstream test

Use only credentials/session data belonging to you. Never commit the real session value.

Create a request body:

```bash
cat > /tmp/opencode-test.json <<'EOF'
{"model":"mimo-v2.5-free","stream":true,"messages":[{"role":"user","content":"Reply with OK only"}],"tools":[{"type":"function","function":{"name":"bash","description":"x","parameters":{"type":"object","properties":{}}}},{"type":"function","function":{"name":"read","description":"x","parameters":{"type":"object","properties":{}}}}]}
EOF
```

Then test the upstream directly:

```bash
curl -N -s https://opencode.ai/zen/v1/chat/completions -H "Content-Type: application/json" -H "User-Agent: opencode/1.18.31" -H "x-opencode-session: <YOUR_VALID_OPENCODE_SESSION>" --data-binary @/tmp/opencode-test.json
```

A healthy response is SSE and ends with:

```text
data: ... "content":"OK" ...
data: [DONE]
```

## New API

Example container deployment:

```bash
docker network create ai
docker run --name new-api -d --restart always --network ai -p 3000:3000 -v /data/new-api:/data calciumion/new-api:latest
```

Create an OpenAI-type channel for OpenCode Zen.

### Base URL

When using Resin:

```text
http://resin:2260/Default/%2E/https/opencode.ai/zen
```

### Request-header override

Keep real secret values only in New API's private configuration.

Example:

```json
{
  "User-Agent": "opencode/1.18.31 ai-sdk/provider-utils/4.0.46 runtime/bun/1.3.14",
  "x-opencode-client": "cli",
  "x-opencode-project": "global",
  "x-opencode-session": "<YOUR_VALID_OPENCODE_SESSION>",
  "x-opencode-request": "msg_placeholder"
}
```

Only `User-Agent` and a valid `x-opencode-session` were proven necessary in the isolated tests above. The additional headers are retained here as a production-style example.

### Parameter override

New API's UI notes that `stream` cannot reliably be forced by parameter override. The client should therefore send `"stream": true` itself.

The important request-body addition is the `bash` + `read` tool pair.

A tested override structure is:

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

## Important: enable stream mode in New API's channel tester

A confusing failure looked like this:

```text
invalid character 'd' looking for beginning of value
```

The reason was:

1. The channel tester was in non-stream mode.
2. The upstream returned an SSE stream beginning with `data:`.
3. New API tried to parse the SSE response as ordinary JSON.
4. The first character it encountered was `d`, producing the parse error.

Enable **stream mode** in the New API test dialog before testing these models.

If a real client request works and returns SSE while the non-stream channel tester shows the error above, the upstream channel may already be healthy.

## Real New API test

The real client request must explicitly use streaming:

```bash
curl -N -s https://api.example.com/v1/chat/completions -H "Authorization: Bearer <YOUR_NEW_API_KEY>" -H "Content-Type: application/json" -d '{"model":"mimo-v2.5-free","stream":true,"messages":[{"role":"user","content":"Reply with OK only"}]}'
```

Expected ending:

```text
data: ... "content":"OK" ...
data: [DONE]
```

## Resin

Example Docker Compose configuration:

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

Keep the management port bound to localhost unless you intentionally secure and expose it.

New API and Resin must share the same Docker network so New API can reach:

```text
http://resin:2260
```

## Troubleshooting

### 403 FreeTierError

```text
OpenCode's free tier can only be used from within OpenCode
```

Check, in order:

- client sends `stream: true`
- upstream request contains an OpenCode-style User-Agent
- `x-opencode-session` is a valid session from your own OpenCode client
- request contains both `bash` and `read` tools
- the request is actually routed through the channel you edited

One subtle routing problem found during testing was having two New API channels for the same model. Configuration changes were made to one channel, while real traffic was routed to the other. Disable the stale channel or synchronize both configurations before retesting.

### 401 Missing API key

```text
AuthError: Missing API key.
```

This generally means the selected upstream model requires authenticated/paid access and is not available through the anonymous/free path being used by this setup.

Do not treat every model returned by a model-list endpoint as free.

### 400 Model is unavailable

The model ID may still exist in an old configuration while the upstream has removed or disabled it.

Refresh the current model list and remove stale IDs.

### `invalid character 'd'`

Enable stream mode in the New API channel-test dialog.

### New API says `IsStream: false`

The built-in channel tester can be non-streaming even when a real client uses streaming. Inspect real relay traffic separately and do not use a non-stream test result as the only health signal for an SSE-only upstream path.

## Distinguishing the three failure layers

```text
403 FreeTierError
  -> upstream client-validation problem

401 Missing API key
  -> model/authentication requirement

429 FreeUsageLimitError
  -> quota or rate-limit layer
```

These should be debugged separately.

Changing proxy IPs does not solve a 403 client-validation failure.

## Proxy pools and multiple egress IPs

Resin can provide multiple proxy egress routes for availability, network fault isolation, and handling IP-scoped rate limits where permitted by the upstream service.

This repository does not provide instructions for bypassing provider quotas or usage restrictions. Follow the upstream provider's current Terms of Service and rate-limit policies.

## Security checklist

Never commit any of the following:

```text
New API token
OpenCode session ID
OpenCode/API credentials
Resin admin token
proxy subscription URL
proxy node credentials
server public IP if you consider it private
real domain if you do not want it public
TLS private keys
Cloudflare credentials
Azure credentials
database backups
New API data directory
HAR files / copied cURL captures containing secrets
```

Recommended patterns:

```text
<YOUR_NEW_API_KEY>
<YOUR_VALID_OPENCODE_SESSION>
<YOUR_RANDOM_ADMIN_TOKEN>
https://api.example.com
<YOUR_SERVER_IP>
```

If a secret is ever pasted into a public issue, commit, log, HAR file, or chat transcript, rotate it rather than relying on redaction after the fact.

## Notes on model availability

Free/temporary model availability changes over time.

Use live upstream model discovery plus a real streaming request to verify a model before publishing it through your gateway.

A practical model-health check should distinguish:

- request succeeds
- 401: authentication required
- 403: client validation failed
- 400: model unavailable
- 429: rate/quota limit

## Status

Verified working flow:

```text
OpenAI-compatible streaming client
  -> New API
  -> Resin
  -> OpenCode Zen
  -> free chat-completions model
  -> SSE response
```

Tested successfully with a placeholder-safe equivalent of:

```text
model: mimo-v2.5-free
stream: true
tools: bash + read
valid OpenCode session
OpenCode-style User-Agent
```

## Disclaimer

This is an independent community deployment note and is not affiliated with OpenCode, New API, or Resin.

Upstream behavior can change without notice. Re-test request requirements after provider updates and use the service in accordance with its current terms and policies.
