## 项目代码来源：https://linux.do/t/topic/873771

### Credit：

- https://linux.do/t/topic/836702
- https://linux.do/t/topic/865204
- https://linux.do/t/topic/864744

#### 本人仅添加香港区域限制，并针对号池允许对 429 错误重试。在此一并感谢上述所有开发者的贡献。

### 一键部署：
<a href="https://vercel.com/new/clone?repository-url=https%3A%2F%2Fgithub.com%2Fsxjeru%2Fvercel-gemini-retry&env=UPSTREAM_URL_BASE,MAX_CONSECUTIVE_RETRIES,DEBUG_MODE,RETRY_DELAY_MS,SWALLOW_THOUGHTS_AFTER_RETRY&envDescription=%E7%95%99%E7%A9%BA%E5%8D%B3%E4%BD%BF%E7%94%A8%E9%BB%98%E8%AE%A4%E5%80%BC%E3%80%82%E5%A6%82%E9%9C%80%E8%87%AA%E5%AE%9A%E4%B9%89%EF%BC%8C%E8%AF%B7%E5%8F%82%E8%80%83%20README%20%E6%96%87%E6%A1%A3%EF%BC%9A&envLink=https%3A%2F%2Fgithub.com%2Fsxjeru%2Fvercel-gemini-retry%2Fblob%2Fmain%2FREADME.md&project-name=vercel-gemini-retry&repository-name=vercel-gemini-retry" target="_blank" rel="noopener noreferrer"><img src="https://vercel.com/button" alt="Deploy with Vercel" style="position: relative; top: 16px;"></a>

# Vercel Gemini try'n'retry

一个 Gemini 代理项目，支持流式（SSE）响应、内部重试机制以及对上游错误的标准化处理。

简单来说，就是 **截断后自动重试**。

## 限制
- [Vercel Function](https://vercel.com/docs/functions/runtimes/edge) 要求在 25 秒内发送初始响应，此后无执行时间限制，包括长时间流式输出。
- 末尾会固定输出 `[done]`

## 快速概览
- Edge Function 路由：`/api/proxy`
- 运行时：Vercel Edge（`runtime: 'edge'`）
- 关键文件：`api/proxy.js`

## 环境变量
您需要在部署或本地运行前设置下列环境变量（可在 Vercel 仪表盘或本地 shell 中设置）：

- `UPSTREAM_URL_BASE`：上游 API 基础 URL，默认 `https://generativelanguage.googleapis.com`
- `MAX_CONSECUTIVE_RETRIES`：最大连续重试次数（默认 `100`）
- `DEBUG_MODE`：开启调试日志（`true`/`false`，默认关闭）
- `RETRY_DELAY_MS`：重试间隔毫秒数（默认 `750`）
- `SWALLOW_THOUGHTS_AFTER_RETRY`：重试后是否吞掉模型的“思考（thought）”片段（`true`/`false`，默认为 `true`）
- `RETRY_ON_RATE_LIMIT`：是否对 429 (Rate Limit) 错误进行重试（`true`/`false`，默认为 `false`）

另外，客户端需要在请求头中传递 `Authorization`（或 `X-Goog-Api-Key`）用于上游认证。

## 本地开发与测试
先安装 Vercel CLI（若未安装）并登录：

```powershell
npm i -g vercel
vercel login
```

在项目根目录启动开发服务器：

```powershell
npm run dev
```

默认监听 3000，您可以用 curl 或浏览器访问：

```powershell
# 非流式 GET 示例
curl "http://localhost:3000/v1/models" -H "X-Goog-Api-Key: YOUR_KEY"

# 流式 POST 示例（SSE）
curl -N "http://localhost:3000/v1/models:streamGenerateContent?alt=sse" \
  -H "Content-Type: application/json" \
  -H "X-Goog-Api-Key: YOUR_KEY" \
  -d '{"messages":[{"role":"user","content":"hello"}]}'
```

> 注意：`vercel dev` 会自动加载 `.env` 文件中的环境变量，但为了在 PowerShell 中正确读取，请确保变量格式为 `NAME=value`（或者使用 `setx` 永久设置）。

## 手动部署到 Vercel
1. 在 Vercel 仪表盘新建项目并连接到您的仓库，或在本地使用：

```powershell
vercel --prod
```

2. 在 Vercel 项目设置中配置上面的环境变量（`UPSTREAM_URL_BASE`、`MAX_CONSECUTIVE_RETRIES` 等）。

3. `vercel.json` 已配置了所有路径都重写到 `api/proxy`，这样您可以直接将任意请求代理到上游服务。

## Edge Function 区域限制
项目已配置为不在香港地域运行，以避免区域报错。如需修改执行区域，请编辑 `config.regions` 配置：

```js
regions: = [
  'sin1',
  'icn1',
  'hnd1',
  'kix1',
  'bom1',
  'cpt1',
  'pdx1',
  'cle1',
  'syd1',
  'iad1',
  'sfo1',
  'gru1',
];
```

> 提示：选择离用户最近的区域可减少延迟。如果不设置 `regions`，Vercel 会在所有可用区域部署。（包括香港）

## 注意事项与调试
- 该代理会将上游返回的 SSE 流直接转发给客户端，并在检测到不完整或被阻断的响应时尝试内部重试。
- 如果遇到 4xx 或 429 类型的非重试性错误，代理会直接将其标准化并返回给客户端。
- 在调试阶段可设置 `DEBUG_MODE=true` 以便在 Vercel 日志中得到更详细信息。

