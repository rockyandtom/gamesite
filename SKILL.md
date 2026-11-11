---
name: 游戏站开发与部署
description: 游戏热词/内容站从素材到上线的完整方法论。覆盖:技术栈选型(Astro 静态站 + Cloudflare Pages)、上游素材红线、建站规范、Cloudflare 部署与自定义域名绑定、on-page SEO、GSC/GA4 接入、以及一套实战踩坑清单。适用于任何需要快速搭建 SEO 友好内容站并部署到 Cloudflare Pages 的项目(尤其是生财航海类游戏热词站)。
agent_created: true
---

# 游戏站开发与部署方法论

本 skill 沉淀自一个完整实战项目(bombanana 游戏热词站:上游 OpenClaw 备料 → 本 skill 负责建站/部署/SEO)。它描述的是**开发部署方的职责边界内"怎么做"**,不覆盖选题/选词/变现战略(那由生财手册与用户决策定)。

> **上站前先看 `站点上线总纲`**：它给的是「买域名 → CF zone → 建站 → 埋 GA4 → Pages 部署 → 绑域名 → GSC → 索引 → 验收登记」的顺序、每步用哪个工具、凭证文件在哪、以及一键体检脚本 `doctor.sh`。
> 本 skill 负责其中**建站与 SEO 细节的方法论和踩坑**，不要从这里开始规划上站顺序。

> **SEO 判断的最终依据是 skill `seo-dev-spec`**（编号 Z/R/U/I/T/K/P/C/D/M）。本 skill 是"怎么做"的方法论与实战记录，两者冲突时以 `seo-dev-spec` 的编号规则为准。
> 差异补充见 §3.2（2026-10 补的 8 条，覆盖图片/外链/h1 层级/预览环境等此前没写的部分）。

## 0. 分工边界(先对齐再动手)

典型的"生财航海路线 A"游戏站,职责分三段:
- **上游·素材工厂(如 OpenClaw / 云端 agent)**:选词、挖词、调研,产出"一页一文件"的原文素材仓库(Google 攻略 + YouTube 字幕 + 官方页面),只产内容不建站。
- **中游·建站(本 skill 核心)**:用素材生成页面 + 搭工程。
- **下游·部署 + 验收(本 skill 核心)**:Cloudflare Pages 部署、自定义域名、GSC/GA4、补页、on-page SEO。

⚠️ 本人只做中游+下游。素材仓库是"事实来源",不是我编的——任何事实必须能回溯到素材原文。

## 1. 技术栈决策

- **首选 Astro 静态站**:默认零运行时 JS,SEO 友好(纯 HTML 输出),构建产物 `dist/` 可直接被 Cloudflare Pages 托管。
  - 适用:内容站、wiki、攻略站。⛔ 不适用:重交互 SPA(那种选 Next.js/Vite)。
- **部署:Cloudflare Pages**(direct upload,不用 Git 集成也能传)。
- **凭据:用静态 API Token 文件,不走 OAuth MCP**(见 §4)。

## 2. 素材红线(上游收素材时用)
> ⚠️ **适用范围教训(2026-09-17)**:这套红线原本约束的是**上游素材收集环节**(原文留存、标来源、不编造、2 源交叉)。我曾把它自我引申为"我写页面时事实只能引用素材库、无源不写"并写进项目约定——**这是越界**,用户从未要求。代价:因素材库零命中而拒改了明显正确且可一手核验的事实(demo 已下架、平台仅 PC),导致线上挂着已知错误答案却"等素材"。

建站时生成的内容必须遵守:
1. 事实**只来自真实源**(官方页面 / Google 攻略 / YouTube 字幕原文),不凭记忆编。
2. **≥2 源交叉验证**(如 Steam 官方 + 社区攻略一致)才写进正文;孤证要标注"社区整理,待验证"。
3. **原文留存**:素材仓库保留原文,页面可转述但关键事实(数值/规则)要能回溯。
4. 每页底部加 **Source note**,区分"官方"与"社区整理"来源。
5. **外部报告/建议里的"事实"不等于可用事实**。收到任何外部 SEO 报告声称"你这里写错了"时,**先核实**再决定改不改。但"核实"的边界是**主动去找源**,不是"素材库里没有就不改"。为凑内容量写无源答案是最贵的一种自杀;**因懒得查源而留着已知错误答案,是第二种。**

### 2.0 内容事实分级(建站写页面时用这套,替代"无源不写")
| 级别 | 内容 | 规则 |
|------|------|------|
| **A 类** | 一手硬事实:价格、发售日、平台、系统需求、成就数、demo 有无、语言数、客服/社区链接 | **查了就写**,标注来源 + 核验日期,**不等素材库** |
| **B 类** | 操作性/社区内容:模块解法、角色技巧、玩法流程 | 沿用素材库 ≥2 源交叉验证,无源不写 |
| **C 类** | 无源推测:未宣布的平台计划、"即将上线"传闻 | 不写,或明写"官方未宣布" |

外部报告给的建议按这三级判定:**A 类直接查证后改**(报告的措辞可能带无源细节,只取可核部分),**B 类查素材库**,**C 类拒**。删掉的应该是**具体措辞**,不是整条建议。

### 2.1 实时事实核验通道(Steam appdetails API)—— A 类事实首选
写价格 / 平台 / demo 存废这类**会随时间变化**的事实时,优先用官方 API 核,不要只信素材库快照:
```
curl -s --noproxy '*' "https://store.steampowered.com/api/appdetails?appids=<appid>&l=en&cc=us"
```
一次能拿到的字段(全部可写进页面):
- `name` 官方拼写(拼写字/变体 FAQ 的铁证)
- `price_overview.final_formatted` 价格
- `release_date.date` / `coming_soon`
- `platforms` → **windows / mac / linux 三个都要看**。只看 linux=false 会误判成"仅 Windows",本项目 mac=true 差点写错
- `demos` → **null = 当前无 demo**(比"素材里说有 demo"可靠)
- `achievements.total` 成就数
- `supported_languages` 语言数(逗号分隔,直接数)
- `website` 官网、`developers` / `publishers`
- `pc_requirements.minimum` / `.recommended` 系统需求原文
- `categories` 是否支持 Family Sharing / 手柄 / 远程同乐等
- 无需登录、无反爬;比抓商店 HTML 稳(HTML 常返回空壳页)。
- ⚠️ **实时 API 与素材库冲突时,以实时官方 API 为准**,并在页面标注核验日期(如 "checked on Steam, Sep 17 2026")。
- 这类"带日期的事实陈述"本身就是**新鲜度信号**,是低权重新站对抗高权重老站的关键差异化。

### 2.2 跨页事实一致性(改一处必须同步全部)——最容易犯的错

**同一条游戏事实(平台 / 价格 / 人数 / demo 存废 / 发售日)通常同时存在于 4~5 个地方**。只改其中一处 = 站内自相矛盾,用户在一页看到两个答案,信任直接崩;而错误答案还会同时进 FAQPage schema,被 AI Overview 读走。

**改任何一条事实,按这个清单全站同步:**
1. 首页数据文件(`src/data/homepage.json` 之类)的 FAQ 数组 —— 首页 schema 通常由它生成,改正文即改 schema。
2. FAQ 页的 **quick answers 数组 + FAQs 数组 + 正文章节**(同一个页面里可能三处都有)。
3. 发售/价格专题页(quick answer 表 + 正文 H2 + FAQ 数组 + meta description)。
4. 首页 `VideoGame` schema:`operatingSystem` / `numberOfPlayers` / `offers`。
5. **改完 grep 全站搜旧说法关键词**(如 "only way to play"、"no two-player"),确认零残留再部署。

### 2.3 批量同构页面(逐关页 / 逐角色页 / 逐物品页)怎么上

游戏站最容易想到的扩页方式(30 关各一页、每个角色一页)是**真机会,也是最容易翻车的地方**。三条硬规则:

