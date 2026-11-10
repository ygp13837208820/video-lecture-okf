---
name: video-lecture-okf
description: 要把一整套讲课视频做成中英对照讲义与 OKF 知识包时用：网关取件、faster-whisper 转写、双语与硬字幕、按打分选关键帧配图注、OKF v0.2 拆分与三件套校验；附 ctranslate2 崩溃、502 退避、图注错配等坑。
---

# video-lecture-okf

把「一整套讲课视频」变成可交付的中文讲义 + OKF 知识包。一次完整跑通的实例是 **LLE26 On Demand 9 讲**（48 分 53 秒，英→中）：产出 1 份双语对照讲义、9 个 OKF 概念文件（另加 index/log/附录共 12 个 .md）、43 张 1280×720 关键帧、9 讲软/硬字幕。下文所有数字都来自那次交付，不是估算。

**先读 §2**：那 30 条是真正花时间换来的，§1 只是骨架。踩过的坑里最贵的三个是——图注错配（§2 第 19 条）、ctranslate2 版本崩（第 6 条）、把质量分当选择分（第 18 条）。

## §0 先决条件与目录约定

### 0.1 目录（固定，别自创）

```text
E:\WorkBuddy交付\<主题>\
├─ 原始480p\            原片 mp4（名字是 YouTube video id，如 pkGX2r_kNPM.mp4）
├─ 中间产物\
│   ├─ audio\<id>.wav       16 kHz 单声道
│   ├─ transcripts\<id>.json {duration, segments[{start,end,text}]}
│   ├─ cues\<id>.json        断句后的双语 cue（i/start/end/en）
│   └─ zh\<id>.json          按 cue 编号的中文译文 {"1": "…"}
├─ 字幕srt\<id>.srt / <id>.zh.srt    双语版 + 纯中文版（UTF-8 BOM）
├─ 软字幕\<讲名>.mp4         mov_text 软字幕
├─ 硬字幕\<讲名>.mp4         烧录
└─ okf\
    ├─ index.md  log.md                包根，index 必须 okf_version: "0.2"
    ├─ 01-<概念名>.md … 09-<概念名>.md  一个概念一个文件
    ├─ 附录-取件与转写记录.md           type: Note，写来源与复现命令
    └─ assets\<slug>\fig1.jpg …        43 张，统一 1280×720
```

脚本全部放 `C:\Users\DELL\.dsh\bot\workspace\<项目名>\`（C 盘只放脚本，< 1 MB）。
模型、库、缓存放 `E:\dsh\<项目名>-env\`（本案例 `E:\dsh\lle26-env\{models,fwlib}`）。

### 0.2 环境（本机已装，不要新装依赖）

| 项 | 值 | 备注 |
|---|---|---|
| ffmpeg | PATH 里 | 取帧、混流、烧字幕全靠它 |
| Python | 3.10 / 3.11 | 系统 python 即可 |
| faster-whisper | 1.2.1 | 装进 `E:\dsh\lle26-env\fwlib`，用 `sys.path.insert(0, …)` 引入 |
| ctranslate2 | **必须 4.4.0** | 4.8.5 在 Windows 一加载模型就崩，见第 6 条 |
| onnxruntime | 有但不可用 | DLL 初始化失败 ⇒ 只能关 VAD，见第 7 条 |
| 模型 | `medium.en`（1.46 GB） | `E:\dsh\lle26-env\models\fw-medium-en` |
| numpy / Pillow | numpy 2.x / Pillow 12.3 | 质量度量、图像返工用 |

每开一个终端先做：

```powershell
chcp 65001 | Out-Null
$env:PYTHONIOENCODING = "utf-8"        # 不设的话中文路径+中文 print 会 UnicodeEncodeError/乱码
$PROJ = "E:\WorkBuddy交付\<主题>"
$WORK = "C:\Users\DELL\.dsh\bot\workspace\<项目名>"
```

## §1 流水线（7 步，每步都能单独重跑）

> 下面每个 `python xxx.py` 都是放在 `$WORK` 里的一次性工具脚本，**本技能不带脚本副本**（它们绑死了上一次的绝对路径和常量，拷过来只会误导）。照每节写明的参数/规则重写，单个 30–80 行，半天能重建全套。

### 1.1 取件

```powershell
cd $WORK
python download.py        # 读 videos.json，逐条 /api/resolve → 下载到 原始480p\<id>.mp4
```

如果只是要单条视频的逐字稿（不做讲义），直接用 `digest-md-pipeline` 通道 I，它封装了同一个网关，别在这里重复造轮子。

`videos.json` 的最小结构：

```json
[{"id": "pkGX2r_kNPM", "title": "Barrett's Oesophagus assessment", "url": "https://youtu.be/pkGX2r_kNPM"}]
```

下载要点（照抄，别改）：

- 用 `urllib` + `Range: bytes=<已有>-` 断点续传，写 `.part` 再 `os.replace`；单条未完成时**只重试这一条**。
- 网关返回的 `quality` 字段是标称值，**落盘后一律 ffprobe 核实**：

```powershell
ffprobe -v error -select_streams v:0 -show_entries stream=width,height,codec_name,r_frame_rate `
        -of default=nw=1 "$PROJ\原始480p\pkGX2r_kNPM.mp4"
```

