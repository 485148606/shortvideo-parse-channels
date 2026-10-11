---
name: 短视频平台解析通道
description: >-
  【短视频平台解析通道】>- 【短视频平台解析通道】抖音/快手/小红书/哔哩哔哩 四个短视频平台的链接解析通道与反爬应对。当需要从分享链接取回视频直链、封面、文案，或遇到"接口返回 200 但没数据""被限流""解析失败""需要签名"这类问题时使用。也适用于给小程序/App 做视频提取后端。触发词：解析抖音、去水印、视频提取、快手解析、小红书解析、ttwid、a_bogus、APOLLO_STATE。
agent_created: true
---

# 短视频平台解析通道

四个平台的解析路线，全都实测验证过（2026-09）。**核心结论：三个平台的失败原因都不是"需要逆向签名"，而是缺一个 cookie 或没重试。**

## 何时使用

- 要从抖音/快手/小红书/B站分享链接提取视频直链、文案、封面
- 接口返回 200 但 body 是空的、或页面里拿不到数据
- 做视频提取类小程序 / App 的后端

## 速查表

| 平台 | 通道 | 稳定性 | 一句话要点 |
|---|---|---|---|
| 哔哩哔哩 | 官方 API | ★★★★★ | 必须带 `Referer: https://www.bilibili.com` |
| 抖音 | 官方 web API（主）+ iesdouyin SSR（兜底） | ★★★★★ | **必须带 `ttwid` cookie** |
| 快手 | `www.kuaishou.com/short-video/{id}` SSR | ★★★ | 会风控（status=2），有移动端兜底 |
| 小红书 | 页面 SSR | ★★ | **必须带用户 Cookie（a1 / web_session）** |

---

## 抖音

### ★★★ 最关键：`ttwid` cookie

```
GET https://www.douyin.com/aweme/v1/web/aweme/detail/?aweme_id={id}&{浏览器参数}
  Cookie: ttwid=xxx          ← 只带这一个！
  Referer: https://www.douyin.com/video/{id}
```

**不带 ttwid → 返回 200 + 空 body。** 这个现象极具误导性：
看起来像"签名缺失"，会让人去逆向 `a_bogus`（几百行 SM3 算法，还容易失效）。
**实际上只要一个 ttwid 就全通，不需要任何签名。**

ttwid 白拿（无需登录）：
- 请求 `https://v.douyin.com/xxx`，跟 302 到 `iesdouyin.com`，响应头里就有 `Set-Cookie: ttwid=...`
- 或直接请求 `https://www.douyin.com/` 首页

**ttwid 可跨域用**：从 `iesdouyin.com` 拿的 ttwid 直接调 `www.douyin.com` 接口，实测有效。

**只带 ttwid，别带其他 cookie**：
- 显式 `Cookie: ttwid=xxx` → 39816 字节，有数据 ✓
- 让 cookie jar 自动带上全部（含 `__ac_nonce`）→ 0 字节 ✗

### 兜底：iesdouyin SSR（页面长度随机抖动，必须重试）

```
https://www.iesdouyin.com/share/video/{id}  →  window._ROUTER_DATA
  → loaderData['video_(id)/page'].videoInfoRes.item_list[0]
```

同一 URL 连续请求的长度分布：
```
2492    ← 字节跳动 WAF JS 挑战页（WAFJS / _wafchallengeid / s256 谜题）
32209   ← 骨架版，有 _ROUTER_DATA 但**没有 videoInfoRes**
42434   ← 有 videoInfoRes ✓
111963  ← /share/note/{id} 的富版本（图文作品更全）
```

判断标准**必须是"内容里有没有 videoInfoRes"**，不能看 HTTP 200。

### 其他

- 视频作品走 `/share/video/{id}`；图文作品走 `/share/note/{id}`
- **图文判定**：`video.play_addr.uri` 以 `.mp3` 结尾 = 图文（那个 uri 是配乐不是视频）
- 无水印：把直链里的 `/playwm/` 换成 `/play/`
- 图集字段：`images[]`，兜底 `image_post_info.images[]`

---

## 快手

