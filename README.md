# undergrad-thesis-self-check

> 南京大学软件工程专业本科毕业论文送审前自检 skill 。

把论文 PDF 提交给 Agent，按 33（工程型）/ 31（学术型）条 checklist 逐项审查（含 5 条
参考文献规范条目，与硕士版要求一致），并对软件工程图表规范性（用例图 / 架构图 / ER 图 /
类图 / 流程图的 UML 规范）进行评估，另可选追加 AI 文风与错别字检查（基于 humanizer-zh），
输出一份带页码定位、按"红线 / 警告 / 通过"分级的 Markdown 修改清单，便于在送审之前
自查可程序化检测的问题。

适用对象：

- 主：南京大学软件工程专业本科生
- 次：其他高校工科本科生。大部分条目（字数、参考文献、UML、图表编号、引用规范、消融实验等）
  也通用

## 用途

学生送审前若命中红线（字数不足 / 参考文献不规范 / 错别字 / 缺关键章节），外审专家
会直接打回大改甚至一票否决。本 skill 将上述可程序化检测的项目执行一遍，让作者在
送审前完成自查，将精力集中在答辩准备上，避免陷入可规避的问题。

设计目标是降低"被外审一票否决 / 打回大改"的概率，不替代答辩或学校查重系统。

参考文献规范方面，本 skill 与硕士版 `nju-se-thesis-self-check` 的要求**保持一致**，仅在
数量门槛上做本科化（条数 ≥ 20 而非 ≥ 40；英文文献 ≥ 5 而非 ≥ 10；中文文献 ≥ 3 而非 ≥ 5）。
其他规范（双向引用 / 编程书 / arXiv 上限 / 防伪英文文献 / 引用格式 / 非同行评议来源处理）
与硕士版完全对齐。


## 何时用 / 何时不用

适合：
- 论文已完成终稿，准备打印 / 上传送审系统之前的自查
- 需要识别哪些条目属于"红线"（一票否决项），以便优先修复

不适合：
- 论文仍处于初稿阶段：结构未定型时执行 checklist 只会产生大量误报
- 用于查重 / 抄袭检测：学校系统已覆盖，本 skill 不重复实现
- 检查代码仓库是否可运行、Demo 是否正常：属答辩范畴

## 安装

本 skill 运行于 Agent 环境内（如 Claude Code），并非独立命令行工具。

### 1. 先装前置依赖

```bash
pip install pdfplumber pymupdf
```

- `pdfplumber`：**硬依赖**，骨架抽取 / chunks 切分 / 题目抽取均依赖此库，缺失则 skill 无法运行
- `pymupdf`（即 PyMuPDF）：**软依赖**，步骤 8 图表规范性检查渲染整页 PNG 时使用，缺失则跳过
  视觉评估、退化为图注文本检查，不影响其他步骤