- 首轮失败不要并发硬冲：单条间隔 20 s，一轮跑完仍有失败就等 180 s 再整轮重跑（9 条首轮成 4 条 → 第二轮 9/9）。

#先搜索再取件（推荐路径）：

```powershell
# 1) 搜关键词，拿 videoId
curl.exe -s "https://yt-gateway.pages.dev/api/search?q=endoscopy+ESD&limit=10"
# 2) 用 videoId 解析直链（40–45 s，timeout ≥90 s）
# 3) 直链直下到 原始480p\<id>.mp4（CDN 2–6 MB/s，支持 Range）
```

### 1.1.1 网关 API 契约（取件的全部机制，照抄）

| 端点 | 参数 | 返回 | 用途 |
|---|---|---|---|
| `/api/resolve?url=` | 完整 watch URL（urlencode） | `{success, result:{title, thumbnail, quality, download_url}}`，响应头 `x-backend` 标识后端 | 拿到可直下的 mp4 链接 |
| `/api/channel?channel=` | UCid 或 `@handle` | `{channel, videos:[{videoId,title,published,views,duration}], nextpage, count}` | 枚举频道视频 |
| `/api/stream?u=` | 原始 URL | 字节流，支持 `Range` | **只在直链是 googlevideo 时才绕这一层**；savetube 直链直接下更快 |
| `/api/search?q=` | 关键词 + `limit`(≤30) + `page` + `sort=views` + `since=today\|week\|month\|year` | `{success, backend, count, videos:[{videoId,title,channel,published,duration,views,url}], continuation}` | **关键词搜索**（2026-11-08 新增）：直接拿 videoId 去 resolve |

- `download_url` 可能是**相对路径** `/api/stream?v=…`，必须自己拼上网关 origin，否则 `urlopen` 直接报错。
- 单次 resolve **40–45 s**（实测 43.5 s），`timeout` 不要低于 90 s。
- 直链落在 `cdn403/405.savetube.vip`，URL 里**没有 `ip=` 参数、不绑 IP**；实测 2–6 MB/s，支持 `Range` 续传。
- ✅ **search 已上线**（2026-11-08 用 ego-browser 直接提交进 `functions/api/search.js`，Pages 自动构建）：
  `GET /api/search?q=<词>&limit=10` → `{success, backend:"innertube", count, videos:[…], continuation}`，实测 4 组查询（含 `胃拡大内視鏡`）均 10 条、1.1–1.3 s；`page=<continuation>` 翻页第 2 页 10 条、重复 0；`q` 空或 >120 字符回 400；`limit` 上限 30。
  取件流程因此可以变成：**搜索 → 拿 videoId → resolve → 下载**，不必事先知道频道 UCid。
  **筛选参数**（2026-11-08 实测）：`since=today|week|month|year` 是**真能用**的时间范围过滤（params 取自真实页面的 `searchFilterRenderer`，protobuf 由 8 行编码器拼：field1 varint(sort) + field2{10 01} + field2{08 N}）；`sort=views` 映射筛选面板的 `Prioritize: Popularity`（`CAM=`）。
  ⚠️ **没有「按上传日期排序」这个入口**了：搜索页的 chip 全部走 continuation token，`Upload date` 只是时间范围组的标题。`sort=date` 会诚实返回 `x-sort: fallback` 并沿用默认排序——别写死 `EgQIAhAB` 之类猜出来的常量，错了不会报错，只会静默按相关度排。
  自证头：`x-sort: none|views|fallback`、`x-since: none|week|fallback`；`sort`/`since` 都不带时请求体与旧版逐字节相同（零回归）。
  ⚠️ **search 现在要带 header**（2026-11-08 起）：配了 `GATEWAY_KEY` 后，未带 `x-gateway-key` 一律 401。
  密钥在本机 `E:\dsh\lle26-env\gateway-key.txt`（32 位 hex），调用示例：
  `curl.exe -s -H "x-gateway-key: $(Get-Content E:\dsh\lle26-env\gateway-key.txt -Raw)" "https://yt-gateway.pages.dev/api/search?q=endoscopy&limit=10"`
  鉴权开着时响应是 `cache-control: no-store`（防边缘缓存绕过校验），`OPTIONS` 预检返回 204。
