# NSFWLover-2api

对 [nsfwlover.com](https://www.nsfwlover.com)（NSFW AI 图像生成站）的**前端逆向分析**与免费通道 API 复刻工程。

## 仓库内容

| 目录/文件 | 说明 |
|-----------|------|
| `源代码、网络包/` | 抓取的前端产物：Next.js chunk（混淆 JS）+ `www.nsfwlover.com.har` 完整抓包 + SSR HTML |
| `分析报告-NSFWLover逆向.md` | 完整逆向报告：技术栈、API 全链路、模型枚举、计费表、Turnstile 机制 |
| `graft/` | 代码图谱缓存（本地生成，不入库） |

## 核心逆向结论（速览）

### 生成链路（已实抓验证）

```
① POST https://www.nsfwlover.com/api/image/generation/{modelType}
   Headers:
     Content-Type: application/json
     CF-Turnstile-Token: <Cloudflare Turnstile token>   ← 免费通道唯一门槛
   Body:
     {
       "imageMetadata": {
         "prompt": "...",
         "modelType": "zimage-turbo",
         "width": 768, "height": 1024
       },
       "userConfirmed": false
     }
   → 同步: {"isUnlocked":false,"base64":"data:image/jpeg;base64,..."}
   → 异步: {"metadata":{"queryParams":{"prompt_id":"<uuid>","signature":"<hex>"}}}

② 轮询 GET /api/image/generation?prompt_id={uuid}
   Headers: X-Signature: <signature>
   未完成 → {"image":""}    完成 → {"image":"data:image/jpeg;base64,..."}

前端轮询策略：38 次（前 2 次间隔 3s，之后 5s，约 184s 超时）
错误码：401=需登录 · 409=状态冲突 · 429=限流（读 Retry-After）
```

### 关键参数

- **Turnstile sitekey**（invisible，免费生成用）：`0x4AAAAAACJLXCF4CTzGutkF`
- **免费模型**：`zimage-turbo`（badge FREE）、`flux2klein`（0 积分）
- **图片 CDN**：`https://loveimages.b-cdn.net`（Bunny CDN）

### 全部模型枚举

```
图片: zimage | zimage-turbo | zimage-turbo-nsfw | zimage-base(-vip)
      banana | banana-2 | banana-pro        ← Gemini 2.5 系列
      flux2klein(-9b/-nsfw/-sfw) | flux-2-pro | *-faceswap
视频: veo-video-default/1080p | grok-video(-10s/-15s)
      ltx-2.3(-fast)-10s/20s | minimax-h3-10s | wan-v2.2-5b/a14b-turbo
```

### 其他端点

```
POST /api/image/prompt/free   免费提示词扩写 {userInput,sfw,modelType} → {positive}
POST /api/image/describe      图生文
GET  /api/image/generation/character  角色立绘
POST /api/proxy/openai        聊天代理 (chat/completions)
GET  /api/user/balance        积分余额
```

## 技术栈判定

Next.js (App Router/RSC) + Zod + Tailwind；Cloudflare (WAF+Turnstile) → CloudFront → 源站；middleware 多站点 rewrite（`/sites/{domain}/...`）。

## 待验证 / 已知边界

- `signature` 生成算法在服务端，前端不可见
- Turnstile token 需真实浏览器环境过 challenge，是自建 2api 的主要障碍
- 服务端限流阈值未知

## 免责声明

本项目仅用于**安全研究与学习目的**。所有接口数据来自公开前端产物的静态分析与个人浏览器抓包。请勿用于对目标站点的攻击、滥用或商业用途。使用者自行承担相关责任。

## License

MIT