```
v.kuaishou.com/xxx  →302→  v.m.chenzhongtech.com/fw/long-video/{photoId}
   ↓ 取 path 里的 photoId（不是 photoId 查询参数！）
https://www.kuaishou.com/short-video/{photoId}
   ↓ window.__APOLLO_STATE__.defaultClient['VisionVideoDetailPhoto:{photoId}'].photoUrl
```

`photoUrl` 就是无水印直链，实测下载返回 `206` + `ftyp` 魔数。

### 风控判定

APOLLO 里会有这个节点：
```json
"$ROOT_QUERY.visionVideoDetail({"page":"detail","photoId":"xxx"})":
  { "status": 2, "photo": null }
```
`status: 2` = **被反爬拦了**，不是"作品已删除"。触发后同 IP 会持续被拦一段时间。

### 兜底

`v.m.chenzhongtech.com/fw/long-video/{id}` 页面（180KB+）里内联了视频地址，
但它给的是 **HLS 实时转码流**（`.m3u8` / `hls-ts` 分片），不是无水印原片，只能应急。

### 别试

`https://www.kuaishou.com/graphql` 会直接返回风控验证码页：
`{"data":{"result":400002,"url":"https://captcha.zt.kuaishou.com/..."}}`

---

## 哔哩哔哩

```
b23.tv/xxx  →302→  www.bilibili.com/video/BVxxxxxxxxxx
  → api.bilibili.com/x/web-interface/view?bvid=...                  标题/简介/UP主/封面/cid
  → api.bilibili.com/x/player/playurl?bvid=&cid=&qn=80&fnval=0&platform=html5&high_quality=1
    → data.durl[0].url                                              单文件 mp4（720P）
```

- `fnval=0` 拿**单文件 mp4**；`fnval=16` 拿 dash（音视频分离，要自己合并，别用）
- **必须带 `Referer: https://www.bilibili.com/`**，否则 CDN 403
- `platform=html5&high_quality=1` 显著提高成功率和清晰度

---

## 小红书

未登录请求 `www.xiaohongshu.com/explore/{noteId}`：
- 返回 200，页面里有 `window.__INITIAL_STATE__`
- 但 **`noteDetailMap` 是空对象 `{}`** —— 只有骨架没内容
- 微信 UA 访问 → 302 跳 `open.weixin.qq.com` 授权
- Googlebot / Baiduspider UA → 10KB 空壳

需要 `x-s` / `x-t` / `x-s-common` 签名头。**两个可行解法**：
1. **让用户粘一次自己的 Cookie**（推荐）—— 需要 `a1` 和 `web_session`，带上才可能出数据
2. **接第三方聚合解析 API** 兜底

### 免费公开接口全死了

实测 `api.cooluc.com` / `api.pearktrue.cn` / `api.kuleu.com` / `api.52vmy.cn` /
`api.oioweb.cn` / `vlink.cc` / `qiaiwan.top`：
**不是 404 就是 512「套餐到期」或 522**。别指望免费公开接口，商用要么买付费
（约 20~50 元/万次）要么自建。

---

## 通用工程要点

1. **必须实现进程内 cookie jar** —— 抖音离了它直接废，不是可选项
2. **必须重试** —— 抖音/快手的 SSR 都是概率性返回数据，重试 4 轮 + 递增间隔
3. **不要相信 HTTP 200** —— 200 也可能是 WAF 页 / 骨架页 / 风控页。
   判断标准永远是"内容里有没有你要的那个字段"
4. **零依赖实现**：云函数环境常常没有 axios/fetch，用 Node 原生 `https` 自己封装，
   记得设 `Accept-Encoding: identity` 免去 gzip 解压
5. **小程序拿不到直链** —— 域名白名单拦死第三方 CDN。
   必须云函数转存到云存储，小程序用 `wx.cloud.downloadFile`
6. **云函数超时**默认 3 秒，转存视频要改成 60 秒

## 排查清单（解析失败时按顺序查）

1. 有 cookie 吗？（抖音查 ttwid，小红书查 a1/web_session）
2. 重试够不够？（至少 4 轮）
3. 响应体长度是多少？对照上面的长度表 —— 2.5KB = WAF，32KB = 骨架，42KB+ = 有数据
4. 快手看 `visionVideoDetail.status` 是不是 2
5. B站有没有带 Referer
6. 是不是该作品本身已删除/私密（换个链接验证）