- 历史备注：search 上线前只能 ① `/api/channel` 枚举已知频道 ② 从别处拿 11 位 id；RSSHub `/youtube/search` 一直是 404/503，`yt-dlp ytsearch` 因网络不可达直接失败。
- ⚠️ `@handle` 解析**不可靠**：实测 `channel=@GIEQs` 返回的是完全无关的频道（印尼语手机维修视频），30 条全错。**优先用 UCid**（`UC…` 开头），拿不到就换来源，别信 handle 的结果。

> 写这一段时我自己踩了一次第 27 条：用 PowerShell here-string 写 Markdown，反引号被当成转义符吃掉，`timeout` 变成 `\timeout`、整块行内代码全丢。**改 Markdown 一律用 Python 写文件**。

#### 1.1.2 取件前先探测，失败按阶梯降级（2026-11-07 实测）

**这是硬要求**：任何批量取件开始前，先跑 30 秒探测矩阵，把结果写进 `附录-取件与转写记录.md`。
本机**只有网关一条路**，单点依赖必须显式写出来，不要假装有备份。

| 探针 | 怎么打 | 通过判据 |
|---|---|---|
| 解析 | `resolve?url=<一个已知能过的 id>` | `success=true` 且 `result.download_url` 非空 |
| 媒体 CDN | `HEAD <download_url>` | 200 |
| 频道枚举 | `channel=<UCid>` | `count>0` 且 `channel.name` 与目标频道一致 |

同一条视频、同一分钟内的四种输入形式（实测）：

| 输入形式 | 结果 | 耗时 |
|---|---|---|
| `watch?v=<id>` | 502 | 32.8 s |
| `youtu.be/<id>` | 502 | 27.0 s |
| `watch?v=<id>&t=30s` | **200** | 4.4 s |
| `shorts/<id>` | **200** | 4.8 s |

⇒ **降级阶梯**：原样重试 1 次 → 换输入形式（加 `&t=30s` 或改 `shorts/`）→ 等 180 s 再整轮重跑。
**502 是常态，不是异常**：同一分钟内换个形式就过；反之，200 也可能 4 s 就回、也可能 45 s 才回，别拿耗时判断成败。

其它实测结论（同一轮探测）：

- `/api` 根路径 **404** 属正常（Pages Functions 按文件路由）；`/api/search` 已于 2026-11-08 上线，见 §1.1.1。
- `channel=@GastroLearning` 返回的是 **Marcello Maida**（别人的频道）⇒ `@handle` 一律不可信，**只用 UCid**。
- RSSHub `youtube/channel/<UCid>` 21 s 超时（早前可用过）⇒ 只能算偶发备用，不要写进降级链。
- `redirector.googlevideo.com` / `m.youtube.com` / `youtubei.googleapis.com` 全部 `10051` ⇒ 媒体不可能直连 Google，必须走 CDN。
- 🔒 `/api/stream` **已加 host 白名单**（2026-11-08）：只放行 `googlevideo.com` / `savetube.vip` / `savetube.me` / `odycdn.com` / 含 `pipedproxy` 的域，可用 `STREAM_ALLOW` 追加；其它 host 返回 403。**它不再是“任意 URL 代理”**——之前用它抓 YouTube 页面做验证的路子已经堵死，要抓页面请走浏览器或本机可用通道。
- `cdn403/405.savetube.vip` HEAD 200 ⇒ **媒体字节不经过网关**，拿到直链就直下；走 `/api/stream` 只会被 19 KB/s 的中转拖慢。

**必做缓存**：每条 resolve 结果落 `中间产物\resolve\<id>.json`（`download_url` + `time`），重跑/断点续传先读缓存，超过 6 h 才重新解析。9 条视频能省约 6 分钟。

### 1.1.3 找「最新一条」：`newest.py`

