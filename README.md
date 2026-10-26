# opencode-codearts-auth

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![magpie plugin](https://img.shields.io/badge/magpie-plugin-8A2BE2)](https://usemagpie.ai/docs/zh/plugins)
[![OpenCode plugin](https://img.shields.io/badge/OpenCode-plugin-blue)](https://opencode.ai)
![no dependencies](https://img.shields.io/badge/dependencies-none-brightgreen)
![tests](https://img.shields.io/badge/tests-56%20passing-success)

[Huawei Cloud CodeArts (华为云码道 代码智能体)](https://www.huaweicloud.com/product/codearts.html)  
作为 magpie / OpenCode 的 provider 插件：用**华为云 OAuth 授权登录**接入码道的模型  
（盘古 openPangu、DeepSeek、GLM 等）。

CodeArts 的模型网关确实是 OpenAI 兼容的 Chat Completions 接口，但它**不接受 API  
Key**：请求要用临时的 AK/SK/security-token 凭据做 *SDK-HMAC-SHA256* 签名，而这些  
凭据要靠 OAuth 登录取得。本插件完整实现了这条链路：

1.  **OAuth（PKCE + DPoP）登录**：`authorize()` 在本机 `127.0.0.1`（端口 ≥ 10000）  
    起一个回调服务器，返回华为云授权页地址给 magpie 打开；浏览器带授权码跳回后，  
    用 `code_verifier` 向 STS 换取临时凭据。登录全程是 PKCE 的，并用一把新生成的  
    ES256（P-256）DPoP 私钥给每次 token 请求签名。
2.  **SDK-HMAC-SHA256 签名**：每次模型请求都由 `loader` 的 `fetch` 用临时凭据签名，  
    和真实 IDE / CLI 的请求一致。
3.  **静默续期**：`auth.refresh` 在凭据过期前用 `refresh_token`（复用同一把  
    `code_verifier` 与 DPoP 私钥）换新凭据。
4.  **模型列表 / 用量**：登录后用签名 GET 拉取账号真实模型列表与积分套餐用量；模型 id、
    显示名与收费倍率都来自接口，插件不硬编码模型名，也不再标注免费模型。

## 安装

```sh
magpie plugin add yuweihao17/codearts

```

或本地开发：

```sh
magpie plugin add ./index.mjs

```

## 登录

选择 **Sign in with Huawei Cloud**，浏览器会打开华为云授权页；登录并在授权页确认  
后，浏览器会自动跳回本机的回调地址，插件随即完成登录。无需填写 API Key、也无需  
手动复制任何 token。

## 使用

-   登录后模型列表以账号真实返回的列表为准，共 8 个，全部按积分计费：
    `GLM-5.2`、`GLM-5.2-ArkTS-SPARK`、`OpenPangu-2.0-Pro`、`OpenPangu-2.0-Flash`、
    `GLM-5.3`、`DeepSeek-V4.1-Flash`、`Kimi-K3`、`DeepSeek-V4.1-Flash-1017`；
    未登录时用同样这 8 个的出厂目录。模型 id、显示名与收费倍率全部取自
    `v1/model/builtin`，插件不硬编码任何模型名或免费标记。
-   CLI 里可用 `codearts --model codearts/GLM-5.2` 之类方式指定。

## 说明与已知限制

-   **只有流式请求会被计量**：网关对非流式（`stream: false`）的 200 回答返回 token
    却不记账，积分不动。插件在签名前把对话请求改写为 `stream: true`，
    收到 SSE 后再折叠回调用方要求的 JSON 形态，宿主无感知。
-   **模型目录**是出厂默认（上下文窗口取自码道模型卡：`GLM-5.2` = 202752、
    `deepseek-v4.1-flash` 与其带日期形态 `deepseek-v4.1-flash-1017` = 1000000；
    接口不公布窗口的模型交给后端默认值）。账号能返回实时列表时会覆盖 `id`/`name`。
-   **免费额度体系已于 2026-10-26 下线**：`deepseek-v4.1-flash` 等模型改为积分计费，
    原 `maas_type: benefit` 头一律被拒（`InferHub.4004.200 benefit not found`），
    原 `opengw gateway/config`（免费模型集）与 `user/tokens/balance`（每日免费额度）
    两个端点返回 `4000 invalid channel, group not found`。插件已移除 benefit 头、
    免费标记与免费额度窗口，只按积分口径读写用量。
-   **带日期与不带日期是两个模型**：后端同时注册 `deepseek-v4.1-flash` 与
    `deepseek-v4.1-flash-1017`，倍率也不同；插件按原始 id 去重、按接口给的
    `model_name` 展示，不做去后缀合并。
-   **路由只看总积分**：magpie 的额度上限判断取自用量窗口，插件只让「总积分」窗口
    参与（其余窗口标 `aside: true`，含义是「用完也不影响账号使用」）。
-   **用量**是尽力而为：读取 `snap-manager/v1/statistics/plugin` 的积分口径，显示为
    「总积分」与「各积分包」窗口；取不到时在用量卡片显示一条错误，不影响对话。
-   **并发与排队**：码道按账号限制 3 个并发会话，超出的聊天请求返回 HTTP 400
    `TM.00001041`（也可能以 HTTP 200 + SSE 内嵌 `error_code` 返回）。插件按账号把
    在途请求限制为 3 个（超出的排队等待），对该错误每 10 秒重试一次（上限 180 次，
    约 30 分钟），重试前查询 `api/v1/queue/status`，返回 `queue_full` / `error` 时
    立即结束重试；调用方取消请求（`AbortSignal`）时立刻停止等待。同一账号复用固定的 `Session-Id` / `Chat-Id`，
    避免每个请求新开一个 60–90 秒才释放的上游会话。可用插件选项 `maxInflight`、`queueRetryDelayMs`、
    `queueMaxAttempts` 调整。
-   **故障切换**：magpie 只对非 2xx 响应切换账号，200 里内嵌的错误会被当作答案透传。插件因此把
    上游的三类软失败换成对应的状态码：额度用尽（SSE 内嵌 `InferHub.4291.x`）回答 429（magpie 会把请求
    转到下一个账号）；其他终态 SSE 错误回答 502（服务端错误，同样切换）；登录凭据过期
    （HTTP 400 + `APIG.0602` / `security token has expired`）先就地刷新凭据并重发一次，刷新令牌已被用掉
    （`STS5.1806`）时回答 401 并带 `X-Magpie-Sign-In: expired`（magpie 标记该账号要求重新登录，请求转下一个
    账号）。刷新令牌是一次性的：刷新后必须保存新令牌，任何不落盘的诊断调用都会把令牌烧掉。
-   **模型过滤**：视觉模型（id 以 `-vl` 结尾，大小写不敏感）不进入实时模型列表。
-   **区域 / 端点**：当前固定华为云 `cn-north-4`（与真实插件一致）。
-   本插件不内置 MCP / 中间件，仅做模型接入。
-   依赖：无（签名、PKCE、DPoP 都用运行时内置的 WebCrypto，无需 `jose` 等三方库）。

## English

**opencode-codearts-auth** is a magpie / OpenCode provider plugin that signs you in to
[Huawei Cloud CodeArts (码道)](https://www.huaweicloud.com/product/codearts.html) with
**Huawei Cloud OAuth** (PKCE + DPoP) and exposes its models (openPangu, DeepSeek, GLM, …).

The CodeArts gateway is OpenAI-compatible Chat Completions, but it does **not** accept an API key:
every request must be signed with temporary AK/SK/security-token credentials (*SDK-HMAC-SHA256*),
and those credentials come from an OAuth sign-in. This plugin implements the whole chain —
local-callback PKCE OAuth, per-request SDK-HMAC-SHA256 signing, silent refresh — with **no runtime dependencies**.

### Install

```sh
magpie plugin add yuweihao17/codearts
```

### Sign in

Pick **Sign in with Huawei Cloud** (or run `magpie plugin login codearts`). Your browser opens the
Huawei Cloud consent page; after you approve, it redirects back to a local `127.0.0.1` callback and
the plugin finishes signing in. No API key, no copy-paste.

### Use

Models appear as `codearts/GLM-5.2`, `codearts/deepseek-v4.1-flash`,
`codearts/deepseek-v4.1-flash-1017`, `codearts/kimi-k3`, … Verify with
`magpie provider test codearts`, or pin one with `codearts --model codearts/GLM-5.2`.
Signed in, the account's own list wins: eight credit-billed models, ids, names and
prices all read from `v1/model/builtin`.

### Notes

- Region/endpoint is fixed to Huawei Cloud `cn-north-4`.
- The free-quota (`benefit`) tier was withdrawn on 2026-10-26: its `gateway/config`
  and `user/tokens/balance` feeds answer `4000 invalid channel`, and the old
  `maas_type: benefit` header is rejected with `InferHub.4004.200 benefit not found`.
  The plugin no longer sends it, sets no `free: true` and shows no free-quota window.
- Dated and bare ids are two models: the backend registers both
  `deepseek-v4.1-flash` and `deepseek-v4.1-flash-1017` (at different rates), so the
  plugin dedupes by raw id and shows each under the published `model_name`.
- Routing reads only the package-total credit window: every other window is `aside: true`
  ("using it up doesn't stop the account").
- At most three concurrent sessions per account: extra chat requests queue behind a
  per-account gate and are retried every 10s (up to 180 times) while upstream answers
  `TM.00001041` (as a 400 or embedded in a 200 SSE stream), with the queue status endpoint
  consulted before each retry. Vision models (`-vl`) are filtered from the live list.
- Failover: magpie only moves a request to another account on a non-2xx answer, so soft
  failures inside a 200 SSE stream are re-answered with the status code that matches:
  exhausted quota (`InferHub.4291.x`) becomes 429, any other terminal SSE error becomes
  502, and an expired credential (400 `APIG.0602`) triggers an in-place credential renewal
  and one resend — a spent refresh token (`STS5.1806`) answers 401 with
  `X-Magpie-Sign-In: expired` so magpie marks the account and moves on. Refresh tokens are
  single-use: a renewal must persist the new token or the account is burned.
- Only streaming requests are metered: the plugin rewrites chat bodies to
  `stream: true` before signing and folds the SSE stream back into a JSON
  completion for callers that asked for non-streaming.
- Usage is best-effort, read from the credits endpoint only.
- No MCP / gateway middleware — model access only.

## License

MIT — see [LICENSE](./LICENSE).