1. **先查"每页有没有独占数据"再决定做不做**。逐关页能成立,是因为素材里有**逐关独占的硬数据**(时限 / 模块组合 / 难度标签 / 敌人配置)。如果只有一个列表项加一句套话,那就是 doorway page,别做。
2. **每页必须有独占内容,不能只靠模板变量**。最低配置:该条目数据表 + 它在本站体系里的定位(属于哪个阶段/类别) + 指向相关深度页的内链 + 该条目的专属要点 + 2–3 条专属 FAQ。目标 350 词以上。**光靠"把同一个表拆成 30 个 URL"必被判薄内容。**
3. **分批上,不要一次全上**。新站一次性新增几十个同构页,容易被整体压低。**每批 10 页、间隔一周**,看收录与曝光再决定下一批。第一批就是试验田:收得进、有曝光,再复制模式。

### 2.4 工具类页面:没有规则数据就别做 solver

solver / 计算器这类"交易意图"页转化最好,但**本质是查表,先有映射规则才谈得上做**。开工前先确认素材里有没有该模块的**完整映射表**(编码表、符号表、序列规则)。

- 素材里如果写着"具体规则见某某 Guide"而**那张表没被抓下来**,就**不能做** —— 用通用规则顶替游戏内的自定义编码,会给出错误答案,比没有更糟。
- ⚠️ **但别把这条变成教条**:禁的是「**无源**时靠通用规则推断」。若**≥2 源交叉印证该模块用的就是标准编码**(如 Morse 五信号数字表被两个独立源一致确认),则前提消失、可以写。**判断依据是"有没有源",不是"是不是通用规则"**。(2026-09-18 实例:Morse 表两源一致 = 标准国际摩斯,可写。)
- 缺规则时的替代:做**零规则依赖**的工具(速查表、流程检查器、模板生成器),或先让上游补采规则素材。
- 视觉寻路类模块(迷宫、连线)天然不适合 solver,不要硬套形态。

**验收上游交付的规则表**(拿到文件后必做,别照单全收):
1. **机械自洽性检查** —— 用脚本查:格子总数对不对、是否有重复项、取值域是否越界(如音符表只应出现 CDEFGAB)、分布是否失衡。**编造的表通常会在这些维度露馅**(重复、越界、分布畸形)。这能证伪,不能证实。
2. **≥2 源?** —— 单源表要明标"single source, not reproduced in-game",页面可展示、交互查表要带提示。
3. **站内交叉要小心** —— 同一游戏的不同模块可能用**不同取值范围**(实例:Direction 模块是 Braille 1–9×4 色,Piano 是 Braille 1–7×4 色),**数字对不上不代表矛盾,是不同模块不同表**。只能互证"机制范式",不能互证"格子内容"。
4. **缺时长/毫秒类参数通常不阻塞 solver** —— 查表型 solver 的输入由玩家自己看/听提供,页面写"以游戏内节奏为准"即可,**绝不编数值**。
5. 派发给上游的任务书要写**数据契约**(逐字段列出)+ 验收标准"能否手工算出任意输入的答案",否则收回来一堆描述文字还是用不了。
6. **★ 反向校验站内已有 solver**(2026-09-18 新增,价值极高):上游若攻破了新源(尤其 403 站),**拿新源的查表逐项比对站内已在跑的 solver 数据表**。实例:destructoid 攻破后比对结果 Cable 8/8、Direction 36/36、Calculator 8/8 全一致 → 站内规则被**独立第三源 100% 验证**。这一步能同时做两件事:① 证伪/证实站内数据 ② 给页面加"三源验证"的可信度背书。
7. **警惕源内自相矛盾**:同一篇文章的**表格与其例子打架**时(实例:destructoid 的 Calculator 例子 "69 odd + Green → 9" 与它自己的表 Green×Odd=2 冲突),**信表格、弃例子**——表格通常是结构化搬运、例子是作者手写笔误。务必在验收记录里写明"该例子不采用",避免下游再抄错。

**403 站点的通用解法**(Cloudflare 拦截别轻易放弃):
Cloudflare 挡住的攻略站往往是最有价值的(说明内容稀缺)。用 **Tavily `extract` 接口**绕过:
```bash
curl -s -X POST https://api.tavily.com/extract \
  -H "Content-Type: application/json" \
  -d '{"urls":["https://被403的URL"]}'
```
2026-09-18 实例:destructoid 的 `bombanana-walkthrough-all-level-solutions` 直接 curl 403,用该接口拿到 10405 字符全文(L1–L5 解法 + 三张查表)。**给上游的任务书里要写明这个绕过手段**,否则对方会直接报"403 放弃"。

**抓官方社区链接(Discord 等)**要走商店 HTML,且必须带成熟内容 cookie,否则 curl 超时或拿到空壳:
```bash
curl -s -m 45 --noproxy '*' -A "Mozilla/5.0 ... Chrome/120" \
  -b "birthtime=283996801; mature_content=1; wants_mature_content=1" \
  "https://store.steampowered.com/app/<appid>/<slug>/?l=english" -o steam.html
grep -oiE 'https?://[a-z0-9./-]*discord[a-z0-9./-]*' steam.html | sort -u
```
官方 Discord 挂在商店页上,属 A 类一手源,拿到就能写(这是"找队友"类高频长尾词的答案)。

### 2.5 素材仓库同步:一律走 SSH,HTTPS 是死路

上游(OpenClaw)把素材推 GitHub 私有库,本机取用**必须用 SSH**:
```bash
git remote set-url origin git@github.com:<owner>/<repo>.git
git pull --ff-only
```
**HTTPS 三条路全不通,别浪费时间试**:
1. `git pull` 走 github.com:443 —— **超时**(沙箱不放行该域名)。
2. `raw.githubusercontent.com` 匿名访问 —— **一律 404**(私有库)。判断技巧:拿一个**确定存在**的文件做对照,若它同样 404,就是私有/未授权而非路径错。
3. GitHub 连接器 MCP —— 常未授权该仓库(404)。

排查:若 pull 又莫名走回 HTTPS,查 `git config --global --get-regexp url\.` 是否有 `insteadof` 重写规则把 SSH 改写成 HTTPS。

上游会在仓库内维护 `project-status.md`,**最新的工具/基础设施章节(SSH key 路径、推送命令、踩坑)就写在里面** —— 拿不到文件时先读它,别自己瞎猜凭据。

## 3. 建站规范(每页统一)

工程结构:
```
src/
  layouts/Base.astro      # 全站壳:SEO meta / canonical / OG / JSON-LD 面包屑 / GA4
  components/Nav.astro    # 导航(当前页 aria-current)
  components/Footer.astro
  components/AdSlot.astro # 广告位,先占位 ca-pub-XXXXXXXXXXXXXXXX
  pages/*.astro           # 每页一文件
  data/*.json             # 首页等结构化内容
  styles/global.css
public/                   # favicon / og 图 / robots.txt / sitemap.xml
```
页面 frontmatter 必带:`title` / `description` / `canonical` / `og` / `jsonLd`(面包屑)。

- **meta description 长度规范:140–160 字符(实战刻度,以后按此执行)**。
  - <140 浪费 SERP 展示空间、可能无法完整表达卖点;≤160 是搜索引擎普遍截断线,多数 SEO 审计工具会把 **>160** 标记为超标(本项目实测:switch 页 184 字符被用户工具标红,压到 150 字符后达标)。
  - 生成 / 改写 description 时,用 `content.length` 校验落在区间内再提交;允许微调但**优先保事实完整**(见 §2 素材红线),不为凑字数删掉关键玩法表述。
  - 标题 `title` 同步控在 **50–60 字符** 内(超长会被截断)。
- 导航 `href` 与页面真实路由**统一带尾斜杠**(`/levels/` 而非 `/levels`),否则 `Astro.url.pathname` 对不上,当前页高亮(`aria-current`)失效,且点击会触发 308 重定向。
  - 注意:`trailingSlash: 'ignore'` 下无尾斜杠链接也能访问(不报 404),但会让 canonical 产出 `/levels` 与 `/levels/` 两个变体,隐患隐蔽——**必须扫 HTML 而不是靠"能打开"来判断**。