脚本：`E:\dsh\lle26-env\tools\newest.py`（工具留在 env；**产物默认落到交付目录**）
默认输出：`E:\WorkBuddy交付\内镜-最新视频验证\`

> ⚠️ 位置铁律（2026-11-08 因犯错补写）：`E:\dsh\` 只放环境与工具，**任何要被用户双击打开的东西**
>（`verified.md`、帧图、成片）都必须进 `E:\WorkBuddy交付\<主题>\`；为了在对话里展示而拼的对比图同样算交付物，
>不许留在 `C:\Users\DELL\.dsh\bot\workspace\` 里。

```powershell
python newest.py --topic endoscopy --within today            # 今天最新
python newest.py --topic ERCP --within week --top 8          # 本周，列前 8
python newest.py --topic "colon polyps" --within week --verify --max-mb 40   # 下载+抽帧核实
```

它做的事：
1. 用 `/api/search` 的 `since=today|week|month|year` 检索，多个查询词合并去重；
2. **双重过滤**：必须命中内镜词（endoscop/ERCP/EUS/ESD/snare/resection…）**且**命中消化上下文（GI/gastro/colon/bile/pancrea/消化/胃镜…），并排除脊柱、关节、耳鼻喉、泌尿、妇科等同名“endoscopy”（`--any-endoscopy` 可关）；
3. 解析 `publishedTimeText`（`38 minutes ago` / `Streamed 15 hours ago`）按新鲜度排序；
4. `--verify` 会真下载（默认只下前 40 MB）+ ffmpeg 抽两帧，用来排除“标题像、画面是访谈/广告”。

已知坑：
- **标题不能当证据**：实测最靠前的几条是播客访谈、动物内镜广告、医院宣传片、动画讲解——只有抽帧后才看得出来；
- **直播 VOD 取不到**：`resolve` 对 live 一律 502；
- **resolve 会整段宕**：`davidcyriltech` 进冷却时全站 502（`{"error":"all backends failed"}`），而 `search`/`channel` 仍 200。脚本内置 3 次指数退避，但上游冷却 5 分钟起步，急不得；
- 中文查询（胃镜/肠镜/内镜）召回质量差，结果被短剧/美食淹没；英文技术词才稳。

### 1.2 转写

```powershell
python transcribe.py      # ffmpeg 抽 wav → faster-whisper → transcripts\<id>.json
```

关键参数（改一个都可能崩或掉质量）：

```python
sys.path.insert(0, r"E:\dsh\lle26-env\fwlib")
WhisperModel(r"E:\dsh\lle26-env\models\fw-medium-en",
             device="cpu", compute_type="int8", cpu_threads=4)
model.transcribe(audio, language="en", beam_size=5,
                 vad_filter=False,                     # VAD 装不上，只能 False
                 initial_prompt=PROMPT,                # 术语表，见下
                 condition_on_previous_text=False,
                 no_speech_threshold=0.6)
```

`initial_prompt` 不是可选项，是术语准确率的主要来源。本案例用的是：

```text
Endoscopy teaching: Barrett's oesophagus, Prague classification, dysplasia, gastric atrophy,
intestinal metaplasia, ESD, EMR, polyp characterisation, NBI, narrow band imaging, JNET,
Kudo pit pattern, withdrawal time, sphincterotomy, cannulation, ERCP, sphincteroplasty,
choledocholithiasis, Z-POEM, achalasia, EndoFLIP, EsoFLIP, submucosal tunnelling.
```

抽音频：`ffmpeg -nostdin -v error -y -i <id>.mp4 -ac 1 -ar 16000 -f wav <id>.wav`

### 1.3 字幕

```powershell
python build_subs.py      # cues + zh → 字幕srt\<id>.srt（双语）/ <id>.zh.srt（纯中文）+ 软字幕 mp4
```

- **合并规则**（572 段 ASR → 383 cue → 335 条字幕）：上一条不以 `.?!` 结尾、间隔 < 1.5 s、合并后 < 14 s ⇒ 并成一条；单条最短 0.8 s。
- srt 必须 `encoding="utf-8-sig"`（BOM）。无 BOM 的 srt 在不少播放器里中文乱码。
- 软字幕混流（不要重编码视频）：

```powershell
ffmpeg -nostdin -v error -y -i <id>.mp4 -i <id>.zh.srt -map 0 -map 1 `
       -c copy -c:s mov_text -metadata:s:s:0 language=zho -metadata:s:s:0 title=中文 out.mp4
```

- 硬字幕（`cwd` 必须是 srt 所在目录，`subtitles=` 里用相对路径最稳）：

```powershell
ffmpeg -y -v error -i <id>.mp4 `
  -vf "subtitles=<id>.zh.srt:force_style='FontName=Microsoft YaHei,FontSize=16,OutlineColour=&H66000000,BorderStyle=3,Outline=1,Shadow=0,MarginV=16'" `
  -c:v libx264 -preset veryfast -crf 23 -c:a copy "硬字幕\<讲名>.mp4"
```

### 1.4 讲义

