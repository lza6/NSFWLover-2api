# NSFWLover.com 逆向分析报告

> 分析对象：`源代码、网络包/`（Next.js 前端产物 + HAR 抓包）
> 分析日期：2026-10-27
> 方法：HAR 解析 + 混淆 JS 静态检索（webpack chunk 反推）

---

## 一、技术栈判定

| 层面 | 结论 | 证据 |
|------|------|------|
| 前端框架 | **Next.js（App Router, RSC）** | `main-app-*.js`、`next-router-state-tree`、`text/x-component` RSC 流式响应、`webpackChunk_N_E` 全局变量 |
| UI 库 | React 18/19 + Tailwind（utility class 字符串） | `useState`、`className:"bg-gradient-to-r..."` |
| 校验 | **Zod**（`f.z.object/string/enum`） | 模型枚举、请求体 schema 全部用 zod 定义 |
| 图表/时间 | dayjs（module 71096） | `_invalidDate` 正则特征 |
| 部署 | **Cloudflare（WAF + Turnstile）→ AWS CloudFront（IAD89-P3）→ 源站** | `cf-ray`、`x-amz-cf-pop`、`x-middleware-rewrite` 说明 Next.js middleware 做多站点路由 |
| 多站点架构 | middleware 将外部域名 rewrite 到 `/sites/www.nsfwlover.com/...` 内部路径 | RSC 请求头 `next-url: /sites/www.nsfwlover.com/...`、segmentPath 含 `sites` 层级 |
| 图片 CDN | Bunny CDN（`https://loveimages.b-cdn.net`） | publicInfo 配置 |
| 本地存储 | IndexedDB 存历史图（`A.iy.add({metadata, originalBase64, watermarkedBase64})`），" creations stay yours — stored locally" | chunk 60182 / 页面文案 |
| 支付 | NOWPayments、Unlimit、Stripe、Squeezy | API 枚举 `/api/webhooks/nowpayments` 等 |

---

## 二、核心 API 接口逆向

### 2.1 文生图主链路（已验证，来自 HAR 实抓）

```
① POST /api/image/generation/{modelType}
   Headers:
     Content-Type: application/json
     CF-Turnstile-Token: <Turnstile 验证码 token>   ← 未登录免费生成的唯一门槛
   Body:
     {
       "imageMetadata": {
         "prompt": "...",
         "modelType": "zimage-turbo",
         "width": 768, "height": 1024,
         // img2img 时附加: "base64_img": "<dataURL>" 或 "base64_imgs": [...]
         // sfw: boolean
       },
       "userConfirmed": false
     }
   Response 200:
     同步模式（快模型）→ {"isUnlocked":false, "base64":"data:image/jpeg;base64,..."}
     异步模式（慢模型）→ {"isUnlocked":false,
                          "metadata":{"queryParams":{
                             "prompt_id":"<uuid>",
                             "signature":"<sha256-like hex>"}}}
```

```
② 轮询 GET /api/image/generation?prompt_id={uuid}
   Headers: X-Signature: <① 返回的 signature>
   Response:
     未完成 → {"image":""}
     完成   → {"image":"data:image/jpeg;base64,..."}

前端轮询策略（chunk 60182 实测还原）：
  共 38 次；第 1~2 次间隔 3s，之后 5s（最长约 184s 超时）
  404 = 任务不存在，继续轮询；!ok = 终止并报错
```

错误码语义（前端处理逻辑）：
- `409` → 触发刷新/重置流程（疑为重复任务或状态冲突）
- `401` → 弹 "Sign In Required"（该模型需登录）
- `429` → 读取 `Retry-After` 头，进入限流倒计时

### 2.2 已还原的全部图片模型枚举（zod schema）

```
zimage | zimage-turbo | zimage-turbo-nsfw | zimage-base | zimage-base-vip
banana | banana-pro | banana-2            ← Gemini 2.5 / Flash Image 系列
sd | flux2klein | flux2klein-9b | flux2klein-9b-nsfw | flux2klein-9b-sfw
flux-2-pro | flux2klein-faceswap | flux2klein-9b-faceswap
veo-video-default | veo-video-1080p       ← 视频类
grok-video | grok-video-10s | grok-video-15s
ltx-2.3-10s | ltx-2.3-fast-10s | ltx-2.3-fast-20s
minimax-h3-10s | wan-v2.2-5b | wan-v2.2-a14b-turbo
```

生成端点路由规则（编辑页 chunk 68129）：
- `zimage`（i2i）→ 专用端点 `/api/image/generation/zimage/i2i`
- `flux2klein-9b*` / `flux-2-pro` → 统一投递到 `/api/image/generation/flux2klein`，真实模型靠 `modelType` 字段区分
- 其余 → `/api/image/generation/{modelType}`

### 2.3 模型目录与计费表（chunk 65489 `x.nz` 常量）

免费（badge FREE）：`zimage-turbo`（fdpo lora 加速）、`flux2klein`（4B 低分辨率，0 积分）

积分价目（image_editor）：
```
flux2klein_4b:0   flux2klein_9b:10  flux2pro:80   zimage_base:10
zimage_base_vip:20  banana:25  banana_2:50  banana_pro:100
zimage_turbo_nsfw:10  kissing:50
flux2klein_9b_face_swap:50  flux2klein_4b_face_swap:0
```
视频（次）：`grok_video:300, grok_video_10s:500, grok_video_15s:800, veo默认:250, veo1080p:400, ltx_2_3_10s:2000, wan_5b:1500 ...`
语音：`openai:5, gemini:5, elevenlabs:20, edge:1`