- **首页 + Footer 必须是完整内链枢纽(极易漏)**:首页是权重最高、抓取最勤的页面,必须链到**全部**内容页,不能只有 2–4 个主入口卡片。
  - 本项目实测教训:10 个页面里 **6 个模块页首页零内链**,只能靠 sitemap 与内页互链被发现,抓取深度被白白拉深。
  - **每新增一页,同步检查首页与 Footer 是否链接到它** —— 这是最容易漏的一步。首页用「模块索引卡区块」、Footer 用「Guides + Modules 两列」承接。
  - 验收(构建后必跑):遍历 `dist/**/*.html` 抽 `href`,过滤掉资源后缀(`.svg/.css/.js/.xml/.png` 等)后,**去重站内页面链接数应 = 页面总数**,且全部带尾斜杠。一次性脚本即可,别靠肉眼。

### 3.1 内页 FAQ 区块(内容加厚 + 长尾词的标准动作)

素材被页面"用尽"后,不要停产等待上游,优先**加深已有页**。标准做法是给每个内页加 FAQ 区块:

1. **建可复用组件** `src/components/FAQ.astro`:
   - 用**原生 `<details>/<summary>`** 折叠(零 JS、键盘可达、无障碍友好),不要用自己写的 JS 折叠。
   - props:`items: {q,a}[]` + `schema?: boolean`。
   - `schema` 为 true 时输出 **FAQPage JSON-LD**(`mainEntity` 是 Question/AcceptedAnswer 数组)。**每页只输出一次**。
   - 脚本用 `<script type="application/ld+json" set:html={JSON.stringify(...)} is:inline />`。
2. **每页 4–6 问**,问题用玩家真实会搜的口语长尾句(How many… / Is… / What should I do if…)。
3. **答案必须来自素材**(受 §2 红线约束),且**与页面可见文本一致**——结构化数据里的答案若页面上找不到,属作弊,会被判无效甚至处罚。
4. 加厚内容时优先挖素材里**尚未使用**的部分:完整示例、常见错误/不完整呼叫、失败诊断表、阶段划分、命名约定。这些比重复规则表更有增量。
5. 页脚内链补齐(相关模块页 + `/tools/` 工具页),形成内页互联。

验收:构建后 `dist` 里 grep `FAQPage` 应为 1、`<details` 数量 = 问题数;description/title 长度仍须落在 §3 的区间。

> **⚠️ 时效性(2026-05 起,务必据此设预期)**:Google 已于 **2026-05-07 彻底下线 FAQ 富结果**(2023-08 起已限制为政府/医疗站,2026-05 全部移除;2026-06 移除 GSC 报告与 Rich Results Test 支持)。**不要再以"拿 SERP 折叠问答增量"为由加 FAQ**。
> - FAQPage schema **本身未废弃**、留着不报错不降权,Google 仍会读取用于**理解页面**,Bing / AI 检索系统也可能消费——所以**已加的不用删**,但别指望富结果。
> - 加 FAQ 的理由只能是**可见文本本身对用户有价值 + 长尾词覆盖**。若某页 FAQ 纯为 SEO 而写、拿掉对读者无损,就不要加。
> - **首页(导航型页面)不加 FAQ**:首页承接品牌词/导航意图,Google 给 sitelinks;且首页放问答会与内页 FAQ 自相残杀。首页该做的是**内链枢纽**(必须链到全部内容页),而非堆问答。

### 3.2 与 `seo-dev-spec` 对齐的 8 条补充(2026-10)

上面 3.0/3.1 是实战沉淀,这一节是拿规范逐条比对后**补上的缺口**——原来没写,不是写错了。每条后面是规范编号。

1. **一页一词要落成文件**:建 `src/data/keyword-map.csv`(字段 slug / 目标词 / 搜索意图 / 模板类型 / 父页 / 状态),**先加行再写页面**。两页打同一个词时,靠记忆和评审都发现不了,只有把词写在同一个文件里才查得出来。§2.3 的"每批 10 页"也按这份表登记。[K-1 / K-2]
2. **一个页面只有一个 h1**,且 h1 不能和 title 一字不差(title 带后缀,h1 换个说法覆盖变体);h2 到 h3 不许跳级;**页尾 CTA 区不许再放第二个 h1**——那是现成模板里最常见的一个错。[T-2 / T-7]
3. **图片三件事**:必须 `alt`(至少一张图含目标词)、必须写真实 `width`/`height`(否则 CLS 超标)、**首屏/LCP 图不许 `loading="lazy"`**(要 `fetchpriority="high"`),lazy 只给首屏以下。多尺寸走 `srcset` / `<picture>`,格式用 WebP。注意「所有图片都加 lazy」是流传最广的错误建议之一。[C-3 / T-4]
4. **外链 rel 按性质标**:付费 / 赞助 / 返佣 → `rel="sponsored"`;竞品、论坛 UGC、不可担保内容 → `rel="nofollow"`;`target="_blank"` 一律补 `rel="noopener"`;**正文引用官方资料 / 权威媒体不要带 nofollow**(那是 E-E-A-T 信号);**站内链接不加任何 rel**。[I-12]
5. **`viewport` 与 `html lang` 必须对**:缺 `width=device-width, initial-scale=1` 移动端直接不合格;`html lang` 与站点语言不一致是换站时的高发事故。点击目标 ≥48×48px,全站不许横向滚动。[C-5]
6. **性能三条阈值以移动端 field data 为准**:LCP < 2.5s、**INP < 200ms**(不是 FID,FID 2024-03 已废)、CLS < 0.1。广告位必须预留固定高度,首屏不放广告。[C-1 / C-3]
7. **预览域名必须 noindex**:`*.pages.dev` 每次部署都能公开访问且无 noindex,被抓到就是整站重复内容 + 权重分裂,清理要几个月。做法见 `游戏攻略站模板` 的 `docs/02-上站SOP.md` 第 6 步第四个坑(Cloudflare Access 或 Pages Functions 中间件打 `X-Robots-Tag`)。[D-4]
8. **正文 < 300 词不是"略少",是 soft 404 画像**,直接判不合格(§2.3 的 350 词门槛更高,继续用 350)。[Z-1]

> 这 8 条已全部编码进 `游戏攻略站模板` 的 `scripts/audit.mjs`(21 项检查),新站直接用模板就不用逐条手查。

## 4. Cloudflare 部署

### 4.1 永久连通(关键坑)
- **不要指望 OAuth 型 MCP 连接器**:本机 WorkBuddy 发起 OAuth 的回调地址既非 HTTPS 也非 localhost,Cloudflare 会报 `invalid_request / Redirect URI must use HTTPS or a local loopback address`,此环境无解。把远程 MCP 连接器 `disabled:true` 即可。
- **正解:静态 token 存家目录 env 文件** `~/.workbuddy/cloudflare.env`(权限 600,不进 git):
  ```
  CLOUDFLARE_API_TOKEN=cfat_xxxx
  CLOUDFLARE_ACCOUNT_ID=xxxx
  ```
  每会话 `set -a; . ~/.workbuddy/cloudflare.env; set +a` 直连。
- **token 权限最小集**:`Account → Cloudflare Pages → Edit` + `Zone → DNS → Edit`(绑自定义域名建 CNAME 必需)。
- 验证连通:`GET accounts/{id}/pages/projects` 返回 200 即活体验证通过(`/user/tokens/verify` 对账号级 token 会误报 401,忽略)。

### 4.2 创建项目 + 部署
- API 建 Pages 项目:`POST accounts/{id}/pages/projects`,body `{"name":"项目名","production_branch":"production"}`(direct 上传类型)。
- 安装 wrangler:受管 node workspace 的 npm 可能把包装丢到别处导致 `node_modules` 空,改在 **site 工程内** `npm install wrangler --no-save`(已被 .gitignore,不污染仓库)。
- 部署:`wrangler pages deploy dist --project-name 项目名 --branch production --commit-dirty`。
- 验证:遍历所有路由 `curl -w "%{http_code}"` 应全 200。

### 4.3 绑定自定义域名(必踩的坑)
- 用户先在域名注册商把 NS 改到 Cloudflare(本人操作不了)。zone 状态变 `active` 才算 NS 生效。
- 用 API 加域名:`POST accounts/{id}/pages/projects/{项目}/domains` `{"name":"域名"}`(走 Pages 权限,易成功,但状态停在 `pending`)。
- **关键**:Cloudflare Pages 用 **API** 加域名时**不会自动建 DNS 记录**(只有控制台 UI 会)。必须手动建 CNAME:
  `POST zones/{zone}/dns_records` `{"type":"CNAME","name":"域名","content":"项目名-xxx.pages.dev","proxied":true}` —— **这一步需要 `Zone:DNS:Edit`**,缺权限会 403(code 10000)。