按 `> **[mm:ss]** 英文原句` + `> 〔中〕中文` 成对排版，一段一对；正文按 `## N. 标题` 分节，节头下写 `` `2026-10-10` · 03:36 · <https://youtu.be/<id>> ``。
把 9 节拼起来时用 Python 读写（`encoding="utf-8"`），**不要用 PowerShell 的 `Get-Content`/`Set-Content`**（见第 27 条）。

### 1.5 关键帧（最费功夫的一步）

四段脚本，顺序固定：

```powershell
python quality.py         # 每讲 2 fps 灰度 → quality.json / sharpness.json
python select_v4.py       # 打分 + 分区选点 → selection_v4.json
# 人工过目选点，写 figs_v3.json（时刻、标题、图注、crop）
python rebuild_figs.py    # ffmpeg 取帧 → okf\assets\<slug>\figN.jpg，同时把图插进讲义与 okf
```

选帧打分（`select_v4.py`，实测能用的权重）：

```python
score = 0.40 * 叙事锚点 + 0.35 * 画面变化 + 0.25 * 事件
# 锚点：here / look at / you can see / now we / this is / I'm going to / watch …
# 画面变化：1 fps 帧差的 3 帧滑动平均，5–95 分位归一化
# 事件：蓝通道(≈NBI)、红通道(≈出血/电凝)相对另一对的超出量
```

选区与去重（这三条是"不选出 6 张几乎一样的图"的关键）：

- 每讲切 K 段（本案例 4–6），**每段只取该段最高分**——等分点只当"分区"，不当"选点"。
- 段内按分数取前 6 名，跳过开头 8 s、结尾 4 s。
- 与已选帧时间差 < 8 s、或 64×36 缩略图平均绝对差 < 6 ⇒ 跳过。

单张取帧（`crop=w:h:x:y`，注意 JSON 里存的是 `x:y:w:h`，**顺序要换**）：

```powershell
ffmpeg -y -v error -ss 00:00:52 -i "$PROJ\原始480p\<id>.mp4" -frames:v 1 `
  -vf "crop=424:366:850:186,scale=848:732:flags=lanczos" `
  "$PROJ\okf\assets\02-gastric-dysplasia\fig1.jpg"
```

### 1.6 OKF 拆分

```powershell
python okf_build.py       # 按 "## N." 切分讲义 → okf\0N-<概念>.md + 附录 + assets 引用
```

frontmatter 模板（键顺序别动，`license` 必须最后一行）：

```yaml
---
type: Article
title: 02-胃黏膜异型增生的评估
description: <一句话，说清这一讲讲什么>
resource: https://youtu.be/SERxp-PRwNk
tags: [内镜, 萎缩性胃炎, 早期胃癌, LLE26]
status: stable
generated: { by: ima.copilot/1.0, at: 2026-11-07T08:00:00Z }
sources:
  - resource: https://youtu.be/SERxp-PRwNk
    id: src-1
    title: "LLE26 On Demand: Assessment of the stomach for dysplasia"
    author: GastroLearning
    last_modified: 2026-10-17
language: en→zh
translated_at: 2026-11-07
license: 版权归 … 所有；仅供个人学习，不再分发
---
```

包根 `index.md` 必须有 `okf_version: "0.2"` 且 `type: Index`，`log.md` 为 `type: Log`。合并版讲义是 `type: Compilation`，不要和概念文件混在一起。

### 1.7 收尾

- 图注格式固定：`![图 N｜帧 mm:ss 标题](assets/<slug>/figN.jpg)` + 空行 + `**图 N｜帧 mm:ss 标题**：说明`。
- 讲义里同一段文字，图片路径换成 `okf/assets/…`。两份必须逐字一致（第 25 条给核对办法）。
- 写 `附录-取件与转写记录.md`：条数、总时长、分段数、取件源、转写参数、复现命令。这份是给下一个人（或下个 AI）省两小时的。

## §2 坑表（30 条，按阶段分组；编号在本技能内唯一）

### 2.1 取件与网络

| # | 症状 | 根因 | 处理 |
|---|---|---|---|
| 1 | YouTube/Google 连不上，报 `WinError 10051`（网络不可达） | 本机 DNS 解析到非公网 IP，路由不通 | **不是 SSL 问题**：别配证书、别关校验、别换代理，走网关 |
| 2 | 只有自建网关能出视频 | 云端中继 `https://yt-gateway.pages.dev`（Cloudflare Pages，`*.pages.dev` 可达） | 端点只有三个：`/api/channel?channel=`、`/api/resolve?url=`、`/api/stream?u=` |
| 3 | 以为拿到的是 480p | 网关返回 `quality=480p` 是标称值 | 实际文件 1280×720；**落盘后 ffprobe 核实**，标称≠事实 |
| 4 | 9 条首轮只成 4 条，返回 502 | 网关按需转码，瞬时过载 | 退避：单条间隔 20 s、整轮失败后等 180 s；第二轮 9/9 全成。**别并发** |
| 5 | `davidcyriltech` 这类频道直连直接被 reset | 本机对该域名的连接被中断 | 一律走网关，不要在 `download.py` 里加直连分支 |

