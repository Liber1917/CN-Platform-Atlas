# Benchmark v0：私域检索 42 题（v0.1）

> 出题哲学（私域版 BrowseComp）：题目刻意选**通用搜索不易直接命中**的信息需求——难点来自平台封闭性，而非网页深处。
> 状态：v0.1（2026-11-07：36 题 + G 组多模态 6 题，全量补 `modality` 字段）；判据均为结构化判据，不依赖内容快照。

## 文件

- `queries.v0.jsonl` —— 36 道题，每行一题

## 字段定义

| 字段 | 含义 | 取值 |
|---|---|---|
| `id` | 题号 | A1-G6（A=事实核查 B=口碑 C=价格 D=热点 E=深研 F=找人 **G=多模态**） |
| `query` | 查询原文 | 以最终用户口吻书写 |
| `intent` | 意图类型 | fact-check / review / price / trending / research / find-people / multimodal |
| `modality` | 内容形态 | text 文本 / video-asr 视频语音 / audio-asr 播客音频 / video-ocr 画面文字 / av-align 音视频对照 / image 图片理解 |
| `layer` | 预期主源可达层 | L0 公开 / L1 登录态 / L2 强反爬 / L3 桌面封闭 |
| `difficulty` | 难度 | single-source 单源可答 / cross-source 需跨源 / time-sensitive 时效敏感 / multi-hop 多跳 |
| `freshness` | 时效要求 | very-high(≤1天) / high / medium / low |
| `expected_sources` | 预期源集合（路由准确率对照） | 平台名列表 |
| `sufficiency_rule` | 充分性判据（结构化） | "≥N 条证据跨 ≥M 平台 + 具体要求" |
| `snapshot_ref` | 离线快照参照源（防评测漂移） | v0 全部为 null，机制见下文 |

## 题量分布

- 六类意图 × 各 6 题 + **G 组多模态 6 题**（流媒体内容可读化：视频口播转写 / 播客音频 / 画面 OCR / BGM 音视频对照 / 图穿搭理解）
- 层覆盖：L0 ×14、L1 ×19、L2 ×7、L3 ×1（L2/L3 刻意少——这两层专门测降级链与人工介入，跑一次成本高）

### G 组（多模态）出题依据

流媒体平台的内容不以文本存在——"搜到"的前提是先"读得懂"（ASR/OCR/音视频对照）。这组题专测**内容可读化管线**：G1/G5 视频口播转写、G2 播客音频、G3 画面 OCR、G4 BGM 归因（音视频对照）、G6 图片理解。预期：基线 A（纯文本搜索）在此组系统性失败——这正是多模态工具链（字幕/转写渠道）的价值增量所在。

## 五维判分

| 维度 | 判什么 | 怎么判 |
|---|---|---|
| 路由准确率 | 实际查询的源 ∈ expected_sources | 按轨迹对照 |
| 充分性达成 | 返回内容满足 sufficiency_rule | 规则判定；人工抽检用 Needs Met 三级（完全/部分/不满足） |
| 答案质量 | 忠实度/完整度 | LLM-as-judge（评审模型固定版本并记录）；判罚 filler（凑字）/paraphrased（仅复述无综合） |
| 成本 | token、耗时、调用次数、人工干预次数、配置次数 | 账本统计 |
| 鲁棒性 | 主源失效后降级成功率 | 混沌测试（人为注入 cookie 过期/源超时） |

每题结果按 **路由/检索/融合/答案** 四环分别记录归因——输出错误分布，不只给总分。

## 基线

| 系统 | 构成 | 回答的问题 |
|---|---|---|
| 基线 A | LLM + 通用联网搜索直答，无私域工具 | A vs C = 私域路由的价值增量 |
| 基线 B | LLM 自持全部聚合工具自由决策：**cn-scraper-mcp**（项目 `.mcp.json` 已注册）+ **Agent-Reach CLI**（github/bilibili/v2ex/youtube/web 默认渠道开箱可用；xiaohongshu/xueqiu/boss/linkedin 需 `agent-reach login <渠道>` 扫码；youtube·xiaoyuzhou 转写需 Groq Key） | B vs C = 专职路由层 vs 临场自由发挥 |
| 对照 C | 路由层 v0 | — |

跑法：42 题 × 各系统 × 3 次；基线 B 单题预算上限预设，超限记失败（这本身也是数据）。G2/G5 等转写题在无 ASR 配额时记 N/A 并注明，不计入失败。

## 结果目录约定

评测产物（轨迹、账本、原始抓取）一律存 `runs/`（已 gitignore，不入库）；入库的只有题目、判分规则与汇总报告。

## 快照机制（v0 未启用，设计预留）

防评测漂移（BrowseComp-Plus/CRAG 教训）：L0 题在入库时抓取参照源存档，`snapshot_ref` 指向存档路径；线上源漂移时以快照判定路由与检索是否"本可命中"。首轮实测后评估是否启用。

## 已知边界

- 时效类题（D 组、freshness=very-high）跨天复跑结果必然不同——判据按结构判定，且复跑须记录时间戳
- E5（微信群谣言源流）依赖 L3 桌面操作与个人数据，仅在有真实样本时执行，默认跳过
- L2 题（A4/B5/C5/D5/F6）涉及登录+验证码，人工介入计入成本维度，不隐藏