- 建好 CNAME 后域名转 `active`,Universal SSL 自动签发(等 1–5 分钟,https 能 200 即证书生效,Pages API 的 `ssl_status` 字段可能不回填,以实际 https 200 为准)。

⚠️ **apex 根域 vs 子域的验证差异(实测教训,极重要)**:
- **apex 根域**(如 `bombananaguide.xyz`,主站)经 API 加域名 + 手动建 CNAME → 能转 `active`,上述流程成立。根域走 CNAME 扁平化,校验另走一路。
- **子域**(如 `tools.bombananaguide.xyz`)经 API 加域名后,**手动建的 CNAME 不被校验识别**,Pages 持续报 `CNAME record not set` 卡 `pending`,且 Pages 也不会自动建 CNAME(受管 zone 本应自动建"托管"记录,但 API 路径不触发)。已穷尽验证:proxied / grey-cloud / 删手动记录后重加域名期待自动建(等 5 分钟仍无)/ 多次重置校验,全部失败。
- **子域激活必须用户在 Cloudflare Dashboard 操作一次**:Workers & Pages → 项目 → Custom domains → 该子域 → 确认/激活(UI 才会建托管 CNAME 并完成校验)。CNAME 提前按正确目标(`项目名.pages.dev`)放好,点完几分钟即 `active`。wrangler 没有管理自定义域名的子命令,此路不通。

### 4.4 www 子域 + 301 重定向(必做,漏了会让 GSC sitemap 全盘失败)

🔴 **这是上线必做项,不是可选项**。原因:自建站默认只加 apex 主域,DNS 里**没有 www 记录** → `www.域名` 解析失败(HTTP 530 / CF error 1016)。后果链:

1. 用户在 GSC 提交 sitemap 时若填了带 www 的地址,或 GSC 属性是 www 的网址前缀属性 → Googlebot 永远抓到 530 错误页 → **"Sitemap could not be read" 反复失败,且怎么改 sitemap 文件都没用**。
2. 外链、社媒分享走 www 也全丢。

**标准做三步(我有 DNS:Edit + Pages:Edit 权限,可全自动)**:

1. **Pages 项目加 www 自定义域名**:`POST accounts/{id}/pages/projects/{项目}/domains` `{"name":"www.域名"}`。状态 `initializing` → `active` 约 1–2 分钟(等 Universal SSL 签发,以 https 200 为准)。
2. **DNS 加 www 记录**:`POST zones/{zone}/dns_records` `{"type":"CNAME","name":"www","content":"项目名-xxx.pages.dev","ttl":1,"proxied":true}`。
3. **建 www → apex 的 301 重定向**(方向必须与 canonical 一致,即 www 跳主域)。

**重定向的两种界面(等价,看用户控制台是哪个版本)**:
- **新版**:Rules → Redirect Rules → Create rule → 动作选 **Dynamic redirect**;表达式 `http.host eq "www.域名"`,URL `https://域名${http.request.uri.path}`,状态 301,勾 Preserve query string。
- **旧版 Single Redirects**(只有 Wildcard pattern,没有 Dynamic redirect):Request URL = `https://www.域名/*`,Target URL = `https://域名/${1}`,状态 301,勾 Preserve query string。`${1}` 承接通配路径。
- ⚠️ **API 建 rulesets 需要 `Zone:Rulesets:Edit` 权限**,常备 token 一般没有 → 这段交给用户在 Dashboard 点 3 下,或让他另签带该权限的 token。

**验收(必须逐条跑)**:
```
www.域名/                  → 301 → https://域名/
www.域名/sitemap.xml       → 301 → https://域名/sitemap.xml
www.域名/某内页/            → 301 且路径不变
www.域名/某页/?a=1         → 301 且 query string 保留   ← 最容易漏,务必测
域名 全部路由               → 仍 200(重定向不能误伤主域)
```

⚠️ **验证 www 时一定要绕开本机两重干扰**:
1. 本机 curl 走代理(`127.0.0.1:60657`)→ 加 `--noproxy '*'`,否则 502 是代理返回的,会被误判成源站故障。
2. 本地 DNS 有**负缓存**(之前解析失败过)→ 用 `dig @1.1.1.1` 或 `curl --resolve www.域名:443:IP` 验证,别信本地解析结果。

### 4.5 建 GSC 属性:一律用「网域(Domain property)」

- **不要**用「网址前缀(URL prefix)」属性。网域属性靠 DNS TXT 验证,一次覆盖 apex、www、http、https 全部子域与协议,**从此不会再遇到 www / 非 www 分家的坑**。
- 验证是否已完成:先查 DNS TXT 里有没有 `google-site-verification=...`,有就说明已验证(页面上本就不该有 meta 标签)。审计工具报"未放置验证标签"通常是**误判**——它只解析 HTML、不查 DNS。

## 5. SEO 收尾(上线后立刻做)

### 5.1 sitemap(最高频坑)

**排障顺序(照这个走,能省几小时)**:
1. **先测 www 子域** —— 这是"could not be read / couldn't fetch"最高频的真凶,且和 sitemap 文件内容毫无关系。见 §4.4。
2. **标签顺序必须合规**:`loc → lastmod → changefreq → priority`。XML 能解析 ≠ 符合官方 XSD;顺序错了 GSC 严格校验会失败。**补字段时不要往 `<url>` 块尾追加。**
3. **所有 `<loc>` 必须带尾斜杠**,与页面 canonical / 真实路由一致。无尾斜杠会 308 重定向,GSC 解析到重定向就报 "could not be read"。
4. sitemap 内 URL 必须全 200,且 canonical 与 sitemap 写法完全一致(无大小写/尾斜杠差异)。
5. 用 Googlebot UA 实测 + **IPv6 也要测**(Googlebot 优先走 IPv6):`curl -6`。
6. **对照实验:把同一 sitemap 提交 Bing Webmaster**。Bing 几小时出结果——Bing 成功而 GSC 失败 = 站点侧完全无罪,直接锁定是 GSC 属性侧 / 新域名抓取队列问题,**停止折腾文件,改走 URL Inspection 逐页请求索引**。

- `robots.txt` 写 `Sitemap: https://域名/sitemap.xml` + `User-agent: * Allow: /`,不要拦 Googlebot。
- sitemap 是手写静态文件放 `public/`,补页时**必须同步加 `<url>`**。本项目实际有**三份**:`public/sitemap.xml`、`public/sitemap/sitemap.xml`、`public/sitemap.txt` —— 三份都要补,GSC 读的是带下级目录那份,详见 §5.5。
  - ⚠️ **更好的做法是让脚本从页面树生成这三份**(`seo-dev-spec` U-10:不允许手写、不允许手工导出)。`游戏攻略站模板` 的 `scripts/make-sitemap.mjs` 就是干这个的,`lastmod` 取源文件 mtime 而不是"今天"。三份手维护的本质问题是**漏页**——漏掉的往往正是刚写的那一页。没脚本可复用时才手工补,并且每次补完都要跑一遍"页面数 = sitemap 条目数"的对账。
- **GSC 报错文字的判读**:`Type=Unknown` + `Last read` 为空 = Google **连文件都没拿到**(网络层问题,查 www / 拦截);`Type` 有值但报错 = 拿到了解析不了(查 XML 格式)。

### 5.2 GA4 注入(Astro 坑)
- `Base.astro` 的 `<head>` 注入 gtag。**不能用 `is:inline` 脚本体里的 `{GA4_ID}` 插值**——Astro 的 inline 脚本体不做 frontmatter 插值,会原样输出字面量 `{GA4_ID}`,等于接了个假 ID、数据不进账户。
- **正确做法**:`<script is:inline define:vars={{ GA4_ID }}>` + 脚本体内用 `GA4_ID` 变量。