### 2.2 转写

| # | 症状 | 根因 | 处理 |
|---|---|---|---|
| 6 | 加载模型瞬间进程消失，`0xC0000005` access violation，无栈回溯 | `ctranslate2` 4.8.5 在 Windows 的已知崩溃 | **锁 4.4.0**。别怀疑模型损坏——本地 `model.bin` 与 HF 端字节数一致 |
| 7 | `onnxruntime` DLL 初始化失败 | 依赖链不全 | VAD 只能关：`vad_filter=False`；长静音段会重复，靠 §1.3 的合并规则压掉 |
| 8 | C 盘被模型/库撑爆 | 默认装进用户目录 | 模型 1.46 GB 放 `E:\dsh\<项目>-env\models`，库放 `…\fwlib`，`sys.path.insert` 引入 |
| 9 | 转写耗时对不上 | CPU int8、4 线程约 **1.6× 实时**（48:53 音频 ≈ 30 min） | Z-POEM 那条 417 s 音频跑了 1772 s，是个案（长静音+口音），重跑前先看日志别急着改参数 |

### 2.2b 取件失败的三种 502（2026-11-08 实测，必须分开处理）

`/api/resolve` 返回 502 时，**body 里的 `errors[]` 说明原因**，别一律当“抖动”重试：

| errors 里的特征 | 真实原因 | 该怎么办 |
|---|---|---|
| `davidcyriltech: cooldown` + `piped: … 530` | 上游集体不可用 | 等 5–10 分钟再试（冷却 5 分钟起步） |
| `davidcyriltech: HTTP 429` | 我们把它打限流了 | 降低并发/频率；不要短间隔连打 |
| `innertube: WEB: playability LOGIN_REQUIRED` | **视频是年龄/登录门槛**，换任何 URL 形式都没用 | 放弃这条（要 cookies 才能取），不是“再试一次”的问题 |

判定方法：同一时刻拿一条已知正常的视频做对照。
- 对照也 502 ⇒ 全局故障，等；
- 只有目标条 502 且报 `LOGIN_REQUIRED` ⇒ 该视频本身受限，立刻放弃。

**年龄门其实有解（2026-11-08 实测）**：登录账号后，页面可能只剩一句「本视频可能不适合某些用户观看 / 我了解并希望继续观看」——
点掉它，`<video>.readyState` 会从 0 变 4、`duration` 才有值，这时 canvas 抽帧才拿得到真画面：

```js
// 1) 点同意按钮
[...document.querySelectorAll("button,a")].find(e => /我了解并希望继续观看|I understand and wish to proceed/.test(e.innerText))?.click();
// 2) 等 readyState>=2 后，逐帧：seek → 等 seeked → drawImage → toDataURL
//    ⚠️ 一次 evaluate 只抓一帧（抓 3 帧会超过 15 s 的 evaluate 超时）
```

实测这条链路拿到了 6:10 视频的 3 帧（541/709/577 KB，非黑帧）。**判据**：PNG 只有几 KB ⇒ 必然是黑帧；
几百 KB ⇒ 真画面。

**被年龄门挡住时怎么办**：换同主题、可取的替代视频来验证“这类内容长什么样”，但**必须在汇报里写明这是替代验证**，
不能拿替代品的画面去证明原目标。同一主题下优先选标题带 `case / live / real-time` 的（多为真实操作），
标题带 `board review / whiteboard / lecture` 的多为幻灯片（实测 TapGI 那类整片都是 slides，没有一帧操作画面）。

**二次确认（浏览器）**：用 ego-browser 打开 `watch?v=<id>`，读 `document.body.innerText`——
出现「请登录，以便我们确认你的年龄 / 视频有年龄限制（应上传者要求）」就坐实了，页面上还能顺手拿到
订阅数与时长。此时 `<video>.readyState` 恒为 0、`paused=true`，canvas 抽帧只会得到黑帧，
**不要把它当成“抽帧成功”**（我就差点把 8 KB 的空白 PNG 当证据）。

### 2.3 字幕