### 2.4 其他关键端点（完整枚举自 chunk 65489 module 42233）

```
POST /api/image/prompt/free        ← 免费提示词扩写 {userInput, sfw, modelType} → {positive}
POST /api/image/prompt             ← 付费扩写
POST /api/image/describe           ← 图生文
GET  /api/image/generation/character ← 角色立绘
POST /api/proxy/openai             ← OpenAI 代理（聊天走 chat/completions）
POST /api/openai/moderation        ← 内容审核
GET  /api/user/balance             ← 积分余额
POST /api/web/init                 ← 站点初始化配置
/api/webhooks/nowpayments | /api/webhooks/unlimit  ← 支付回调
/api/refer | /api/refer/redeem     ← 邀请返佣（commission 50%）
/api/livekit/connection            ← 实时语音 (LiveKit)
/api/characters/advanced[/list|/generate/advanced]  ← 角色系统
```

---

## 三、安全/风控机制还原

1. **Cloudflare Turnstile（两把 sitekey）**
   - 无痕验证（免费生成用）：`TURNSTILE_INVISIBLE_SITE_KEY = 0x4AAAAAACJLXCF4CTzGutkF`
   - 可见验证：`TURNSTILE_SITE_KEY = 0x4AAAAAAAxIUMmrsy557rov`
   - 前端以 `size:"invisible"` 显式渲染，动作名如 `"image-edit"`，token 放 `CF-Turnstile-Token` 头
   - 加载器有防重复注入（检测 `script[src*="challenges.cloudflare.com/turnstile"]`）

2. **X-Signature 任务签名**：POST 成功返回 `signature`，轮询必须携带。服务端签名算法未知（服务侧生成），前端仅透传——这是防"轮询他人任务"的绑定机制，非前端可逆点。

3. **分层准入**：匿名（`ANONYMOUS_ENABLED:true`）可玩免费模型；VIP 模型（如 `nsfw-34b` 聊天、`zimage-base-vip`）401 强制登录；credit 模型查 `/api/user/balance`。免费图片限额 `SUBSCRIPTION_FORBIDDEN_MIN_FREE_IMAGE_COUNT:10`，聊天条数 30。

4. **内容审核**：`userConfirmed:false` + `/api/openai/moderation`，敏感词/水印在前端完成（`watermarkedBase64` 本地合成，watermark=`/favicon-128x128.png` + "NSFWLover.com"）。

5. **风控指纹采集**：HAR 中可见 Cloudflare challenge（`cf-chl` PrivateToken PAT 流程），即浏览器环境指纹校验在 CF 边缘完成。

---

## 四、站点配置全量（publicInfo，SSR 注入）

```
FREE_MODEL: [lfm-7b, llama3-8b]   DEFAULT_MODEL: lfm-7b   DEFAULT_VIP_MODEL: nsfw-34b
EXCLUDE_MODEL: [microsoft-phi-4, gemini-2-flash]
GOOGLE_ANALYTICS: G-EW1ZV1NJMV / AW-481385607（含 signup/subscribe 转化事件）
ENABLE_SHARE_TO_EARN: false   SHARE_COMMISSION_RATE: 0.5
CDN_URL: https://loveimages.b-cdn.net
```

---

## 五、调用时序图（文字版）

```
浏览器                    Cloudflare            Next.js 源站
  │ --Turnstile widget--> challenges.cloudflare.com (拿 token)
  │ POST /api/image/generation/zimage-turbo  (CF-Turnstile-Token)
  │ --------------------------> WAF ----------> 校验 token + 限流
  │                             <----- {prompt_id, signature} 或同步 base64
  │ loop (3s×2 → 5s×36, X-Signature 头):
  │ GET /api/image/generation?prompt_id=...    → {"image":""} 直到出图
  │ <----- {image: data:image/jpeg;base64,...}
  │ 本地 IndexedDB 落库 + 前端合成水印
```

---

## 六、复刻要点（若要自建同型服务）

1. **架构**：Next.js App Router + middleware 多站点 rewrite + CF 前置；生成走异步任务队列（prompt_id + HMAC 签名轮询是成熟且廉价的做法）。
2. **免费通道风控**：Turnstile invisible + 匿名会话 + 429 Retry-After，三层即可撑住免费模型滥用。
3. **计费设计**：积分按"模型单价 × 步数/时长"动态计（`draw.steps`、`call.creditPerSecond:2`），SDXL 与 SD1.5 两套基价。
4. **前端证据链**：所有请求体先过 Zod schema（`imageMetadata` 校验 `prompt.max(ew)`、`width/height positive()`），服务端 schema 可从前端枚举完整还原。
5. **待验证项**：`signature` 生成算法、服务端限流阈值、Turnstile 校验失败响应格式（需服务端流量，前端产物不可见）。

---

## 附：文件归属速查

| 文件 | 内容 |
|------|------|
| `www.nsfwlover.com.har` | 完整生成链路抓包（Turnstile + POST + 2 次轮询） |
| `60182-…js` | 生成器主组件：txt2img 轮询逻辑、错误处理、IndexedDB |
| `65489-…js` | 模型枚举/计费表/API 端点总表/Zod schema（逆向价值最高） |
| `68129-…js` | 图片编辑（i2i）链路 |
| `88310-…js` | Turnstile hook（useTurnstile）实现 |
| `nsfw-ai-image-generator` | SSR HTML：publicInfo 全量配置、两把 sitekey |
| `page-*.js` | 各页面路由组件（generator/edit/history） |
| 其余 chunk | React/Next 运行时、moment、polyfill 等公共库 |
