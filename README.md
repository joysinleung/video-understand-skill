# video-understand · 通用 AI 视频理解技能

让任意 AI Agent 把一个视频**真正看懂**——不是只看标题或元信息，而是解析画面内容 + 语音转写，按需求产出结构化结论：拆爆款结构、找 bug、总结要点、时间戳定位、提取口播稿。

> 本 skill 不绑定任何特定 AI 平台。核心调用链只有两个原子能力：**Bash**（跑 Python 脚本抽帧 / 转写）与 **Read（多模态）**（逐张读帧图）。只要你的 agent 支持这两点，就能用。

> 版本：1.2.0 ｜ 许可证：MIT

---

## 一、30 秒快速开始

`<skill_dir>` = 本 skill 目录的绝对路径（即包含 `SKILL.md` 的那个文件夹）。本仓库 clone 后就是该目录本身，例如 `~/skills/video-understand`；你自己装到哪里就填哪里。

**① 本地视频文件（推荐：无需联网、无需 cookie、视频不出本机）**
```bash
python3 <skill_dir>/scripts/watch.py "/绝对路径/视频.mp4" --detail balanced --out-dir ~/watch-out
```

**② 视频 URL（需 yt-dlp；部分平台需 cookie，见第七节）**
```bash
python3 <skill_dir>/scripts/watch.py "https://youtu.be/XXXX" --detail balanced --out-dir ~/watch-out
```

跑完会打印一份 markdown 报告到终端，并生成帧图目录 `<out-dir>/frames/`。**用多模态 Read 工具逐个打开报告中 `## Frames` 下列出的帧图路径**，再结合 `## Transcript` 字幕时间轴，按你的需求产出结论。

> 先确认环境就绪：`python3 <skill_dir>/scripts/setup.py --check`（无输出即 OK）。

## 二、它解决什么问题

普通 agent 拿到一个视频链接或文件，往往只能读标题、描述或你手动贴的字幕。video-understand 把视频拆成「帧图 + 时间轴字幕」，让 agent 真正"看"到画面、听到语音，再综合产出人话结论。

典型用法：

- **拆爆款**：结构拆解（钩子 / 节奏 / 镜头语言 / 文案公式）、可复用模板、情绪曲线
- **看录屏 / 找 bug**：定位问题帧 + 时间戳 + 根因 + 修复建议
- **总结要点**：要点清单 + 时间戳索引（方便回跳）
- **提取口播稿**：全文 / 字幕 txt / 关键信息卡片
- **内容选题**：从视频里提炼可二创角度

---

## 三、兼容性

| Agent / 平台 | 兼容性 | 备注 |
|-------------|--------|------|
| WorkBuddy | ✅ 原生 | Bash + Read 多模态 |
| Claude Code | ✅ 原生 | 底层脚本原始开发场景 |
| Codex (OpenAI) | ✅ | Bash + Read 图片 |
| Cursor | ✅ | Agent 模式 |
| Trae (字节) | ✅ | Agent 模式 |
| Gemini CLI | ✅ | Bash + Read |
| Windsurf / 其他 | ⚠️ 待验 | 需支持多模态 Read |

---

## 四、安装

### 方式 A：WorkBuddy 推荐市场一键安装（仅当技能被策展入库后）
若 `video-understand-skill` 已被收录进 WorkBuddy 的「推荐市场 / BuiltinMarket」，在 WorkBuddy 中说一句「安装 video-understand-skill」即可一键装好（脚本随包就绪）。

> ⚠️ **重要事实**：把本仓库推到 **GitHub 不会自动**让它在 WorkBuddy 市场里被搜到。WorkBuddy 的应用内「推荐市场」是平台**策展列表（BuiltinMarket）**，安装器只在该列表内按 `skillId` 搜索，**没有自动爬取 GitHub 的机制**。发布到代码托管平台 ≠ 上架市场。必须先经提交流程进入 BuiltinMarket，用户才能"一句话安装"（见文末「如何上架推荐市场」）。

