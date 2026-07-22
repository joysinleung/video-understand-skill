# 平台 Cookie 需求与 Token 效率参考

> 本文件从 SKILL.md 渐进式披露（progressive disclosure）出来，仅在需要时查阅，避免主文件过长。

## 平台 Cookie 需求速查（仅影响 URL 下载环节）

skill 本身**不依赖 cookie**——cookie 只是部分平台「下载」环节 yt-dlp 的要求。

**实测确认不要 cookie（2026-07-21 真跑过）**：YouTube ✅ | Bilibili ✅ | Dailymotion ✅

**必须 cookie 黑名单**：抖音 / TikTok / X / Instagram / Facebook / 微博 / 小红书

**回退铁律**：URL 被 cookie 拦 → 让用户发本地视频文件 → 走本地路径模式，**永不需 cookie、永不出网**。

**判定法**：先 `yt-dlp -s <url>` 无 cookie 试；报错 `Fresh cookies needed`/`401`/`403` 即需 cookie → 转回退。

## Token 效率参考

| 内容 | Token 估算（约）|
|------|----------------|
| 80 帧 @ 512px | 50–80k image tokens |
| 转写（10min 视频）| ~3k text tokens |
| 分辨率 1024 | 图像 token ×4（仅必要时用）|

同一会话内同一视频**勿重复跑脚本**——已有帧和转写在上下文中，直接回答即可。