| # | 症状 | 根因 | 处理 |
|---|---|---|---|
| 10 | 字幕碎成一句一条 | ASR 按停顿切段 | 合并：无句末标点 + 间隔 < 1.5 s + 合并后 < 14 s（572 段 → 383 cue → 335 条） |
| 11 | 播放器里中文全是乱码 | srt 没写 BOM | `encoding="utf-8-sig"`；软字幕用 `-c:s mov_text -metadata:s:s:0 language=zho` |
| 12 | 烧字幕极慢 / 有的片子格外慢 | 第 1 讲源片是 **AV1**，其余 H.264 | 5 分钟片要 15 分钟；交付说明里必须写清"第 1 讲 AV1，老播放器可能不支持" |

### 2.4 OKF v0.2

| # | 症状 | 根因 | 处理 |
|---|---|---|---|
| 13 | `okf_check.py` 报缺 type / license 不在末位 | frontmatter 键顺序错 | `type` 必填；`license` **必须是最后一个键**；推荐 `title/description/resource/tags`，再加 `generated{by,at}`、`sources[]` |
| 14 | 包打不开 / 索引缺版本 | 包根缺 `okf_version` | `index.md` 必须含 `okf_version: "0.2"` 且 `type: Index`；`log.md` 记变更；`index/log` 是保留名，不能当概念文件 |
| 15 | 一个文件塞一整讲，检索不到 | 粒度错了 | 一个概念一个文件；合并版另存为 `type: Compilation` |

### 2.5 关键帧（血泪最多）

| # | 症状 | 根因 | 处理 |
|---|---|---|---|
| 16 | 选出来的帧没信息量 | 只有清晰度 | 打分 = 0.40 锚点 + 0.35 帧差 + 0.25 事件（§1.5 的正则与公式） |
| 17 | 全片扎堆在前 2 分钟 | 按全局最高分取 N 张 | 等分点只做**分区**，每段取段内最高分；既保覆盖又让分数决定时刻 |
| 18 | 选进纯红 UI 面板，正文里一堆"亮而无效"的帧 | 把 Laplacian 方差/曝光/熵**计入分数** | 质量指标只当**门限**（`Q_GATE=0.35`，过不了就跳过），绝不进 score；实测计分后清晰度 +24%，但选的全是过曝帧 |
| 19 | 5 处图注张冠李戴：NBI 说成白光、无蒂说成带蒂、计划说成已完成 | 换一套选帧后，按"时间相近（≤4 s）"复用旧图注 | **换图必须重写图注**；`figs_v*.json` 里 `reuse/NEW` 标记没人执行就会出事，取帧脚本只读前 3 个字段 |
| 20 | 想自动判断"NBI / 白光 / 透视" | 语义分类器 | CLIP 零样本把内镜判成"透视"置信度 0.9；4 类分类器按视频留一验证，整讲留出时准确率 **0.000**。结论：本类素材帧级语义筛选不可行，**人工过目不可省** |
| 21 | 双画面/合成画面整张取进来，一半是无效区 | 版式问题 | 用版式解决：第 2 讲双画面就 `crop=424:366:850:186`，08 合成画面裁固定区域，比训模型有效 |
| 22 | 画中画/透视帧裁完还是黑的 | 按灰度取"亮区"，选中了灰阶透视野 | 按**饱和度**取：内镜窗 sat 130–160，透视野 sat 0；否则会裁出术者摄像头 |
| 23 | 拿"黑边占比"当硬门槛，永远过不了 | 1280×720 画布 + 窄图必然留边 | 834 宽居中 → 34.84% 纯黑边，是几何必然；门限应看内容区或接受留边，别把百分比当质量指标 |

### 2.5b 用浏览器往 GitHub 提交网关代码（本轮实测）

| 症状 | 根因 | 处理 |
|---|---|---|
| 提交后 `sha256` 与本地不一致，但内容看着一样 | 本地文件被 Python 文本模式写成 **CRLF**，GitHub 上存的是 **LF** | 比较前两边都做 `replace(/\r\n/g,"\n")`；提交前用二进制模式 `open(p,"rb").read().replace(b"\r\n",b"\n")` 归一化。别用 `Get-FileHash` 直接比线上原始字节 |
| 编辑页 “Commit changes” 按钮是灰的（disabled） | 内容与线上完全一致（常见于“想重提一次”时） | 这是**好事**：说明上一次提交已生效。用 `?t=<时间戳>` 绕开 raw.githubusercontent 的 5 分钟缓存再核对，别重复提交 |
| 大文件粘贴后内容缺行 | 剪贴板/编辑器上限 | 16 KB 实测可用；再大就拆文件或改用 Git 推送（需要 token） |

### 2.6 审稿与协作