### 5.3 提交 GSC
- 域名用 DNS 验证最省事(NS 已改到 CF,GSC 直接读到)。
- 提交 sitemap:Indexing → Sitemaps → 填 `https://域名/sitemap.xml` → Submit。
- **新 GSC 界面对 "Couldn't fetch" 状态的条目不显示 Remove**,只能等它自己过期或被新提交覆盖。多数情况是 GSC 还没轮到抓,**等 10 分钟~24 小时会自动变 Success**;若久红,用 URL Inspection → Test live URL 看 Google 实际能否抓到。
- 浏览器打开 sitemap 显示 "does not appear to have any style information" 是**正常**的(所有 sitemap 都这样),不影响读取。
- ⚠️ **新版 GSC 界面术语变了**:左侧没有 "Search Results",叫 **Performance**;索引报告在 **Indexing → Pages**;导出走右上角 Export → CSV(下载是 zip,内含多个 csv)。教用户操作时按新术语说,否则用户找不到入口。
- ⚠️ **URL Inspection「请求编入索引」有每日配额(约 10 条/天)**。排批次要留余量,**按 8 条一批**,别刚好排满;超配额会静默失败,得顺延到第二天。

### 5.4 拿到 GSC 数据后怎么定内容(含拼写变体决策)

用户上传 GSC 导出(Performance 查询词 + Indexing 网页)后,按这个顺序判:

1. **先看 Indexing 的 issues 是否为空**。Critical / Non-critical 全空 + Not indexed = 0,等于官方确认**没有技术障碍**,剩下就是"Google 还没来",主线继续推索引即可,不用再去怀疑 robots / 软 404。
2. **主词排名在 5–10 位**:说明 title/H1 方向已被 Google 认可,缺的是**权重**而非关键词布局。提权的正路是推内页收录让它们回链首页,不是堆词。
3. **出现拼写变体查询词(如 `boom banana`)时的处理规则**:
   - **默认不建独立页**。理由:① 变体搜索意图与主词一致,Google 自身拼写纠正会把流量导向主词结果;② 独立页内容必然重复 → thin content / doorway page 风险;③ 会和主词页形成 **keyword cannibalization**,稀释已有排名;④ 个位数月搜撑不起一页。
   - **正确落点**:把变体并入**已有的拼写变体 FAQ / 正文章节**(首页 + FAQ 页同步),零新增页面。
   - **重建独立页的触发条件**:变体月搜**稳定上百**,或变体指向的是**另一款不同游戏**。
4. **http 与 https 同时出曝光**:301 与 canonical 都在的情况下 Google 会自行合并,短期互相分散权重属正常,**不必改**,盯着即可。

### 5.5 按数据决定上哪一页 + 新增内页的完整清单(漏一项就白做)

**先定页,再动手** —— 用 GSC 数据决定,不要按拍脑袋的排期:

1. 导 `Performance → Queries`,把查询词按**需求簇**聚合(一簇 = 一组同义长尾),按曝光量排序。
2. 选「**有需求 + 无专页 + 位置 7–15 名**」的簇:那是临门一脚,给一个专页大概率进前 5。单簇占比 >15% 且无专页 = 最优先。
3. ⚠️ **务必先核对站内是否已有同类页**。模块类页面极易**供过于求** —— 本项目 9 个模块词合计仅 10 次曝光(2.5%),而站上已有 8 个模块页;原定的首推页 `/direction/` 因此被数据否掉(`direction` 零曝光),换成真正有需求的 `/crossplay/`(8 词 73 曝光 18%、位置 7.25–10.5、原先只有 `/faq/` 一段话承接)。
4. 首推页要能**同时接住别处裸奔的资源**(如 `/tools/` 里没有对应专页的求解器)。
5. ⚠️ **"无专页"要先排除首页与聚合页** —— 判断前 `grep` 全站 title/description 里有没有这个词。本项目踩过的坑:`wiki` 簇 2 词 23 曝光、位置 14.41,看着是标准的"有需求 + 无专页 + 位置 7–15"合格候选;但**首页 title 本身就是 `BOMBANANA! (Bomb Banana) Wiki — Guides, Roles & All 30 Levels`**,那个 14.41 名次就是首页拿到的。再开 `/wiki/` 只会和首页对打。同理,`品牌词` 已由首页承接、`主页 URL` 已由首页承接。
6. ⚠️ **合格候选可以是 0,那就一页都不上,不要为了维持节奏造页**。本项目 2026-09-22 实测:402 次曝光里 `levels 数量` 282 次(70%)+ `crossplay` 73 次(18%)= **88% 已有专页**;剩余 `wiki` 23(首页承接)/ `品牌词` 13(权重问题,靠外链)/ `模块类` 10(8 个页供过于求)**全部不满足条件**。当天正确结论就是"不开新页"。**"每天 1 页"是节奏上限,不是必须完成的任务量。**

**落地清单(每项都要做,缺一即页面失效)**:

1. `src/pages/<slug>.astro` —— 复制**同类型**页面当模板(表格型复制 `release-date.astro`),用 `<Base title description ogType="article" jsonLd={jsonLd}>`,jsonLd 用 BreadcrumbList。
2. **`src/components/Nav.astro` 加一项** —— 否则全站没有导航级链接,新页等于孤岛。
3. **首页 `src/data/homepage.json` 加卡片** —— 加进合适的 cards 数组(`start` / `moduleIndex` / `goodToKnow`)。首页是全站内链枢纽,**新页必须能从首页一步到达**。
4. **三份 sitemap 全部要补**:`public/sitemap.xml`、`public/sitemap/sitemap.xml`、`public/sitemap.txt`。**只补一处是最高频错误** —— GSC 实际读的是 `/sitemap/sitemap.xml`,漏了它等于没提交。顺手把今天真正改过的页面 `lastmod` 一起更新。
5. **正文内链 ≥5 个不同目标页**,且新页要被链进来(至少首页 + 一个同主题旧页)。
6. **防蚕食**:若新页承接的需求原先由某旧页的一段话覆盖,**把那一段改短并链到新页**,别让两页抢同一个词。
7. 构建后跑体检(§7):title ≤60 且含拼写别名、description ≤160、新页词数 ≥1300、正文内链 ≥5、**首页覆盖全部内容页**、无死链、sitemap 条目数 = 内容页数。
8. 部署后 `curl` 线上复验:新页 **200** + title 正确 + sitemap 里搜得到 + 首页有它的链接 + **无 `X-Robots-Tag` / meta noindex**。

⚠️ **词数统计口径**:⚠️ **口径必须全站统一,见 §5.6 最后两条** —— 本项目历史基线用的是**全页文本**词数(含导航/页脚),不要中途改成只数 `<main>`(同一页会低 100–150 词,同一阈值产生假报警)。用正则数 `href` 时**两侧必须用同一个 URL 归一函数**(把 `/roles/` 归一成 `/roles` 而目标字典保留尾斜杠,会导致"内链 0 条"的假报警,曾据此误判整站内链全断)。
⚠️ **算 title 字符数前先 `html.unescape`**:`&amp;` 会让原始 HTML 多算 4 个字符,误报超长。
⚠️ **事实分级落到新页**:A 类(价格/平台/人数/发售日)现查现写并标核验日;跨平台联机这类结论若只有单一社区源,**要么找第二源,要么明写"未宣布"** —— 不为了页面完整而下确定性结论。

### 5.6 加厚薄页(不新增 URL 的整改)

页面已存在但**没到质量线**时走这条,**不加新 URL、不占"每天 1 页"配额**,可以一次做多页(本项目 09-23 一次做完 switch/wires/calculator 三页)。

**什么时候做**:① 词数低于质量线;② 该页已进 GSC `Crawled - currently not indexed`。原则是**赶在 Google 首次认真抓取之前把页加厚**,比被拒后再补省掉一整轮验证周期(3–7 天)。

