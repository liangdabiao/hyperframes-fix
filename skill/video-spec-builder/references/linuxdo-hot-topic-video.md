---
name: linuxdo-hot-topic-video
description: Turn 1-N Chinese tech community hot topic posts (LinuxDO / V2EX / 即刻 / 微博热搜) into a fast-paced vertical short video (1080x1920, ~45s) in the style of a tech-news commentator on Douyin/TikTok. Covers source identification, the 5-question gate, the /faceless-explainer handoff, and the post-render sub-composition opacity bug that hides 80% of frames.
---

# LinuxDO / V2EX 热榜话题 → 竖屏短视频

把中文科技社区的"热榜帖子"做成一条有态度的科技评论短视频。**这是 video-spec-builder 的一种专用子类型**，不是 faceless-explainer 的简单复用——它的输入形状、对话节奏、视觉套路都是固定的。

> **关键差异 vs `/faceless-explainer`**：那个 skill 假设输入是"一篇文章/一段文字，请把它讲清楚"。这里输入是"几条热榜帖子，请评论它们"——立场更鲜明、节奏更快、必须每个 topic 一镜一镜头地拆，不能合并叙述。

## 触发条件

满足以下任一即走本 ref：

- 用户消息含 `linuxdo` / `LinuxDO` / `V2EX` / `v2ex` / `即刻` / `微博热搜` / `热榜` / `热搜`
- 用户粘一段 hot topic 标题（典型形态：「晴天霹雳…」「有没有…的工作」「笑死…」），并配概要
- 用户明确说"科技新闻风格"、"吐槽一下"、"做个快节奏评论视频"

如果用户只说"做个解释视频"但没给 topic → 走正常 0-1 流程，不走本 ref。

当用户直接说 做一个 linuxdo 热榜视频，那么就按以下：

`
挑选
    https://zenfeed.xyz/api/query?backendUrl=http%3A%2F%2Fzenfeed%3A1300&summarize=false ，   "source": "LinuxDO热榜",
    挑选 不是闲聊而是 重磅内容的 大概3-7条内容， 然后 制作一个类似报道 科技新闻 一样的快节奏
  竖屏短视频适合抖音，内容要干货有用，不是泛泛而谈，目标用户为 先进分子知识工作者等等， MiniMax 云端 TTS
  key在.env,不需要找外部视频素材，科技自媒体那种"有点调侃"的感觉
`

## Step 0: 5-question gate（**不要跳过**）

faceless-explainer 默认会做完整苏格拉底对话，但**热榜类需求太典型**，直接问这 5 个问题收口即可：

1. **平台 + 时长** — 抖音默认 9:16 / 45s 左右；B 站横屏选 16:9 / 90s
2. **topic 数量** — 通常 2 个（一条对比强，两条节奏刚好），3 个开始显赶，1 个勉强能撑 30s
3. **风格定调** — 三选一：
   - **冷面吐槽**（推荐）：克制叙述 + 关键处反问，适配"不可能三角"类话题
   - **情绪炸裂**：感叹号 + 排比 + 大量 highlight，适配"程序员被裁"类情绪话题
   - **知识科普**：先抛冲突再讲原因，适配"为什么 V8 比 V6 快"类解释型
4. **TTS 声线** — male-qn-qingse（冷静吐槽）/ male-qn-jingying（播音腔）/ 一段自定义参考音频
5. **配色** — 默认走 frame.md 的 neobrutalist（黑底 + 粉黄强调），用户不改即可

**这 5 个问题答完才能进 Step 1**。哪怕用户只丢一段话过来，也要先问这 5 个。

## Step 1: 解析每个 topic

每条 hot topic 提炼 4 个字段，写进 `video-spec.md` 的 `topic_sources` 区（**这是本类型的特殊字段，其他类型没有**）：

```yaml
topic_sources:
  - source: LinuxDO 热榜
    rank: #1
    title: "晴天霹雳，正刷着站，被通知要领大礼包了"
    core_conflict: "上班摸鱼被裁，但 N+1 赔偿引发羡慕/酸/讨论"
    keywords: [裁员赔偿, N+1, 提前毕业]
    emotion: shock
  - source: V2EX 热榜
    rank: #3
    title: "有没有稳定高薪压力小的工作"
    core_conflict: "稳定/高薪/不加班的职场不可能三角"
    keywords: [不可能三角, 考公, 996]
    emotion: sarcasm
```

**core_conflict 必填**——faceless-explainer 的 storyboard 拿不到这个就只能写流水账。

## Step 2: 调用 `/faceless-explainer`

在用户确认 5-question gate 后，**走 `/faceless-explainer` 的完整 Step 0-6 流程**，但在交接时必须显式声明三件事：

1. "这是热榜话题评论类，不是概念解释" → 强制走 `beat: surprise/recognition/sarcasm` 系列，不走 `beat: clarity`
2. `frame.md` 用 `blockframe` 预设（maximalist neobrutalist，黑底 + 粉/黄强调 + Inter 900 display）—— **不要选 soft-signal / velvet-standard 等"温和"预设**，那是给知识科普用的
3. 每帧的 reveal 必须**钉到 VO 节奏**——一个 topic 一镜，每个 topic 至少 2 镜（intro + 社区反应/评论）
4. MiniMax TTS API (see `external-tts.md` for full details).