另可选安装 [humanizer-zh](https://github.com/op7418/Humanizer-zh) skill（用于步骤 11
AI 文风扫描）。未安装时文风检查仅执行内置错别字部分。

详细依赖矩阵见下文 [前置依赖](#前置依赖) 段。

### 2. 安装 skill 本身

在一个 Agent（比如 Claude Code 等）里输入：

​	安装一个 skill： https://github.com/dongshao/undergrad-thesis-self-check.git

### 3. 触发使用

新建 Agent 会话后，输入 "本科论文自检 /path/to/paper.pdf" 即可触发。

## 触发词

必须 **同时** 满足两个条件：
1. 用户明确说出以下触发词之一：
   - "本科论文自检" / "本科论文自查" / "本科送审前自查"
   - "我的本科论文有什么问题" / "我的本科论文能过吗"
   - "undergrad self-check"
2. 提供 PDF 路径

仅检测到 PDF 而无触发词时，**不要** 启动 skill，以免把无关 PDF 当作论文执行一次完整审查。

## 调用示例

```
本科论文自检 ~/Downloads/我的论文终稿.pdf
本科论文自检 ~/Downloads/我的论文终稿.pdf --type=engineering    # 跳过类型判定 Gate
本科论文自检 ~/Downloads/我的论文终稿.pdf --type=academic --team # 学术型 + 团队子模块
本科论文自检 ~/Downloads/我的论文终稿.pdf --auto                # 全自动模式（任何 Gate 都不询问）
```

参数：
- `--type=engineering / academic`：跳过步骤 3 的类型判定 Gate
- `--team`：标记为团队子模块论文；不带这个参数也会在步骤 4 主动询问
- `--auto`：全自动模式。所有 Gate（题目确认 / 类型判定 / 团队判定 / 文风检查询问）都不再询问，
  由 skill 自行判断后直接走完 11 步流程；用户消息含 "全自动" / "自动模式" / "auto" / "-y" /
  "yes to all" 等关键词也会触发

## 主流程（11 步）

人工模式下每步均设有 Gate：用户未确认或关键信息缺失时即停下询问，**不自动推断**；
`--auto` 模式下全程不询问，由 skill 自行判断执行完毕。

| 步 | 名称 | 一句话说明 |
|---|---|---|
| 1 | 骨架抽取 + 题目抽取 | 使用 `scripts/extract_skeleton.py` 将 PDF 转换为带页码的 Markdown 骨架；同时按 4 策略从前 10 页抽取论文真实题目（不依赖文件名）；中间产物写入 `<PDF 同目录>/tmp/`（顺延命名，不覆盖前次） |
| 2 | 题目确认 Gate | 将抽取到的题目交学生确认，避免封面 OCR 错误污染后续报告 |
| 3 | 类型判定 Gate | 依据摘要 + 章节结构信号判别工程型 / 学术型，列出依据请学生确认；指定 `--type` 时跳过 |
| 4 | 团队子模块 Gate | 默认主动询问"是否团队论文"；如是，追问团队总项目名与本人模块名 |
| 5 | 加载 reference + overrides | 按类型读取对应 checklist；团队论文额外加载 `team-submodule-overrides.md` 完成条目 patch |
| 6 | 红线扫描 | 工程 4 条 / 学术 3 条红线优先执行：均为评审一票否决项 |
| 7 | 全量审查 | 其余约 30 条按"评判 + 定位 + 修改建议"模式逐条审查（含 5 条 UND-ENG-F 参考文献规范） |
| 8 | 软件工程图表规范性检查 | 探测模型图像识别能力 → 渲染含图页 PNG → 按 `diagrams.md` 的 DGM-1~6 对用例图 / 架构图 / ER 图 / 类图 / 流程图执行 UML 规范评估；模型不支持图像输入时退化为图注文本检查 |
| 9 | 主报告生成 | 写入 `<PDF 同目录>/<论文题目>-review.md`，同名已存在则顺延为 `-review-1.md / -2.md` 以免覆盖 |
| 10 | 对话输出摘要 | 仅输出报告路径、类型、红线数量与必修项，详细分项另存报告 |
| 11 | （可选）AI 文风 + 错别字检查 | 调用 `scripts/extract_chunks.py` 按章切分，再调用 [humanizer-zh](https://github.com/op7418/Humanizer-zh) 扫描 AI 写作模式 + 内置错别字 / 标点检查；详细列表写入独立 `-style.md` 副报告，主报告仅追加摘要指针 |

为什么图表与文风独立成两步：

- 步骤 8 的图表警告 **不计入** 第一节的"红线 / 警告 / 通过"总计，仅供修改参考，以免单张
  非规范流程图影响整篇论文的整体统计
- 步骤 11 设为可选：人工模式下先询问学生意愿（耗时较长且消耗较多 token）；AUTO_MODE
  默认执行，humanizer-zh 未安装时降级为仅错别字检查
- 文风结果不与主报告混编（条目可达数十条，会冲淡核心结论），改为独立 `-style.md` 副报告，
  与主报告共用同一编号顺延（如 `-review-2.md` 对应 `-style-2.md`）

## 目录结构

```
undergrad-thesis-self-check/
├── README.md                           # 本文件
├── SKILL.md                            # skill 主入口（Claude 启动时读这个）
├── references/
│   ├── engineering-30.md               # 工程型 28 条 checklist（4 红线 + 10 学术质量 + 10 结构 + 4 格式）
│   ├── academic-30.md                  # 学术型 26 条 checklist（3 红线 + 10 学术质量 + 5 结构 + 3 格式 + 5 学术专属）
│   ├── team-submodule-overrides.md     # 团队子模块论文的条目 patch（约 8 条 override）
│   └── diagrams.md                     # 软件工程图表评估清单（DGM-1~6：用例图/4+1 视图/ER 图/类图/流程图/时序图）
└── scripts/
    ├── extract_skeleton.py             # PDF → 带页码的 Markdown 骨架抽取脚本（依赖 pdfplumber）
    └── extract_chunks.py               # PDF → 按章切块的纯文本 chunks（步骤 11 文风检查使用）
```

## references 清单

skill 运行时按论文类型与是否为团队论文动态加载对应文件，主入口 `SKILL.md` 不重复包含 checklist 内容。

### `references/engineering-30.md`（工程型，33 条）

适用于"系统 / 平台 / 工具的设计与实现"类论文。条目分布：

| 分类 | ID 前缀 | 数 | 关注点 |
|---|---|---|---|
| 红线 | `UND-RED-*` | 4 | 错别字、字数 ≥ 15,000、软件系统六要素完整、参考文献 ≥ 20 |
| 学术质量 | `UND-ENG-A-*` | 10 | 摘要 / 绪论 / 工作量 / 创新性 / 引用规范 等 |
| 结构内容 | `UND-ENG-B-*` | 10 | 目录、UML、需求 / 架构 / 模块 / 数据库 / 代码 / 截图 六要素细节 |
| 格式细节 | `UND-ENG-C-*` | 4 | 代码呈现（等宽 + 1×1 表格）、图表编号、行间距、参考文献格式 |
| 参考文献 | `UND-ENG-F-*` | 5 | 双向引用抽样 / 编程书 + arXiv 上限 / 英文文献门槛 + 防伪 / [N] 编号规范 / 非同行评议来源处理 |

每条采用统一的 yaml 结构表达：`id / 类别 / 对应 PDF 要点 / 问题 / 检查方法 / 评判标准 / 修改建议模板 / 团队论文 override`。

UND-ENG-F 系列借鉴硕士版 `nju-se-thesis-self-check/references/engineering-50.md` 中
ENG-F-01..F-05 的设计，仅在数量门槛上做本科化调整，其他规范要求与硕士版保持一致。

### `references/academic-30.md`（学术型，31 条）

适用于"方法研究 / 算法 / 模型"类论文。条目分布：

| 分类 | ID 前缀 | 数 | 备注 |
|---|---|---|---|
| 红线 | `UND-RED-*` | 3 | 同工程版去掉"六要素"那条 |
| 学术质量 | `UND-ENG-A-*` | 10 | 复用工程版 |
| 结构内容 | `UND-ENG-B-*` | 5 | 仅保留目录 / UML / 引用规范，不查六要素 |
| 格式细节 | `UND-ENG-C-*` | 3 | 不强制要求 1×1 代码表格 |
| 参考文献 | `UND-ENG-F-*` | 5 | 复用工程版，规范要求与硕士版完全一致 |
| 学术专属 | `UND-ACA-S-*` | 5 | 问题形式化 / 方法新颖性 / 基线对比 / 评价指标 / 消融实验 |

学术型论文识别两种合规子类型：A 为研究方法型（IMRaD 5 章），B 为系统研究型（多章 motivation +
challenge + solution）。子类型 B 在 `UND-ACA-S-03`（基线）与 `UND-ACA-S-05`（消融）上享受豁免，
以避免将仿 SIGCOMM / NSDI 风格的实证研究论文误判。

文件中前 18 条为"占位指引"，加载时回到 `engineering-30.md` 读取完整定义；仅 `UND-ACA-S-01..05`
五条专属条目在本文件完整定义。此设计避免规则重复维护。

### `references/diagrams.md`（软件工程图表评估清单）

步骤 8 加载。按图类型分 6 个评估清单（DGM-1~6），每条针对一类常见软工图给出 UML 规范要点 +
典型问题 + 失败模板：

| ID | 图类型 | 关键点 |
|---|---|---|
| DGM-1 | 用例图 | actor 用 stickman、use case 用椭圆、system boundary、include/extend 箭头方向 |
| DGM-2 | 4+1 架构视图 | 4 层判定：触发 grep → 声明强度 → 五视图覆盖度 → 概念混用 / 图文一致性 |
| DGM-3 | ER 图 | 实体矩形 / 属性椭圆或内嵌 / 关系菱形 / 基数标注 / 主键标识 |
| DGM-4 | 类图 | 三段式（类名+属性+方法）/ 可见性符号 +/-/# / 关联 / 聚合 / 组合 / 继承 / 实现 |
| DGM-5 | 流程图 / 活动图 | ANSI 图元语义（起止椭圆 / 处理矩形 / 判定菱形）/ 分支标注 / 流程闭合 |
| DGM-6 | 时序图 / 部署图 / 状态图 | 出现才查；lifeline / 激活条 / 节点立方体 / 状态转移 guard |

此外，文末提供"工程型论文关键图缺失检查"：用例图 / 系统架构图 / ER 或表结构图至少应有两类。

文件顶部明确规定"PDF 渲染伪影禁报"原则：字符叠影 / 字体回退属 PyMuPDF 渲染管线副作用，并非论文
问题，不写入报告，以免误导学生修改根本不存在的问题。

### `references/team-submodule-overrides.md`（团队子模块 patch）

仅当步骤 4 团队 Gate 答 yes 时加载。包含约 8 条 override，典型示例：

- `Patch 1: UND-RED-3`：团队论文允许"系统架构"以三种合规形态承载（自含整体架构 / 局部架构 + 引用
  团队报告 / 整体简述 + 模块详述），不强制要求六要素全部包含于本论文
- `Patch 2: UND-ENG-B-04`：第 2 章介绍的技术仅需在"本人模块"的实现章中被使用即可，多余技术允许
  声明为"由队友实现"
- `UND-TEAM-DECL`（追加项）：团队论文须显式声明团队总项目名与本人模块边界，否则评审难以判断
  "本论文具体覆盖哪一部分"

## scripts

### `scripts/extract_skeleton.py`

将 PDF 转换为带页码标记的 Markdown 骨架，供后续步骤 grep / Read 使用。输出示例：

```
<!-- page=12 -->
## 第 3 章 需求分析
### 3.1 功能需求
...
```

调用方式（SKILL.md 步骤 1 已封装）：

```bash
python3 scripts/extract_skeleton.py <pdf_path> --out <output.md>
```

依赖 `pdfplumber`。脚本失败 / 骨架 < 30 页 / 章节切分退化时，skill 会先告知用户骨架异常再问是否
继续，基于不完整骨架做审查会漏检。

### `scripts/extract_chunks.py`

将 PDF 按"范围"切分为独立 `.txt` 文件及一份 `chunks-index.json` 索引，供步骤 11 文风检查使用。
仅在步骤 11 实际执行时调用（lazy 切分模式：未执行文风检查则不切分）。

输出（写到 `<TMP_DIR>/chunks/`）：

```
cover-info.txt           # 封面
abstract-cn.txt          # 中文摘要
abstract-en.txt          # 英文摘要
toc.txt                  # 目录（如能识别）
ch1.txt ... chN.txt      # 各章正文
acknowledgement.txt      # 致谢（如能识别）
references.txt           # 参考文献
chunks-index.json        # 含 chunks 列表 + range_map + stats.split_failed 标志
```

调用方式：

```bash
python3 scripts/extract_chunks.py <pdf_path> --out <output_dir>
```

依赖 `pdfplumber`（复用 `extract_skeleton.py` 中的章节切分逻辑）。章节切分失败时，
`chunks-index.json` 中 `stats.split_failed=true`，文风检查降级为骨架可见段落扫描。

## 输出形态

主报告写到 `<PDF 同目录>/<论文题目>-review.md`（同名已存在则顺延 `-review-1.md` / `-2.md`），
结构（节选）：

```markdown
# 本科论文自检报告 — 2026-05-06

**论文**：我的论文终稿.pdf
**抽取题目**：《基于大语言模型的代码补全工具设计与实现》
**类别**：工程型（已确认）
**团队论文**：否

## 一、首页摘要
| 项 | 数 |
|---|---|
| 🔴 红线命中 | 1 |
| ⚠️ 警告 | 5 |
| ✅ 通过 | 22 |
| 📊 图表警告（步骤 8，不计入总警告） | 2 |

## 二、🔴 红线条目（4 条全列）
...

## 六、软件工程图表规范性（步骤 8 产出，不计入红线/警告总数）
- 检测能力：✅ 模型支持图像输入 / 图注扫描识别 N 张图
- 各图评估：图 X-Y 类型 / 问题 / 修改建议
- 工程型关键图缺失：用例图 / 架构图 / ER 图 三类齐全

## 九、修改 checklist（待勾选）
- [ ] [UND-RED-2] 字数补到 15,000 以上
- [ ] [UND-ENG-A-07] 图 3-2 在正文未被引用，补"见图 3-2"或删图
...

## 十、文风与错别字检查（可选）
> 详细列表已写入独立副报告：`<题目>-style.md`
> 错别字 X 条 / AI 文风痕迹 Y 条（涉及 Z 类模式）
```

若执行了步骤 11，则额外产出文风副报告 `<PDF 同目录>/<论文题目>-style.md`（与主报告共用
同一下标顺延），含错别字 / 标点 / humanizer-zh 24 类模式逐条详情，以及按章节分布的命中频次表。

中间产物（骨架 / 渲染图 / chunks）统一保存于 `<PDF 同目录>/tmp/` 子目录，便于对照修改进度，
多次运行之间不会互相覆盖。

对话窗口仅输出报告路径、摘要与必修项三行，避免冗余信息淹没关键结论。

## 前置依赖

| 依赖 | 安装 | 缺失时影响 |
|---|---|---|
| Python 3 | — | 整个 skill 无法运行 |
| `pdfplumber` | `pip install pdfplumber` | 整个 skill 无法运行 |
| `PyMuPDF` | `pip install pymupdf` | 步骤 8 跳过视觉评估，仅做图注文本检查；其他步骤不受影响 |
| [humanizer-zh](https://github.com/op7418/Humanizer-zh) skill | 在 Agent 里安装该 skill | 步骤 11 跳过 AI 文风扫描，仅做错别字检查 |

`PyMuPDF` 与 `humanizer-zh` 均为 **软依赖**：未安装时仍可完整执行前 10 步主流程，仅第 6 节图表
评估和第 10 节文风扫描的功能会受到限制。

## 局限

- 不进行查重（学校系统已覆盖）
- 不检查 Demo / 代码仓库可运行性（属答辩范畴）
- 扫描版 PDF（CID 编码 / 中文字符极少）会显著降低准确率，skill 会在步骤 1 提前警告
- 不替代评审专家：本 skill 仅为 checklist 自动化，**不能** 判断研究创新性、工作量饱满程度等
  需要专家主观打分的维度
- 图表检查依赖模型的图像识别能力。能力不足的模型自动退化为图注文本检查，流程不会中断；
  但 UML 图形元素（actor 是否绘制为 stickman、箭头方向是否正确等）仍需人工抽查
- 文风检查会调用 humanizer-zh 逐章扫描，单次执行的 token 消耗与论文长度成正比；人工模式下
  会先告知预估耗时再询问是否继续

## 贡献

欢迎以 issue 与 PR 的形式参与贡献：

- **Bug / 误报**：附上脱敏后的 PDF 片段或骨架文本，并说明哪一条被误判为"通过"或"不合格"
- **新增条目**：在 `references/` 下对应文件添加 yaml 条目，遵循现有 `id / 类别 / 检查方法 /
  评判标准 / 修改建议模板` 结构；ID 沿用 `UND-RED-* / UND-ENG-A/B/C-* / UND-ACA-S-*` 命名
- **其他高校模板适配**：欢迎 fork 后修改 reference，组成 `undergrad-thesis-self-check`
  的兄弟项目

提交前请确认：所修改的条目已在至少 1 篇真实论文（建议为已通过外审的优秀论文）上验证过，
以避免将"局部样本观察"当作"普适规则"。

## License

MIT License. 见 [`LICENSE`](LICENSE) 文件。

## 致谢

- Checklist 条目设计参考了南京大学软件工程专业本科毕业论文写作要求文档
  以及历年优秀论文实证。
- skill 框架基于 Anthropic [Claude Code](https://docs.claude.com/en/docs/claude-code) 的 skill 机制。