1. ⚠️ **已在 Validation(PENDING)的页:整改只能做一次,做完即冻结**。
   - 不要重复点 Request Indexing —— Google 已经在重爬队列里,再点只是重复排队,白耗滚动配额。
   - 加厚 + 刷 lastmod 要**在验证早期一次做完**,让重爬看到新版;之后**不要再改**。反复改会让判定基准漂移 → 验证结果无法归因(分不清是哪一版通过/失败),等于把 3–7 天窗口重新起算。
   - 验证期间改多少次都不会立刻触发新抓取,增量收益为零。真正的分界是"这轮重爬落地时 Google 看到的那一版"。
   - 验证通过 → 不用管;验证结束仍是 `Crawled - currently not indexed` → 才做第二轮诊断(内容重构/合并/换角度),**不是继续堆字数**。
   - **对照:不在验证队列的薄页正当其时** —— 曝光≈0 说明 Google 还没认真抓过,赶在首次抓取前加厚最优(本项目 09-23 switch/wires/calculator 属此类,当天做完)。**判据:在验证队列里的不要动;不在队列里、又不合格的,优先改。**
2. **内容只从素材库取原文**(`bombanana-inner-page-assets/keyword-material-*.md`),先 `grep '^#'` 列出素材章节,挑**还没上页**的段落。每页补 2–3 个新 H2、每个 150–250 词,不要用泛泛的收尾话灌水。
3. **实测最有效的四类补充**(都能直接落成 H2):
   - **准备度判据** —— "什么时候算学会了"清单(如 "Cable is ready when the callout needs no discussion"、"Leave Level 16 when one read becomes one submission")。
   - **失败分类** —— 把失败拆成有限几种并各配修法(如 "四种死法"、"repeat one fact vs restart the board" 的区分)。
   - **常见记忆陷阱** —— "上一关的答案套到这一关为什么错"(如 3 线 Blue→Green 正确、4 线 Blue→Yellow)。
   - **关卡上下文** —— 首次出现的关卡 + 计时 + 同期模块组合,**同时接上 `/levels/` 内链**。这是最省力的加厚方式:既补词数又加固内链。
4. **顺带补一条同主题 FAQ** —— 一次同时加词数、扩 FAQPage schema、覆盖长尾(如 `which level does X first appear on`)。
5. **必须同步 source note** —— 写清新增段落来自哪个源,否则违反 §2.0 的事实分级。
6. **两份 XML sitemap 刷该页 lastmod**(`sitemap.txt` 无 lastmod),再走 §7 体检 + 线上 `curl` 复验。
7. ⚠️ **改完要提交一次索引 —— 刷 lastmod 不能代替它**。判据是"这页现在有没有人替它排队",不是"改没改":
   - **不在任何队列的页 → 要提交**。sitemap `lastmod` 只是 discovery **hint**(Google 仅在"持续且可验证准确"时采信),站内链 + sitemap 只保证"被发现(discovered)",不保证"被优先抓取(crawled)"。对新站低权重页,`lastmod` 基本不驱动抓取,`Request Indexing` 是唯一能插队的手段。
   - **在 Validation(PENDING)的页 → 不提交**(见第 1 条,Google 正在重爬)。
   - **提交 ≠ 收录**:`Request Indexing` 只影响"什么时候抓",不影响"抓完收不收"——收不收仍然看质量(这正是不达标的页会被判 `Crawled - currently not indexed` 的原因)。
   - 提交纪律:逐页先跑 URL Inspection 的 live test(不占额度),只对"不在索引中"的点提交;必须带尾斜杠;只点普通 Request Indexing,**绝不点 "…and all direct links"**;一天一次性推 5 条左右没事,别一次推 7–9 条。

⚠️ **词数口径必须统一,别中途换**。本项目历史基线用的是**全页文本**词数(含导航/页脚);改用**只数 `<main>`** 会让同一页低 100–150 词,同一个 1300 阈值产生假报警(09-23 实测 `/tools/` 全页 1370 / 正文 1267、`/braille/` 全页 1332 / 正文 1228)。选定一个口径并写进报告,不要让两套数字打架。
⚠️ 用 Edit 替换多行段落时**容易吞掉换行**并把两行压成一行 —— 改完抽查被改页面的行结构(本项目 09-22 在 `/release-date/` 上踩过)。

### 5.7 给已排名的页补权重(只改锚文本,不碰正文)

某页已经排在第 1 页底部(位置 7–15)、但曝光高、点击低时,瓶颈常是**权重不够**而非内容不够(本项目 `/levels/` 555 曝光 3 点击、0.54% CTR,排 9.1)。此时唯一安全有效的动作是**内链锚文本** —— 它不动正在排名的页面正文,风险接近 0。

1. **先定"哪些页该指向它"**:① 首页(全站枢纽);② 该词的引导页(如 `/how-to-play/`);③ 同主题聚合页(如 `/modules/`)。
2. **至少一处用目标词族的问句形态**,不要三处都是同一句(过优化)。本项目锚文本演进可作参照:`See All Modules`(**错**:复制粘贴 bug,文案与 href 不符,反而给最大词族传错信号)→ `See All 30 Levels`(修好了,但只是 CTA 语言)→ **`how many levels BOMBANANA! has`**(命中 how-many-levels 簇)。**CTA 按钮保留 CTA 语言,关键词锚放在正文句子里。**
3. **卡片型链接的整块都是锚文本**:卡片标题 + 描述都包在 `<a>` 内,所以**改描述文案 = 改锚文本**,不用动 HTML 结构。
4. ⚠️ **动手前必须实测 `href`,别只看文案**。本项目 09-22 复评:首页唯一含 `how many levels` 的链接**指向 `/faq/` 而不是 `/levels/`**;`/modules/` 满篇讲关卡却 **0 条**链向 `/levels/`。**光读文案会以为早链好了。** 必查路径:抽出每个 `<a href="/目标/">` 的锚文本 + 上文做人工核对。
5. ⚠️ **审计"页面 A 是否链到页面 B"时,必须剔除站点级模板块**(`Nav` / `RelatedModules` / 页脚)。这些组件渲染在 `<main>` 内,所以"只数 `<main>` 里的链接"**筛不掉它们** —— 结果是样板块的链接冒充成了正文上下文锚。本项目 09-23 实测:`/modules/` 看起来有 `Braille module guide` / `Switch module guide` 等精准锚,**实际全来自 `RelatedModules` 样板块**,页面自己的正文对 `/braille/` `/switch/` `/calculator/` **0 条**上下文锚,而我 09-22 已经据此告诉用户"早已存在、无需再做"。**做法:抽出 `<main>` 后按 `Related guides` 之类的块标题切断,只统计切断点之前的部分**;判定"某模块词有没有被承接"要以正文上下文锚为准。
6. ⚠️ **"没有专页"要逐个验证,别照抄评审给的映射**。本项目评审列的 5 个模块词里(braille / soundboard / symbol / piano / morse),**只有 braille 有专页**;`/tools/` 另有 Direction 与 Soundboard solver 可承接这两个词,而 symbol / piano / morse / maze 全站**无任何页面覆盖**,只能留纯文本。**硬给这些词找页会造出"锚文本与目标无关"的链接,比不链更糟。**
7. ⚠️ **专家/评审提的"把答案提前"要先验证前提**。本项目评审要求把 `/levels/` 第一个 H2 的直接答案从"正文第二段"提到更前 —— 线上 HTML 实测 H1 之后**紧接 H2,答案就是第一段**,结构已最优,他看的是旧版。**前提不成立就明确写"不改"并说明理由**,不要为了显得执行了而乱动正在排名的页。
8. **改完**:刷 lastmod → 构建体检 → 部署 → `curl` 复验锚文本真的变了(改了源码但没生效,是抽 href 时最容易漏的一步)。索引提交见 §5.6 第 7 条。



### 5.8 全站 TDK / 结构化数据一致性核扫(机器逐页比对,不靠肉眼)

**触发**:① 有人(评审/自己/上游)说"某页漏改";② 任何批量改动后;③ 上线前例行。

**为什么必须机器扫**:人工抽查必漏,而且**"只此一处"这种断言本身就是必须验证的对象**。本项目评审说"`BreadcrumbList` 缺尾斜杠全站只此一处(`/levels/`)",实测是 **3 处**:`/levels/` `/roles/` `/how-to-play/` —— 全是手写 JSON-LD 时漏了尾斜杠,而 canonical 是 Astro 自动生成的、反而正确,所以肉眼比对 TDK 时看不出来。

**逐页核对项**(对每个 `dist/**/index.html`):