| # | 症状 | 根因 | 处理 |
|---|---|---|---|
| 24 | 审稿意见互相说服不了 | 只给结论 | 必须给可复现证据：行号 + 旁白原句 + 像素 MAE + `ffprobe` 输出 |
| 25 | 图注说的画面和图对不上 | 人工目测不可靠 | 机械核对：按图注时间 `ffmpeg -ss <t> -frames:v 1` 重截一帧，与 assets 里的图算 MAE，**< 6 视为同帧** |
| 26 | 子代理交回来的东西不能用 | 验收标准写虚了 | 验收写成数字（"近黑 <35%""0 缺失""14 PASS"），要求"做不到如实报"，允许有理由的偏离；回来**人要看产物**，不只看报告 |
| 27 | `assets/` 变成 `\x07ssets/`，链接全断 | PowerShell here-string 把反引号转义吃掉（`` `a `` → BEL） | 批量替换后必扫一遍 `[^\w]ssets?/`；能用 Python 写文件就别用 PowerShell 拼字符串 |
| 28 | 全局替换 `480p→720p` 把**目录名**也改了，死链 | 替换范围过大 | 只替换正文里确定的位置；改完必须逐个 `os.path.exists` 校验 |

### 2.7 交付

| # | 症状 | 根因 | 处理 |
|---|---|---|---|
| 29 | C 盘又满了 | 交付物和依赖都落在 C 盘 | 交付物 `E:\WorkBuddy交付\<主题>\`；依赖/模型/缓存 `E:\dsh\<项目>-env\`；C 盘只放脚本（< 1 MB） |
| 30 | 交付时才发现断链/缺图 | 没跑校验 | 三件套：`okf_check.py`、`check_links.py`、图数一致性，见 §3 |

## §3 校验门（可执行 + 期望输出）

| 门 | 命令 | 期望 |
|---|---|---|
| 取件探测 | 见 §1.1.2 的三行探针 | 三条全过才开始批量；结果抄进附录 |
| OKF 结构 | `python okf_check.py "$PROJ\okf" "$PROJ"` | `12 通过 / 0 失败` + `2 通过 / 0 失败`（两参数各跑一遍目录），退出码 0 |
| 图片链接 | `python check_links.py` | `images referenced: 86 missing: 0` |
| 图片尺寸 | 遍历 `okf\assets\**\*.jpg` 统计 `Image.open(p).size` | 分布只有 `{(1280, 720): 43}` |
| 图注逐字一致 | 讲义与 okf 同一段的图片行/图注行做字符串比对 | 9 讲 × N 图全部命中，0 差异 |
| 字幕流 | `ffprobe -v error -show_entries stream=codec_type,codec_name:stream_tags=language -of compact <mp4>` | 有 `subtitle/mov_text` 且 `language=zho` |
| 视频编码 | `ffprobe -v error -select_streams v:0 -show_entries stream=codec_name -of csv=p=0` | 与交付说明一致（本案例第 1 讲 `av1`，其余 `h264`） |
| 图数一致 | 讲义里的 `![图` 计数 = okf 里计数 = 43 | 相等，且每讲 6/6/5/6/4/4/4/4/4 |

把 1–3 写成一个 `selftest.py`，每次改完图或图注就整跑；数字对不上就别交付。

## §4 什么时候不该用它

- **单条短视频（< 2 分钟）**：取件+模型加载+拆 OKF 的固定成本远大于收益，直接手工整理更快。
- **需要自动语义筛选帧**：本类素材上 CLIP/分类器实测不可用（第 20 条），这个技能的输出质量取决于人有没有过目。
- **竖屏/手机手持素材**：1280×720 画布和固定 crop 布局都不成立，得先重做版式。
- **没有旁白/没有音轨**：整条流水线第 2 步就断了，只剩取帧一件事。
- **版权不允许转载**：讲义含逐段译文与截图，先确认授权；本案例是"仅供个人学习、不再分发"。

## §5 本案例基线（LLE26，2026-11-07，可用来对表）

| 项 | 数值 |
|---|---|
| 讲数 / 总时长 | 9 讲 / 48 分 53 秒 |
| 转写分段 / cue / 字幕条 | 572 / 383 / 335 |
| 关键帧 | 43 张（6/6/5/6/4/4/4/4/4），全部 1280×720 |
| 图片引用 | 86 条（讲义 + okf），0 缺失 |
| OKF 文件 | 12 个（9 概念 + index + log + 附录），14 PASS / 0 FAIL |
| 转写速率 | CPU int8 × 4 线程 ≈ 1.6× 实时 |
| 返工记录 | 07/fig3 近黑 86.9%→34.8%；06/fig4 54.5%→31.1%；02 讲 6 张 848×732→1280×720（近黑由 5.1% 变 38.1–38.6%，其中 34.84% 是留边） |