### 方式 B：从 GitHub 手动安装（任意 agent，最可靠、立即可用）
无论有没有上架市场，clone 本仓库到 skills 目录就能立刻用，无需等待策展：
```bash
# 用户级（所有项目可用）
git clone https://github.com/joysinleung/video-understand-skill.git ~/.workbuddy/skills/video-understand

# 项目级（仅当前项目；注意：若项目 .gitignore 忽略了 .workbuddy 则仅本地用）
git clone https://github.com/joysinleung/video-understand-skill.git <你的项目>/.workbuddy/skills/video-understand
```
clone 后目录需含 `SKILL.md` / `README.md` / `LICENSE` / `scripts/`（本仓库已齐备）。首次运行见「一、快速开始」。

> 注：本技能以「我的项目/video-understand」为 git 主仓（GitHub: `joysinleung/video-understand-skill`）；Claw 工作区里的副本仅作本地加载、被 `.gitignore` 排除。

---

## 五、前置依赖

| 依赖 | 必须？ | 缺失时处理 |
|------|--------|-----------|
| `ffmpeg` / `ffprobe` | 必须 | `brew install ffmpeg`（macOS）/ `apt install ffmpeg`（Linux） |
| `GROQ_API_KEY` | 推荐 | 写入 `~/.config/watch/.env`；无 key 时无声字幕的视频仅出帧、不出转写 |
| `yt-dlp` | 仅 URL 模式需要 | `brew install yt-dlp`；本地文件路径模式不需要 |

### 配置 Groq Key（仅需一次）
1. 打开 https://console.groq.com → API Keys → Create Key
2. 写入 `~/.config/watch/.env`：`GROQ_API_KEY=gsk_...`
3. 该文件不在任何 git 仓库内，安全

`setup.py` 有三种模式：
```bash
python3 <skill_dir>/scripts/setup.py --check    # 静默自检，无输出=就绪（exit 0）
python3 <skill_dir>/scripts/setup.py --json     # 机器可读状态（给 agent 解析）
python3 <skill_dir>/scripts/setup.py            # 首装向导：自动装依赖(macOS brew) + 生成 ~/.config/watch/.env
```
首装向导若检测到缺 ffmpeg/yt-dlp 会自动 `brew install`；生成的 `.env` 留好 `GROQ_API_KEY=` 占位，你填 key 后即全功能。

---

## 六、使用流程

### 6.1 触发方式
- **Agent 自动**：把 skill 装进 skills 目录后，agent 读取 `SKILL.md` 自动按"抽帧 → 读帧 → 综合"流程干活，你只需给视频源 + 需求。
- **人肉手动**：照下面命令自己跑，方便排查或验证（`<skill_dir>` 含义见「一、快速开始」）。

### 6.2 确认视频源
- **本地绝对路径**：`/Users/you/视频.mp4` —— **无需 yt-dlp、无需 cookie、视频不出本机**，最省心。
- **URL**：`https://...` —— 需 `yt-dlp`；YouTube / B 站 / Dailymotion 公开链直下，抖音等需 cookie → 回退发本地文件（见第七节）。

### 6.3 跑 watch.py
```bash
# 本地文件（推荐，零联网）
python3 <skill_dir>/scripts/watch.py "/绝对路径/视频.mp4" --detail balanced --out-dir ~/watch-out

# 长视频只聚焦一段
python3 <skill_dir>/scripts/watch.py "URL" --start 00:03:00 --end 00:05:00 --detail balanced

# 只要语音/字幕（不抽帧）
python3 <skill_dir>/scripts/watch.py "URL" --detail transcript

# 抓录屏里某几个固定时刻（适合"看录屏找 bug"，--no-dedup 保留相似帧）
python3 <skill_dir>/scripts/watch.py "/录屏.mp4" --timestamps 00:12,01:45 --no-dedup
```

参数速查：