## Step 3: 渲染后必做 Playwright 验帧（**硬步骤，不可跳**）

热榜视频的 HTML 结构特征：几乎 100% 是 sub-composition（每个 topic 单独一个 frame 文件，由 `data-composition-src` 引用）。**这里有一个反复踩的坑**：

### 🐛 Bug: sub-composition 内容全空

**症状**：渲染后 7/8 帧只看到黑底/白底，看不到文字。看 HTML 源码元素都在、属性都对、lint 0 errors、inspect 0 issues。

**根因**：子组件 HTML 用了 `style="...;opacity:0"` 作为初始隐藏 + GSAP `tl.to(el, {opacity:1, ...})` 作为入场动画。HyperFrames 渲染器在静态帧回退模式下**不会跑 GSAP timeline**——它只 snapshot DOM 当时的样子。所以 `opacity:0` 永远是 0，文字永远不可见。

**检测**（V3 视觉验证，必跑）：

```python
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    b = p.chromium.launch(
        executable_path='C:/Program Files/Google/Chrome/Application/chrome.exe',
        args=['--no-sandbox']
    )
    pg = b.new_page(viewport={"width": 1080, "height": 1920})
    pg.goto("http://localhost:8000/index.html")
    pg.wait_for_load_state("networkidle")
    for sel in [".kinetic-line-1", ".hero-quote", ".topic-1-line"]:  # 每帧的关键文案选择器
        el = pg.locator(sel).first
        if el.count() == 0: continue
        opacity = pg.evaluate(f"getComputedStyle(document.querySelector('{sel}')).opacity")
        assert float(opacity) > 0.5, f"❌ {sel} opacity={opacity} — sub-composition 静默失败"
    pg.screenshot(path="check.png")
    b.close()
```

**修复**（批量脚本，存到项目根 `fix_opacity.py`）：

```python
# 1. 把每个 frame 文件里的 inline style="...;opacity:0" 全部删掉（元素默认可见）
# 2. 把 GSAP 的 tl.to(el, {opacity:1}) 改成 tl.fromTo(el, {opacity:0}, {opacity:1})
#    这样 timeline 跑时是动画、不跑时元素也是 opacity:1（可见）
```

**预防规则**（写进 video-spec.md 的"硬约束"区）：

> 子组件的入场动画**禁止**用 CSS `opacity:0` 初始 + GSAP `tl.to({opacity:1})`。
> 必须用 `tl.fromTo({opacity:0}, {opacity:1})`，让 opacity:1 成为 DOM 默认状态。

## Step 4: V4 肉眼抽帧复检

V3 通过后还要**逐帧肉眼抽图**（ffmpeg 抽 8 张 + Read 看图），确认：

- 文字不溢出 1080×1920 边界
- 多行文案没挤成一坨
- 强调色（粉/黄）出现在它该出现的地方
- caption 字幕（如果启用）不和画面 hero text 撞位置

V3 + V4 任何一步不过，**不要 render，直接回去改 HTML**。

## Step 5: 常见 bug 速查

| 症状 | 根因 | 修复 |
|---|---|---|
| 8 帧 7 帧空白 | sub-composition opacity:0（见 Step 3） | 批量 fix 脚本 |
| 帧切换时画面闪一下 | `transition_in: cut` 写成 `crossfade` 但前帧没 hold 完 | 改回 `cut`，或加 `data-hold-end` |
| VO 节奏和画面 reveal 对不上 | storyboard 的 Scene 时间码写死了秒数，没跟着 VO 词时间戳 | 跑 `audio.mjs sync-durations` 后再写 reveal |
| 字体变方块 | 用了 `font-family: 'Noto Sans SC'` 但 `@import` 没加粗体权重 | 改成 `'Noto Sans SC', sans-serif` 兜底 + 加 `wght@900` |
| 渲染 OOM（竖屏 1280×1920） | 多 worker 吃满内存 | 渲染时加 `--workers 1`（见 memory `vertical_render_windows`） |

## 交付物清单

```
videos/<project>/
├── video-spec.md          # 含 topic_sources 字段（必填）
├── SCRIPT.md              # 8 段 VO，每段标 Delivery 语气
├── STORYBOARD.md          # 8 镜，每镜钉到 VO 时间戳
├── frame.md               # 强制 blockframe 预设
├── compositions/frames/   # 01-hook.html ... 08-cta.html
├── index.html             # 主组件，8 个 data-composition-src
├── fix_opacity.py         # 渲染后跑一次，批量修子组件 opacity bug
├── check_frames/          # V4 抽帧存放处
└── renders/video.mp4      # 最终交付
```

## 完成标准

- [ ] 5-question gate 全部答完
- [ ] `topic_sources` 写齐（每条 source/rank/title/core_conflict/keywords/emotion）
- [ ] frame.md 是 blockframe 预设（**不是 soft-signal**）
- [ ] V3 Playwright opacity 检查全过
- [ ] V4 8 帧肉眼抽帧全过
- [ ] 字幕 + VO 节奏对齐（无错位 / 无重影）
- [ ] `renders/video.mp4` 文件大小 > 500KB，duration 误差 ≤ 0.5s
