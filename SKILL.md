---
name: video-understand
slug: video-understand
displayName: "video-understand 通用 AI 视频理解"
version: "1.2.0"
description: "让 AI Agent 读懂视频——通过抽帧 + 语音转写 + 多模态读帧，综合产出视频理解：拆爆款结构、找 bug、总结要点、时间戳定位、提取口播稿。触发词：读懂视频、看视频、分析视频、拆爆款、看录屏、视频理解、这个视频讲啥。适配所有支持 Bash + 多模态 Read 的 agent（WorkBuddy/Codex/Cursor/Trae/Claude Code/Gemini CLI 等）。"
argument-hint: "<video-url-or-path> [question]"
allowed-tools: Bash, Read, WebSearch, WebFetch
license: MIT (底层脚本来自 bradautomates/watch)
author: joysinleung
homepage: "https://github.com/joysinleung/video-understand"
agent_created: true
---

# video-understand · 通用 AI 视频理解技能

让任意 AI Agent 能把一个视频**真正看懂**——不是只看标题或元信息，而是解析画面内容 + 语音转写，按需求产出结构化结论。

## 零、通用适配声明

本 skill **不绑定任何特定 AI 平台或品牌**。核心调用链只有两个原子能力：
1. **Bash**：跑 Python 脚本下载视频 / 抽帧 / 转写语音
2. **Read（多模态）**：逐张读取帧图理解画面内容

只要你的 agent 支持以上两个能力，就能用这个 skill。已验证兼容：
| Agent / 平台 | 兼容性 | 备注 |
|-------------|--------|------|
| WorkBuddy | ✅ 原生 | Bash + Read 多模态 |
| Claude Code | ✅ 原生 | 底层脚本原始开发平台 |
| Codex (OpenAI) | ✅ | Bash + Read 图片 |
| Cursor | ✅ | Agent 模式 |
| Trae (字节) | ✅ | Agent 模式 |
| Gemini CLI | ✅ | Bash + Read |
| Windsurf / 其他 | ⚠️ 待验 | 需支持多模态 Read |