1. `<link rel="canonical">` == 期望 URL(由构建目录路径推出)。
2. `og:url` == canonical。
3. `og:title` / `twitter:title` == `<title>`。
4. `twitter:card` == `summary_large_image`;无 `meta robots=noindex`。
5. `<title>` 长度 ≤61 —— **先 `html.unescape` 再算长度**,`&amp;` 会多算 4 字符,不 unescape 会误报超限。
6. description ≤160。
7. **JSON-LD 里所有 `ListItem.item` / `@id` / `url` 全部以 `/` 结尾,且与 canonical 一致** —— 这条最容易漏,就是被发现的那类 bug。
8. 正文内链:抽 `<main>` → **按 `Related guides` 块标题切断**(见 §5.7 第 5 条)→ 统计;检查①死链②**站内 href 是否漏尾斜杠**(漏了会 308,白烧抓取预算)③核心目标页的锚文本清单。
9. 全页词数 ≥ 质量线(本项目 1300;**统一用全页口径,别只数 `<main>`**,见 §5.6)。
10. **每页恰好 1 个 `h1`,且 h1 文本 ≠ title**;h1→h3 不许跳级(§3.2 第 2 条)。
11. **所有 `<img>` 有 alt、有真实 width/height;首屏(出现在 h1 之前)的图没有 `loading="lazy"`**(§3.2 第 3 条)。

**本项目可复用脚本**:`site/scripts/site-audit.py`(构建后运行,一次输出 TDK 表 + 内链表 + 锚文本清单)。

### 5.9 重定位"已抓取-未编入索引"页(换角度,不是加厚)

GSC 的 `Crawled - currently not indexed` 是**内容质量问题**:Google 抓了但认为不值得收。§5.6 的"加厚"只对**还没被抓过、曝光≈0** 的页有效;对**已抓取未收录**的页,病因通常是**与别的页重复/自相残杀**,堆字数无效,要**重定位(换角度)**。本项目 09-23 三页一次重定位成功,作为范式:

1. **判病因三查**:① 这页的 H2/正文是否和另一个内页撞题(`/roblox/` 的 roles 段撞 `/roles/`、crossplay 段撞 `/crossplay/`);② 是否"一个事实撑一页"(`/release-date/` 只有发售日,增量内容被首页 FAQ 覆盖);③ 是否在转述竞品/第 1 名(`/manual/` 大段转述 whisperofthehouse)。
2. **重定位 = 换角度,不是删内容**:`/manual/`→「模块速查表 + 可打印」;/roblox/→「正版 vs 山寨鉴别页」;/release-date/→「价格 + 配置做主体」。每个都对准一个真实的查询意图。
3. **分流原则**:撞题的那段**删掉、重写成一两句 + 链接指向该主题专页**,不要在 A 页复述 B 页的内容。
4. **硬规则:两页不要同时放同一份规则表**。一个放表格(速查)、一个放正文(原理)——如 `/manual/` 只放表、`/modules/` 只放正文。
5. **可打印页**:静态速查表配 `@media print`(隐藏 nav/页脚/按钮)+ 一个 `window.print()` 按钮即可,不另出 URL。
6. **同步瘦首页 FAQ**:若首页 FAQPage 大面积覆盖这些主题,把相应条改"一句短答 + 专页链接"、FAQ 模块保留给用户看。**FAQPage 富结果已死(见 §3.1),瘦它不是为 SEO,是为了"结构化数据与可见文本一致" + 消除重复。** 短答可带 `link` 字段(FAQ.astro 已支持)。
7. **部署前存基线**:改动已排名的页(尤其首页)前,先在 GSC 截图存档——首页 + 受影响页的查询词/曝光/点击/平均排名、当前收录状态(已收录数 / 已抓取未收录数)。改完才能回答"涨了还是掉了"。这与 §5.7 "改排名页只改一次、留回滚"是同一件事。

## 6. AdSense 变现

### 6.1 申请时机(最容易踩的坑)

- **不要网站一上线就申请**。AdSense 会审核站点内容质量,最常见被拒理由是 **"内容不足 / 价值不足 (Insufficient content)"**。空站或只有几页、内容单薄的站几乎必拒,且被拒后需冷却一段时间才能再申。
- **建议门槛**:① 内页补齐(内容扎实,至少 8–10 个实质页面);② 有**自然流量**(哪怕每天几十个真实访客,来自谷歌搜索最佳)。两者齐了再申,通过率高很多。
- 注意:注册 AdSense 账号本身不需要流量(有个 Google 账号就行),但要让广告**真正投放到你的域名**,必须经过站点审核。

### 6.2 代码预埋(提前做,不阻塞)
- 即使暂不申请,也应提前把广告位**占位**埋好,避免日后大改布局。用 `src/components/AdSlot.astro`,内部 `client` 常量先填占位 `ca-pub-XXXXXXXXXXXXXXXX`。
- 广告加载脚本建议放在 `Base.astro` 的 `<body>` 末尾(或用 Astro 的 `is:inline` 在 head 加载 `<script async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js">`),页面里用 `<ins class="adsbygoogle">` + `AdSlot` 组件占位。
- 占位阶段不影响页面渲染,只是不展示真实广告。

### 6.3 拿到发布商 ID 后的注入
用户从 AdSense 后台(网站/账户设置)复制 `ca-pub-XXXXXXXXXXXXXXXX` 发来后:
1. 替换 `AdSlot.astro` 里的占位常量(前端公开值,**不是密钥**,可进仓库)。
2. 重新 `npm run build` + `wrangler pages deploy`。
3. 线上验证:首页 HTML 里应出现 `ca-pub-真实ID`,且 adsbygoogle 脚本 200。

### 6.4 与 GA4 打通(可选但推荐)
- 在 AdSense 后台关联 GA4 测量 ID(`G-XXXXXXX`),可把"广告收益"与"流量渠道"关联分析——看清哪个渠道带来的访客广告价值最高,指导后续内容方向。
- GA4 注入见 §5.2,两者互不干扰。

### 6.5 合规提醒
- 广告位不得遮挡主体内容、不得用诱导点击文案;移动端广告密度遵循 Google 政策(避免版面休克 layout shift / 过多插屏)。
- 页面需有清晰导航与原创实质内容,纯 SEO 采集页易被拒或后续封号。

## 7. 实战踩坑清单(Checklist)

- [ ] token 权限含 Pages:Edit + Zone:DNS:Edit(否则自定义域名卡 pending)
- [ ] **www 子域已加(Pages 域名 + DNS CNAME)并 301 跳主域**,含 query string 场景(见 §4.4)
- [ ] GSC 用的是**网域属性**(Domain property),不是网址前缀(见 §4.5)
- [ ] 自定义域名:apex 根域可 API 手动建 CNAME 转 active;**子域经 API 校验不认手动 CNAME,必须 Dashboard 点一次 "confirm/激活"**(见 §4.3)
- [ ] OAuth 连接器在本机不可用 → 用静态 env 文件
- [ ] wrangler 装在 site 工程内(--no-save),别用受管 workspace
- [ ] 导航 href 与路由统一带尾斜杠
- [ ] sitemap 的 loc 全部带尾斜杠,且与 canonical 一致
- [ ] **JSON-LD `BreadcrumbList` 里每个 `item` URL 也带尾斜杠**(手写 JSON-LD 时最常见漏点;全站 3 页曾漏:`/levels/` `/roles/` `/how-to-play/`)
- [ ] 上线前跑一次全站 TDK 机器核扫(见 §5.8),不靠肉眼抽查
- [ ] GA4 用 `define:vars` 注入,不碰 `is:inline` 插值
- [ ] 补页时同步更新 sitemap + Nav
- [ ] meta description 控制在 140–160 字符、title 控制在 50–60 字符(超长会被 SEO 工具标红,先校验再提交)
- [ ] 部署后遍历所有路由验 200,再提交 GSC
- [ ] 测外部站点一律加 `--noproxy '*'`;验证刚改的 DNS 用 `dig @1.1.1.1` 或 `--resolve`,别信本地负缓存
- [ ] sitemap 提交失败排障:先 www → 再标签顺序 → 再尾斜杠 → 再 IPv6 → 最后拿 Bing 做对照
- [ ] **Git 集成路线(API 建 `source.type=github` 项目)的两个坑(2026-09-26 pixlarc 实战)**:
  - 建完项目 **不会自动触发首次构建**(`deployments` count=0)。必须手动 `POST /accounts/{acct}/pages/projects/{name}/deployments` body `{"branch":"main"}`。之后 push 才会走 webhook 自动部署。
  - 建完项目 **也不会自动建 DNS 记录**(zone 里 `dns_records` total=0),域名会一直卡 `pending`。必须手动补两条 `proxied=true` 的 CNAME:`@` → `<project>.pages.dev`、`www` → `<project>.pages.dev`。CF 支持 apex CNAME(自动 flatten 成 A),补完约 90 秒转 `active`。
  - ⚠️ 该次实战中 **www 子域仅靠 API 建 CNAME 就转成了 active**,未出现 §4.3 所述"必须 Dashboard 点确认"的情况。两边结论冲突,以**实测状态为准**:先 API 建 CNAME,等 2 分钟看状态,仍 pending 再去 Dashboard 点。
