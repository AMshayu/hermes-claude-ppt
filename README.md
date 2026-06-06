<div align="right">

[中文](#中文) | [English](#english)

</div>

---

# hermes-claude-ppt

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

<a name="中文"></a>

## 中文

用 [Hermes Agent](https://hermes-agent.nousresearch.com) 调度 Claude Code CLI，根据场景自适应风格，生成和修改带动画的 PPT 和 PDF。

### 为什么做这个

Hermes Agent 是优秀的私人管家，但直接让它生成 PPT 效果有限；Claude Code CLI 有强大的代码生成能力，但手动调用繁琐、流程难以自动化。

本 Skill 将两者桥接——让 Hermes 负责意图理解、**场景识别**和流程编排，Claude Code 负责高质量代码执行，各取所长。一句话说清：**让"说一句话就能拿到专业 PPT"真正可用，而且不同场景自动匹配不同风格。**

### 工作流程

**新建：**
```
用户需求 → Hermes 识别场景 → 注入场景风格 → Claude Code 执行 → 桌面输出 .pptx + .pdf
```

**修改：**
```
用户修改要求 + 目标文件 → Hermes 拼 prompt → Claude Code 读取并修改 → 覆盖输出 .pptx + .pdf
```

### 场景自适应风格

不同场景需要不同的视觉语言。Hermes 会根据用户描述自动判断场景，注入对应的动画风格、配色方案和排版指导：

| 场景 | 触发关键词示例 | 动画风格 | 配色倾向 |
|------|--------------|----------|----------|
| **学术报告** | 论文、答辩、研究、课题 | 克制 fade，章节 wipe 展开 | 深蓝/白/灰，低饱和 |
| **晨会/周报** | 晨会、周报、日报、standup | 快速 wipe/push，数据强调 | 亮色系（绿/蓝/橙） |
| **商业提案** | 商业计划、融资、BP、pitch | 专业 fade + 关键数字缩放 | 深色背景+亮色点缀 |
| **教学课件** | 课件、教案、教学、课程 | 平缓 fade + float 浮现 | 浅色背景，友好感 |
| **创意展示** | 创意、设计、作品集、灵感 | 大胆 fly + float + split | 高对比撞色 |
| **通用** | 以上都不匹配 | fade 为主 | 蓝白灰，干净专业 |

用户明确指定风格时（如"我要深色科技风"），以用户为准，场景仅作为默认建议。

### 这是什么

一个 Hermes Agent Skill。用户用自然语言描述 PPT 需求或修改要求，Hermes 自动识别场景并调用 Claude Code CLI 生成或修改专业 PPT（带 OOXML 动画）和对应的 PDF 文件。

系统几乎处于"出厂状态"——仅需安装 Hermes Agent 和 Claude Code CLI，无需额外配置复杂环境。所有 Python 依赖（python-pptx、lxml、fpdf2）在首次执行时由 Claude Code 自动安装。

### 技术栈

| 组件 | 技术 |
|---|---|
| PPT 生成 | python-pptx + lxml（OOXML 动画注入） |
| PDF 生成 | fpdf2（独立生成，不经过 PPTX 转换） |
| 执行引擎 | Claude Code CLI（`claude -p`） |
| 场景识别 | Hermes Agent 内置 LLM 意图理解 |

### 为什么用 fpdf2 转 PDF

直接从 PPTX 转 PDF 在 Windows 环境下几乎不可能自动化，经过实测全部失败：

| 方案 | 结果 |
|---|---|
| PowerShell COM（`New-Object PowerPoint.Application`） | ❌ 超时 / COM 错误 |
| pywin32（`win32com.client`） | ❌ COM 启动失败 |
| comtypes | ❌ 同样 COM 错误 |
| LibreOffice soffice | ❌ 未安装 / 格式兼容问题 |
| **fpdf2 独立生成** | ✅ 稳定可靠 |

### 前置条件

1. [Hermes Agent](https://hermes-agent.nousresearch.com) 已安装并运行
2. [Claude Code CLI](https://code.claude.com) 已安装并认证
3. Python + pip 可用
4. Windows 环境（WSL 或原生均可）

### 安装

```bash
# 克隆到 Hermes skills 目录
git clone https://github.com/<你的用户名>/hermes-claude-ppt.git \
  ~/.hermes/skills/productivity/hermes-claude-ppt
```

安装完成后，Hermes 会在你提到"PPT"相关需求时自动加载此 Skill，无需手动配置。

**验证安装：**

```bash
# 检查 skill 文件是否存在
ls ~/.hermes/skills/productivity/hermes-claude-ppt/SKILL.md
```

或在 Hermes 对话中说："帮我做个测试 PPT"，能正常生成即表示安装成功。

### 使用

**新建 PPT：**

> "帮我做个 PPT，主题是 XXX"

Hermes 会自动判断场景（学术/晨会/商业/教学/创意/通用），匹配对应的动画和配色风格。

**修改已有 PPT：**

> "把 XXX.pptx 第三页的标题改成 YYY"
> "给那个 PPT 加一页总结"

Hermes 会自动加载此 Skill，调用 Claude Code 完成制作或修改，桌面上会出现更新后的 `.pptx` 和 `.pdf` 两个文件。

### 依赖

Claude Code 执行时会自动安装：

- `python-pptx` + `lxml` — PPT 生成 + 动画
- `fpdf2` — PDF 生成

### 局限性

- PDF 由 fpdf2 独立排版，布局与 PPTX 视觉上可能存在细微差异（字号、对齐等）
- 中文字体依赖 Windows 系统自带的微软雅黑（msyh.ttc），跨平台需自行替换字体路径
- 动画仅在 .pptx 中生效，PDF 不含动画
- 场景识别基于 LLM 理解，偶尔可能误判，用户可明确指定风格覆盖

### 许可证

MIT License

---

<a name="english"></a>

## English

A [Hermes Agent](https://hermes-agent.nousresearch.com) skill that orchestrates Claude Code CLI to generate and modify animated PPT presentations with **scenario-adaptive styling** and PDF exports.

### Why this exists

Hermes Agent is a great personal assistant, but its native PPT generation has quality limits. Claude Code CLI is powerful at code generation, but orchestrating it manually is tedious and error-prone.

This skill bridges the two: Hermes handles intent parsing, **scenario detection**, and workflow orchestration; Claude Code handles high-quality execution. One sentence: **turn "describe your PPT in plain language" into a reliable, end-to-end workflow — with automatic style adaptation for different contexts.**

### Workflow

**Create:**
```
User request → Hermes detects scenario → Injects style guidance → Claude Code executes → Desktop output .pptx + .pdf
```

**Modify:**
```
User modification request + target file → Hermes builds prompt → Claude Code reads and modifies → Overwrites .pptx + .pdf
```

### Scenario-Adaptive Styling

Different contexts need different visual languages. Hermes automatically detects the scenario from the user's description and injects matching animation styles, color palettes, and layout guidance:

| Scenario | Example triggers | Animation style | Color palette |
|---|---|---|---|
| **Academic** | thesis, defense, research, paper | Restrained fade, section wipe | Navy/white/gray, low saturation |
| **Standup/Report** | standup, weekly, daily, sprint | Fast wipe/push, data emphasis | Bright (green/blue/orange) |
| **Business Pitch** | pitch deck, funding, BP, proposal | Professional fade + key numbers grow | Dark bg + accent pops |
| **Teaching** | slides, lecture, courseware, lesson | Gentle fade + float reveal | Light bg, friendly feel |
| **Creative** | creative, portfolio, design, showcase | Bold fly + float + split transitions | High-contrast, unconventional |
| **General** | none of the above | Fade-dominant | Blue/white/gray, clean |

When the user explicitly specifies a style (e.g. "I want a dark tech look"), user preference takes priority. The scenario is only a default suggestion.

### What is this

A Hermes Agent skill. Users describe their PPT needs or modification requests in natural language, and Hermes automatically detects the scenario and calls Claude Code CLI to generate or modify professional PPT (with OOXML animations) and corresponding PDF files.

The system is essentially a blank slate — only Hermes Agent and Claude Code CLI are required. All Python dependencies (python-pptx, lxml, fpdf2) are auto-installed on first run.

### Tech Stack

| Component | Technology |
|---|---|
| PPT generation | python-pptx + lxml (OOXML animation injection) |
| PDF generation | fpdf2 (standalone, not converted from PPTX) |
| Execution engine | Claude Code CLI (`claude -p`) |
| Scenario detection | Hermes Agent built-in LLM intent understanding |

### Why fpdf2 for PDF

Converting PPTX to PDF directly on Windows is nearly impossible to automate. All of these approaches failed:

| Approach | Result |
|---|---|
| PowerShell COM (`New-Object PowerPoint.Application`) | ❌ Timeout / COM error |
| pywin32 (`win32com.client`) | ❌ COM launch failed |
| comtypes | ❌ Same COM error |
| LibreOffice soffice | ❌ Not installed / format issues |
| **fpdf2 standalone generation** | ✅ Works reliably |

### Prerequisites

1. [Hermes Agent](https://hermes-agent.nousresearch.com) installed and running
2. [Claude Code CLI](https://code.claude.com) installed and authenticated
3. Python + pip available
4. Windows environment (WSL or native)

### Installation

```bash
# Clone into Hermes skills directory
git clone https://github.com/<your-username>/hermes-claude-ppt.git \
  ~/.hermes/skills/productivity/hermes-claude-ppt
```

After installation, Hermes will automatically load this skill when you mention PPT-related requests. No manual configuration needed.

**Verify installation:**

```bash
# Check that the skill file exists
ls ~/.hermes/skills/productivity/hermes-claude-ppt/SKILL.md
```

Or say in a Hermes conversation: "Make a test PPT". If it generates successfully, the installation is working.

### Usage

**Create a new PPT:**

> "Make a PPT about XXX"

Hermes will automatically detect the scenario (academic, standup, business, teaching, creative, or general) and apply the appropriate animation and color style.

**Modify an existing PPT:**

> "Change the title on slide 3 of XXX.pptx to YYY"
> "Add a summary slide to that PPT"

Hermes will automatically load this skill, call Claude Code to produce or modify the files. Both updated `.pptx` and `.pdf` will appear on your desktop.

### Dependencies

Auto-installed by Claude Code during execution:

- `python-pptx` + `lxml` — PPT generation + animation
- `fpdf2` — PDF generation

### Limitations

- PDF is independently typeset by fpdf2, so layout may differ slightly from the PPTX (font size, alignment, etc.)
- Chinese fonts rely on Microsoft YaHei (msyh.ttc) bundled with Windows; cross-platform usage requires replacing the font path
- Animations only work in .pptx; PDF does not include animations
- Scenario detection is LLM-based and may occasionally misread the context; users can always override by specifying a style explicitly

### License

MIT License