> 底层脚本链源自 [bradautomates/watch](https://github.com/bradautomates/claude-video)（MIT），原项目名为 claude-video（指代 Claude Code 插件生态），但**脚本本身是纯 Python 工具链（yt-dlp + ffmpeg + Whisper API），不含任何平台锁定代码**。本 skill 已将其抽象为通用范式。

## 一、前置依赖（首次运行自检）

| 依赖 | 状态 | 缺失时处理 |
|------|------|-----------|
| `ffmpeg` / `ffprobe` | 通常已装（`/opt/homebrew/bin`） | `brew install ffmpeg` |
| `GROQ_API_KEY` | 必须存在于 `~/.config/watch/.env` | 配置方法见下方；用于 Groq Whisper 转写语音（纯 stdlib 调用，无需装 SDK）|
| `yt-dlp` | **仅当 source 是 URL 时需要** | `brew install yt-dlp` 或 `pip install yt-dlp`；本地文件路径模式不需要 |
| 底层 Python 脚本 | 已随 skill 打包在 `scripts/`（开箱即用） | 若缺失，重装 skill 或从仓库拉取 |

### GROQ_API_KEY 配置（仅需一次）
1. 打开 https://console.groq.com 注册 → API Keys → Create Key
2. 写入 `~/.config/watch/.env`：`GROQ_API_KEY=gsk_...`
3. 该文件不在任何 git 仓库内，安全

> 自检命令（Bash）：`for b in ffmpeg ffprobe yt-dlp; do command -v $b >/dev/null && echo "$b OK" || echo "$b 缺失"; done; test -f ~/.config/watch/.env && echo "GROQ key OK" || echo "GROQ key 缺失"`

### 脚本位置
本 skill 已自带 `scripts/`（与 SKILL.md 同级，已打包，开箱即用），无需再去 clone 任何上游仓库。查找顺序：
```
1. <SKILL_DIR>/scripts/                     ← 默认位置（已打包）
2. 环境变量 VIDEO_UNDERSTAND_SCRIPTS 指向的目录  ← 自定义脚本位置（含 watch.py）
```
> `<SKILL_DIR>` = 本 SKILL.md 所在目录的绝对 path。agent 加载此文件时通常能拿到该路径。

## 二、工作流程（强制按序）

1. **确认视频源**：本地绝对路径（`/Users/.../xxx.mp4`）或 URL（需 yt-dlp）。
2. **跑 watch.py 提取帧 + 字幕**（Bash，用系统 Python 或指定 Python）：
   ```bash
   # 本地文件（推荐：零联网、零 cookie、视频不出本机）
   python3 <SKILL_DIR>/scripts/watch.py "/绝对路径/视频.mp4" --detail balanced --out-dir <workdir>
   # URL 模式（需 yt-dlp；部分平台需 cookie，见第七节）
   python3 <SKILL_DIR>/scripts/watch.py "URL" --detail balanced --out-dir <workdir>
   ```
   - `<SKILL_DIR>` = 本 SKILL.md 所在目录（即包含 `scripts/` 的那个文件夹）
   - 默认全片 balanced 模式（scene-aware，cap 100 帧）
   - 长视频（>10min）聚焦一段：`--start HH:MM:SS --end HH:MM:SS`
   - 只要语音/字幕、不要画面：`--detail transcript`
   - 只抓某几个时刻的帧：`--timestamps 01:23,02:45`
   - 看录屏 / 幻灯片找 bug：`--timestamps 00:12,01:45 --no-dedup`（关闭近重复帧折叠，否则相似帧被合并）
   - 自定义分辨率：`--resolution 1024`（默认 512，需要读屏幕文字时才调高）
   - 产物：帧图落在 `<workdir>/frames/`，报告（含 `## Frames` + `## Transcript`）打印到 stdout
3. **解析 stdout**：watch.py 输出 markdown 报告到 stdout，含
   - `## Frames`：帧绝对路径列表（`frames/frame_XXXX.jpg`，带 `t=MM:SS` 时间戳 + 选中原因 `reason`）
   - `## Transcript`：时间轴字幕文本块（来自内嵌字幕，或 Groq Whisper API 转写）
4. **用多模态 Read 工具逐个读 `<workdir>/frames/` 里列出的帧图**——这是"看懂"的关键步骤，不能跳过。
5. **综合帧画面 + 字幕时间轴**，按用户需求产出结论。

## 三、按需求产出（对用户说人话）

- **拆爆款**：结构拆解（钩子/节奏/镜头语言/文案公式）、可复用模板、情绪曲线
- **找 bug / 看录屏**：定位问题帧 + 时间戳 + 根因 + 修复建议
- **总结要点**：要点清单 + 时间戳索引（方便回跳）
- **提取脚本**：口播稿全文 / 字幕 txt / 关键信息卡片
- **内容选题**：从视频里提炼可二创的角度

## 四、与知识库联动（WorkBuddy 体系专属）

以下节仅适用于使用 WorkBuddy 记忆库体系的场景。其他 agent 可忽略。

- 产出可落盘：拆爆款 → `灵感库/`；技术录屏 → `工作记录/` 或 `AI研究/`；值得反复看 → 记 `灵感原始/`
- 挂载对应**领域 profile**：网安视频 → `领域profile-网络安全.md`；数字人 → `领域profile-数字人.md`；搞钱 → `领域profile-搞钱.md`；通用 AI → `领域profile-AI.md`
- 与上下文工程呼应：超长视频务必分段（`--start/--end`），控 token 预算

## 五、红线（绝对遵守）

- **不**代为下载明显侵权内容；URL 模式仅用于用户有权访问的视频
- 视频不出本机本地处理；Groq 仅传 mono 16kHz 音轨用于转写（符合隐私，且 whisper.py 自带 24MB 分块上限）
- **绝不**把 `~/.config/watch/.env` 里的 API key 写进任何日志 / 记忆 / 会同步出去的文件
- 读帧图数量受 `--detail` cap 限制（balanced=100，efficient=50），超长视频稀疏是正常的，提示用户用聚焦模式

## 六、已知边界

- URL 视频依赖 yt-dlp 出网下载；若沙箱禁网或 yt-dlp 未装，请用户提供本地文件路径
- Groq 免费档有速率限制（whisper.py 内置 429 重试）；超大视频（>30min）转写可能较慢
- 画面理解质量取决于抽帧密度（detail 模式）+ 给定的分辨率（`--resolution`，默认 512）

## 七、平台 Cookie 需求速查（仅影响 URL 下载环节）

> WorkBuddy 用户可参考记忆库内的《video-understand-平台Cookie速查表》获取 1752 站点分类与方法论；非 WorkBuddy 用户忽略本句。

skill 本身**不依赖 cookie**——cookie 只是部分平台「下载」环节 yt-dlp 的要求。

**实测确认不要 cookie（2026-07-21 真跑过）**：YouTube ✅ | Bilibili ✅ | Dailymotion ✅

**必须 cookie 黑名单**：抖音 / TikTok / X / Instagram / Facebook / 微博 / 小红书

**回退铁律**：URL 被 cookie 拦 → 让用户发本地视频文件 → 走本地路径模式，**永不需 cookie、永不出网**。

**判定法**：先 `yt-dlp -s <url>` 无 cookie 试；报错 `Fresh cookies needed`/`401`/`403` 即需 cookie → 转回退。

## 八、Token 效率参考

| 内容 | Token 估算（约）|
|------|----------------|
| 80 帧 @ 512px | 50–80k image tokens |
| 转写（10min 视频）| ~3k text tokens |
| 分辨率 1024 | 图像 token ×4（仅必要时用）|

同一会话内同一视频**勿重复跑脚本**——已有帧和转写在上下文中，直接回答即可。