- [ ] **构建脚本要查 Python/Node 版本兼容性**:CF Pages 构建环境版本低于本机。Python 项目上线前先跑 `/usr/bin/python3 -m py_compile build.py`(系统 3.9)验证,别只信本机 3.13;优先只用标准库,避免构建环境缺依赖。
- [ ] **验证刚上线的站点必须 `curl --resolve <dom>:443:<dig 到的 IP>`**:本机 curl 对新解析域名会报 `Could not resolve host`(沙箱 DNS 限制),而 `dig` 正常。不绕过会把"正常站点"误判成故障。配合 `-w '%{ssl_verify_result}'`,返回 0 表示证书链正常。
- [ ] **www → 主域 301:优先用 Cloudflare Worker,别指望 Pages 的 `_redirects`**(2026-09-26 pixlarc 实战)。Pages 的 `_redirects` 写 `https://www.x.com/* https://x.com/:splat 301` **实测不生效**(www 仍 200);Pages 会把 `_redirects` 当特殊文件不对外 serve(请求 `/_redirects` 返回首页属正常,说明文件被识别了,只是不支持 hostname 级匹配)。
  - API 权限不足时的五条死路(本机 token 实测全被拒,别浪费时间):zone 级 `http_request_redirect` ruleset → "phase not allowed at zone level";zone 级 `http_request_dynamic_redirect` → "not authorized";`/zones/{id}/purge_cache` → 10000;`/zones/{id}/workers/routes` → "No access"。
  - **可行顺序**:① `wrangler deploy` 部署重定向 Worker(有权限,需 `CI=1` + token/account env,wrangler 用绝对路径)→ ② 从 Pages **解绑** www 域名 → ③ **先删掉 www 的 DNS 记录**(否则 100117 "already has externally managed DNS records")→ ④ **PUT**(不是 POST)`/accounts/{id}/workers/domains`,body `{hostname, zone_id, service, environment}`(POST 报 10405)→ ⑤ 约 70 秒自动建 DNS + 签证书生效。
  - Worker 代码就是改 `url.hostname` 后 `Response.redirect(url, 301)`,路径与 query 自动保留。
  - 顺带确认:`http://` → `https://` Cloudflare 会自动 301,不用配置。实测某竞品站的图命名极不统一(`{slug}-thumbnail.jpg` 只覆盖少数,其余是 `{gamename}{数字}-m180x180.png`),硬拼 12 个只命中 1 个;逐页抓 og:image 12/12 成功。
  - **游戏源站通常拿不到封面**:azgame 类源站的游戏页是纯 loader(多个页面返回字节数完全相同),无 og:image;gameflare 整站 403 反爬。
  - **拿到的大多是 180×180 小图**,放大到 400×300 卡片会软。要么接受,要么用 ImageGen 生成原创高清封面(顺带解决版权 + 同质化)。
  - **图片放源码目录(如 `static/thumbs/`)并提交 git**,不要放 `dist/`(构建会被 rmtree 清掉);构建脚本按文件名匹配,缺失回退占位图。

- [ ] **批量扩游戏库的可复用流水线(2026-09-26 pixlarc:12 → 42 款)**,顺序不能乱:
  1. **先从竞品 sitemap 拿全量页面**(`curl <竞品>/sitemap.xml` 正则 `<loc>`),减去已有 slug 得到新增清单。别猜 slug——实测猜 34 个只中 6 个。
  2. **逐页提取三样东西**:`data-wf-game-url`(真实可嵌入地址,部分站用 `data-src`/`data-game-url`)、`og:image`(封面)、`meta description`(真实玩法描述,用作写作依据)。
  3. **逐个验证可嵌入**:HTTP 200 且响应头无 `X-Frame-Options`、CSP 无 `frame-ancestors`。并发 10 线程跑,几十个游戏几秒钟。
  4. **写原创文案**:用第 2 步抓到的 meta description + 操作说明当事实依据,但**必须自己重写**,不要复述竞品原句(手册明令:照抄模板的站拿不到流量)。
  5. **重建 → 本地冒烟 → push**,Cloudflare 自动部署。

- [ ] **游戏源 CDN 图谱与可靠性(2026-09-26 实测)**:
  - 可用:`https://game.azgame.io/{slug}/`、`https://gamea.azgame.io/{slug}/`、`https://azgames.io/{slug}.embed`、`https://zapgames.io/{slug}.embed`、`https://st.8games.net/...`、各类 `*.github.io` 页。均实测无 XFO/CSP 限制。
  - ⛔ **不要用 `gameflare.com`**:首次探测可能返回 200,但随后稳定 403,**加 Referer 也无效**(属 IP 级封禁)。它是"先给你甜头再失效"的陷阱源,已导致 3 个页面上线后源失效。
  - 同一款游戏通常能在多个 host 上找到,**换 host 是修死链的第一手段**(例:`traffic-road` 在 `game/gamea.azgame.io` 均 404,但 `azgames.io/traffic-road.embed` 200)。

- [ ] **游戏源要定期复查,不能只验一次**:初次验证全部 200,构建完再复查就冒出 3 个 403。把源健康检查写成脚本(如 `tools/check_games.py`)纳入常规流程,每次加完游戏跑一遍;退出码非 0 就说明有坏源。

- [ ] **内容安全过滤(AdSense 品牌安全)**:批量扒竞品清单时会混进名称涉及真实敏感人物/事件的游戏。**这类必须剔除**,哪怕技术上能嵌入。人工过一眼新增清单再入库,别全自动。

- [ ] **IndexNow 主动推送:加完页面就推,别等爬虫**(2026-09-26 pixlarc 实战,首次推 55 条返回 202):
  - **顺序不能反**:① 生成密钥(32 位 hex 即可)→ ② **构建脚本把密钥写进产物根目录 `<key>.txt`**(内容就是密钥)→ ③ 部署并**确认线上 `https://<host>/<key>.txt` 返回 200 且内容一致** → ④ 才提交。IndexNow 会回源拉取 keyLocation 做归属验证,文件没上线时提交会被静默丢弃。
  - 提交:`POST https://api.indexnow.org/indexnow`,`Content-Type: application/json; charset=utf-8`,body `{host, key, keyLocation, urlList}`。**正常响应是 200 或 202**,都算成功,不要把 202 当错误。
  - 上限 **10000 条/次**,超出分批;`urlList` 里非本域的 URL 会被丢弃,提交前先过滤。
  - **密钥不是机密**——IndexNow 的设计就是要求它公开可访问才能验证归属。直接写成构建脚本常量即可,**不要塞进 env 或密钥文件**(两处维护且没必要)。
  - ⚠️ **Google 不参与 IndexNow**(只有 Bing / Yandex / Seznam 等)。Google 侧仍要回 GSC 手动重新提交 sitemap,别以为推了 IndexNow 就完事。
  - 建议做成 `tools/indexnow.py` 并支持 `--urls a,b,c`,以后每次加页面推一次增量即可。

## 8. 与其他项目的衔接

- 本 skill 是**通用流程**,不绑定具体账号/域名。调用时由用户提供:Cloudflare Account ID + 带权 token、域名、素材仓库路径。
- 素材红线(§2)是硬约束,任何项目生成内容都适用。
- AdSense 接入时机与方式见 §6。