| 参数 | 作用 |
|------|------|
| `--detail` | `transcript`(仅字幕) / `efficient`(关键帧,封顶 50) / `balanced`(场景感知,封顶 100,默认) / `token-burner`(场景,不封顶) |
| `--resolution` | 帧宽像素，默认 512；读屏文字多可调 `--resolution 1024` |
| `--start` / `--end` | 聚焦时间段（支持 `SS` / `MM:SS` / `HH:MM:SS`） |
| `--timestamps` | 指定绝对时刻抽帧，逗号分隔（如 `01:23,02:45`） |
| `--max-frames` | 覆盖帧数上限 |
| `--no-dedup` | 关闭近重复帧折叠——**录屏 / 静态幻灯片必备**，否则相似帧被合并 |
| `--whisper groq\|openai` | 强制转写后端；`--no-whisper` 关闭转写回退 |
| `--out-dir` | 工作目录，默认系统临时目录 `watch-*` |

### 6.4 产物在哪、长啥样
- **帧图**：落在 `<out-dir>/frames/`（如 `~/watch-out/frames/`），文件名含时间戳。
- **markdown 报告**：打印到终端 stdout，含：
  - `## Frames` —— 逐帧**绝对路径** + `t=MM:SS` + 选中原因（`reason`）。
  - `## Transcript` —— 时间轴字幕（来源：`captions` / `whisper`）。
- 报告末尾给出 `Work dir: <out-dir> — delete when done.`，跑完可删。

### 6.5 "看懂"的关键一步（不能跳过）
**用多模态 Read 工具逐个打开 `## Frames` 里列出的帧图路径**，真正"看"画面；再综合字幕时间轴，按你的需求（拆结构 / 找 bug / 总结 / 提口播稿）产出结论。只解析 stdout 文本而不读帧图 = 没真正看懂画面。

---

## 七、平台 Cookie 说明（仅影响 URL 下载）

skill 本身**不依赖 cookie**——cookie 只是部分平台「下载」环节 yt-dlp 的要求。

- **实测不要 cookie**（公开链接直下）：YouTube ✅ ｜ Bilibili ✅ ｜ Dailymotion ✅
- **必须 cookie 黑名单**：抖音 / TikTok / X / Instagram / Facebook / 微博 / 小红书
- **回退铁律**：URL 被 cookie 拦 → 让用户发本地视频文件 → 走本地路径模式，**永不需 cookie、永不出网**。
- **判定法**：先 `yt-dlp -s <url>` 无 cookie 试；报 `Fresh cookies needed` / `401` / `403` 即需 cookie → 转回退。

---

## 八、红线

- 不代为下载明显侵权内容；URL 模式仅用于用户有权访问的视频。
- 视频不出本机本地处理；Groq 仅传 mono 16kHz 音轨用于转写（whisper.py 自带 24MB 分块上限）。
- **绝不**把 `~/.config/watch/.env` 里的 API key 写进任何日志 / 记忆 / 会同步出去的文件。

---

## 九、许可证与署名

MIT License。底层脚本改编自 [bradautomates/watch](https://github.com/bradautomates/claude-video)（MIT，Copyright (c) 2026 Bradley Bonanno），本 skill 仅将其抽象为通用、去平台化的范式。详见 `LICENSE`。

---

## 十、如何上架 WorkBuddy 推荐市场（BuiltinMarket）

- **现状**：经核对 WorkBuddy 内置 `marketplace-skill-installer` 源码，应用内安装器只向 BuiltinMarket（平台策展列表）发起 `search` / `install`，**不支持从 GitHub URL 直接安装或自动索引**。
- **要让用户"一句话安装"，需先把本 skill 提交进 BuiltinMarket 策展列表**。公开文档（codebuddy.cn/docs/workbuddy）目前未给出自助提交通道的具体入口/表单，通常走平台方提供的技能提交流程（可在 WorkBuddy 客户端「技能管理」或官方渠道咨询提交方式）。
- **在此之前**：对外分发与安装请走「方式 B」（GitHub clone），这是确定可用的路径。本仓库本身即是完整的可 clone 单元。
