# undergrad-thesis-self-check

> 南京大学软件学院 / 智能软件与工程学院 **本科毕业论文** 送审前自检 skill —— 一个为
> [Claude Code](https://docs.claude.com/en/docs/claude-code) 编写的领域 skill。

把论文 PDF 喂给 Claude，按 28（工程型）/ 26（学术型）条 checklist 逐项审查，输出一份
带页码定位、按"红线 / 警告 / 通过"分级的 Markdown 修改清单——帮你在送外审之前自己先把
能查的问题都查掉。

**适用对象**：

- 主：南大软件学院 / 智能软件与工程学院本科生
- 次：其他高校工科本科生——大部分条目（字数、参考文献、UML、图表编号、引用规范、消融实验等）
  是通用的；只有"软件系统六要素"等少数条目带 NJU SE 模板特征，可参考性 > 直接使用

## 用途

学生写完论文终稿、准备送外审之前，把 PDF 路径喂给 Claude，由 skill 模拟评审专家视角，按
28（工程型）/ 26（学术型）条 checklist 逐项审查，输出一份带页码定位的 Markdown 修改清单。

设计目标是 **降低"被外审一票否决 / 打回大改"的概率**，而不是替代答辩或学校查重系统。


## 何时用 / 何时不用

适合：
- 论文已写完终稿，准备打印 / 上传送审系统之前的自查
- 想知道哪些是"红线"（一票否决项），需要优先修

不适合：
- 论文还在初稿阶段——结构没定型时跑 checklist 只会刷出一堆假阳性
- 想做查重 / 抄袭检测——学校系统已经做了，本 skill 不重复
- 想检查代码仓库能不能跑、Demo 是否正常——这是答辩范畴

## 安装

本 skill 需要 [Claude Code](https://docs.claude.com/en/docs/claude-code) 作为运行环境。
Claude Code 的 skill 默认从 `~/.claude/skills/` 加载，clone 到该目录即可：

```bash
mkdir -p ~/.claude/skills
cd ~/.claude/skills
git clone https://github.com/dongshao/undergrad-thesis-self-check.git

# 安装 PDF 解析依赖
pip install pdfplumber
```

验证安装：

```bash
ls ~/.claude/skills/undergrad-thesis-self-check/SKILL.md
```

新开一个 Claude Code 会话后，对它说"本科论文自检 /path/to/paper.pdf"即可触发。

> **注**：当前 `SKILL.md` 步骤 1 里的脚本路径是绝对路径 `~/.claude/skills/undergrad-thesis-self-check/scripts/extract_skeleton.py`，如果你装到别的位置，需要在 SKILL.md 中改一下路径。

## 触发词

必须 **同时** 满足两个条件：
1. 用户明确说出以下触发词之一：
   - 「本科论文自检」「本科论文自查」「本科送审前自查」
   - 「我的本科论文有什么问题」「我的本科论文能过吗」
   - 「undergrad self-check」
2. 提供 PDF 路径

只看到 PDF 而没有触发词，**不要** 启动 skill——避免把无关的 PDF 当成论文跑一遍。

## 调用示例

```
本科论文自检 ~/Downloads/我的论文终稿.pdf
本科论文自检 ~/Downloads/我的论文终稿.pdf --type=engineering    # 跳过类型判定 Gate
本科论文自检 ~/Downloads/我的论文终稿.pdf --type=academic --team # 学术型 + 团队子模块
```

参数：
- `--type=engineering / academic`：跳过步骤 3 的类型判定 Gate
- `--team`：标记为团队子模块论文；不带这个参数也会在步骤 4 主动询问

## 主流程（9 步）

每步都是一个 Gate：用户不确认、关键信息缺失就停下来问，**不自动猜**。

| 步 | 名称 | 一句话说明 |
|---|---|---|
| 1 | 抽骨架 + 抽题目 | 用 `scripts/extract_skeleton.py` 把 PDF 转成带页码的 Markdown 骨架；同时按 4 策略从前 10 页抽取论文真实题目（不用文件名） |
| 2 | 题目确认 Gate | 把抽到的题目给学生确认，避免封面 OCR 错误污染后续报告 |
| 3 | 类型判定 Gate | 按摘要 + 章节结构信号判别工程型 / 学术型，列出依据请学生确认；带 `--type` 时跳过 |
| 4 | 团队子模块 Gate | 默认主动问"是否团队论文"；如是，追问团队总项目名 + 本人模块名 |
| 5 | 加载 reference + overrides | 按类型读对应 checklist；团队论文额外加载 `team-submodule-overrides.md` 做条目 patch |
| 6 | 红线扫描 | 工程 4 条 / 学术 3 条红线先跑一遍——这些是评审一票否决项 |
| 7 | 全量审查 | 剩余 ~25 条按"评判 + 定位 + 修改建议"模式逐条过 |
| 8 | 写报告 | 写到 `<PDF 同目录>/<论文题目>-review.md`，同名已存在则加时间戳后缀避免覆盖 |
| 9 | 对话输出摘要 | 只贴报告路径 + 类型 + 红线数量 + 必修项；详细分项不刷屏 |

## 目录结构

```
undergrad-thesis-self-check/
├── README.md                           # 本文件
├── SKILL.md                            # skill 主入口（Claude 启动时读这个）
├── references/
│   ├── engineering-30.md               # 工程型 28 条 checklist（4 红线 + 10 学术质量 + 10 结构 + 4 格式）
│   ├── academic-30.md                  # 学术型 26 条 checklist（3 红线 + 10 学术质量 + 5 结构 + 3 格式 + 5 学术专属）
│   └── team-submodule-overrides.md     # 团队子模块论文的条目 patch（约 8 条 override）
└── scripts/
    └── extract_skeleton.py             # PDF → 带页码的 Markdown 骨架抽取脚本（依赖 pdfplumber）
```

## references 清单

skill 运行时按论文类型 + 是否团队动态加载对应文件，主入口 `SKILL.md` 不重复写 checklist 内容。

### `references/engineering-30.md`（工程型，28 条）

适用于"系统 / 平台 / 工具的设计与实现"类论文。条目分布：

| 分类 | ID 前缀 | 数 | 关注点 |
|---|---|---|---|
| 红线 | `UND-RED-*` | 4 | 错别字、字数 ≥ 15,000、软件系统六要素完整、参考文献 ≥ 20 |
| 学术质量 | `UND-ENG-A-*` | 10 | 摘要 / 绪论 / 工作量 / 创新性 / 引用规范 等 |
| 结构内容 | `UND-ENG-B-*` | 10 | 目录、UML、需求 / 架构 / 模块 / 数据库 / 代码 / 截图 六要素细节 |
| 格式细节 | `UND-ENG-C-*` | 4 | 代码呈现（等宽 + 1×1 表格）、图表编号、行间距、参考文献格式 |

每条用统一的 yaml 结构表达：`id / 类别 / 对应 PDF 要点 / 问题 / 检查方法 / 评判标准 / 修改建议模板 / 团队论文 override`。

### `references/academic-30.md`（学术型，26 条）

适用于"方法研究 / 算法 / 模型"类论文。条目分布：

| 分类 | ID 前缀 | 数 | 备注 |
|---|---|---|---|
| 红线 | `UND-RED-*` | 3 | 同工程版去掉"六要素"那条 |
| 学术质量 | `UND-ENG-A-*` | 10 | 复用工程版 |
| 结构内容 | `UND-ENG-B-*` | 5 | 仅保留目录 / UML / 引用规范，不查六要素 |
| 格式细节 | `UND-ENG-C-*` | 3 | 不强制要求 1×1 代码表格 |
| 学术专属 | `UND-ACA-S-*` | 5 | 问题形式化 / 方法新颖性 / 基线对比 / 评价指标 / 消融实验 |

学术型识别 **两种合规子类型**——A: 研究方法型（IMRaD 5 章）、B: 系统研究型（多章 motivation +
challenge + solution）。子类型 B 在 `UND-ACA-S-03`（基线）和 `UND-ACA-S-05`（消融）上享受豁免，
避免把仿 SIGCOMM/NSDI 风格的实证研究论文误判。

文件中前 18 条是"占位指引"——加载时回到 `engineering-30.md` 读完整定义；只有 `UND-ACA-S-01..05`
五条专属条目在本文件完整定义。这样避免规则重复维护。

### `references/team-submodule-overrides.md`（团队子模块 patch）

仅当步骤 4 团队 Gate 答 yes 时加载。包含约 8 条 override，典型如：

- `Patch 1: UND-RED-3` — 团队论文允许"系统架构"以三种合规形态承载（自含整体架构 / 局部架构 + 引用
  团队报告 / 整体简述 + 模块详述），不强制要求六要素全在本论文里
- `Patch 2: UND-ENG-B-04` — 第 2 章介绍的技术只需在"本人模块"的实现章被用即可，多出的允许声明
  "由队友实现"
- `UND-TEAM-DECL`（追加项）— 团队论文必须显式声明团队总项目名 + 本人模块边界，否则评审会困惑
  "你写的到底是哪部分"

## scripts

### `scripts/extract_skeleton.py`

把 PDF 转成带页码标记的 Markdown 骨架，供后续步骤 grep / Read。输出形如：

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
继续——基于不完整骨架做审查会漏检。

## 输出形态

报告写到 `<PDF 同目录>/<论文题目>-review.md`，结构（节选）：

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

## 二、🔴 红线条目（4 条全列）
...

## 八、修改 checklist（待勾选）
- [ ] [UND-RED-2] 字数补到 15,000 以上
- [ ] [UND-ENG-A-07] 图 3-2 在正文未被引用，补"见图 3-2"或删图
...
```

对话里只贴路径 + 摘要 + 必修项 3 行，避免刷屏。

## 前置依赖

- Python 3
- `pdfplumber`：`pip install pdfplumber`

## 局限

- 不做查重（学校系统已经做了）
- 不做 Demo / 代码仓库可运行性校验（答辩范畴）
- 扫描版 PDF（CID 编码 / 中文字符极少）会大幅降低准确率，skill 会在步骤 1 提前警告
- 不替代评审专家——本 skill 是 checklist 自动化，**不能** 判断研究创新性 / 工作量饱满程度等
  需要专家主观打分的维度

## 贡献

欢迎 issue 和 PR：

- **Bug / 误报**：附上脱敏后的 PDF 片段或骨架文本，说明哪一条被误判成"通过"或"不合格"
- **新增条目**：在 `references/` 下对应文件加 yaml 条目，遵循现有 `id / 类别 / 检查方法 /
  评判标准 / 修改建议模板` 结构；ID 沿用 `UND-RED-* / UND-ENG-A/B/C-* / UND-ACA-S-*` 命名
- **其他高校模板适配**：欢迎 fork 后改 reference，做成 `undergrad-thesis-self-check`
  的兄弟项目

提交前请确认：你修改的条目在至少 1 篇真实论文（建议是已通过外审的优秀论文）上验证过，避免
把"局部样本观察"当成"普适规则"。

## License

MIT License. 见 [`LICENSE`](LICENSE) 文件。

## 致谢

- Checklist 条目设计参考了南京大学软件学院 / 智能软件与工程学院本科毕业论文写作要求文档
  以及历年优秀论文实证。
- skill 框架基于 Anthropic [Claude Code](https://docs.claude.com/en/docs/claude-code) 的
  skill 机制。
