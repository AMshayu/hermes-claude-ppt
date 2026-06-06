---
name: hermes-claude-ppt
description: "Use Hermes Agent + Claude Code CLI to generate animated PPT + PDF with scenario-adaptive styles"
tags: [ppt, claude-code, pdf, presentation, hermes-agent]
related_skills: [claude-code]
---

# 用 Claude Code 制作 PPT

## 概述

两种模式：
- **新建**：用户说"做个 PPT"，Claude Code 从零生成 PPTX + PDF
- **修改**：用户说"改一下 XXX PPT"，Claude Code 读取现有 PPTX，按要求修改后重新输出

## 新建流程

```
用户需求 → 识别场景 → 拼场景化 prompt → claude -p 执行 → 桌面生成 .pptx + .pdf
```

## 修改流程

```
用户修改要求 + 目标文件路径 → 拼修改 prompt → claude -p 执行 → 覆盖生成 .pptx + .pdf
```

修改时 prompt 需额外告知 Claude：
1. 原始 .pptx 文件的完整路径
2. 具体要改什么（内容/样式/动画/增删页面）
3. Claude 会先用 python-pptx 读取现有 PPT 结构，再针对性修改，而不是从零重建

注意：修改模式会覆盖原文件。如果用户没说保留原件，直接覆盖即可。

## 命令

```bash
cd "/mnt/c/Users/27570/Desktop" && \
"/mnt/c/Users/27570/AppData/Local/Microsoft/WinGet/Links/claude.exe" \
  -p "<PROMPT>" \
  --dangerously-skip-permissions \
  --max-turns 20
```

- `cd` 到桌面先 — Claude Code CLI 不支持 `--workdir` 参数（2026-06 实测会报 unknown option），必须先 cd 再执行
- `--dangerously-skip-permissions` — 必须，否则弹确认框卡住
- `--max-turns 20` — PPT+PDF 需要多轮（装依赖→写脚本→运行→修bug→转PDF）

## 场景识别与风格系统

**Hermes 在拼 prompt 前，先根据用户需求判断属于哪个场景，然后注入对应的风格指令。**

### 场景定义

| 场景 | 关键词/触发条件 | 动画风格 | 配色倾向 | 排版特点 |
|------|----------------|----------|----------|----------|
| **学术报告** | 论文、学术、答辩、研究、课题、thesis | 克制入场（fade），强调逻辑流，章节标题用 wipe | 深蓝/白/灰，低饱和 | 信息密度高，留白少，字号偏小 |
| **晨会/周报** | 晨会、周会、日报、周报、standup、总结 | 快速切换（push/wipe），每页停留短，数据图表强调 | 亮色系（蓝/绿/橙），活力感 | 一页一要点，大字号，bullet 精简 |
| **商业提案** | 商业计划、融资、BP、提案、pitch、合作 | 专业 fade + 适度 fly，关键数字用 growShrink 强调 | 深色背景+亮色点缀（黑金/深蓝橙） | 封面冲击力强，数据页图表为主 |
| **教学课件** | 课件、教案、教学、课程、PPT课件 | 平缓 fade，知识点逐步浮现，适合 float 效果 | 浅色背景（白/浅蓝），友好感 | 标题+正文结构，关键词加粗高亮 |
| **创意展示** | 创意、设计、作品集、portfolio、灵感 | 大胆 fly + float，转场用 split，节奏感强 | 高对比撞色，渐变，不拘一格 | 打破网格，非对称排版 |
| **通用/默认** | 以上都不匹配 | fade 为主，适度 wipe | 中性色（蓝白灰），干净专业 | 平衡信息密度与可读性 |

### 场景风格注入规则

在 prompt 的 `## PPT 制作规范` 部分，根据识别到的场景追加以下内容：

```
## 场景风格：<场景名>
- 动画：<该场景的动画指导>
- 配色：<该场景的配色指导>
- 排版：<该场景的排版指导>
- 节奏：<每页建议停留时间/动画间隔>
```

**示例——学术报告场景注入：**
```
## 场景风格：学术报告
- 动画：以 fade 入场为主，章节标题可用 wipe 从左到右展开，数据图表用 fade+延迟依次出现。避免花哨的 fly/float。
- 配色：主色 #1B3A5C（深蓝），辅色 #F5F7FA（浅灰白），强调色 #2E86DE（亮蓝）。背景白色或极浅灰。
- 排版：信息密度适中，标题 28-36pt，正文 18-22pt，行距 1.3。左侧可留窄栏放章节编号。
- 节奏：动画间隔 300-500ms，转场用 fade 1200ms，适合逐条讲解。
```

**示例——晨会总结场景注入：**
```
## 场景风格：晨会总结
- 动画：快速 wipe 或 push 入场，关键数字用 growShrink 缩放强调。整体节奏快。
- 配色：主色 #27AE60（绿）或 #3498DB（蓝），背景白，重点用 #E74C3C（红）标注风险/阻塞。
- 排版：一页一个主题，大标题 + 3-5 条精简 bullet，字号 24-32pt。数据用色块/进度条可视化。
- 节奏：动画间隔 200ms，转场用 push 800ms，每页停留时间短，适合快速过。
```

**如果用户明确指定风格（如"我想要深色科技风"），以用户为准，场景风格仅作为默认建议。**

## 新建 Prompt 模板

```
你是一个 PPT 制作专家。请完成以下任务：

## 需求
<把用户的主题、内容、风格要求原样填在这里>

## 输出要求
1. 在当前目录（桌面）生成 <文件名>.pptx
2. 同时在桌面生成对应的 <文件名>.pdf

## PPT 制作规范
- 使用 python-pptx，先 pip install python-pptx lxml
- 幻灯片尺寸 13.333×7.5 英寸（16:9）
- 中文内容必须使用 Microsoft YaHei 字体
- 动画：用 lxml 注入 OOXML XML（参考下方代码），为标题和关键元素添加入场动画
- 根据下方场景风格指令选择合适的动画效果和配色方案

<在此注入场景风格指令——从上面"场景风格注入规则"中选择匹配的场景>

## PDF 转换规范（重要！必须按此执行）

**不要尝试以下方法（全部会失败）：**
- ❌ PowerShell COM 自动化（New-Object PowerPoint.Application）— 会超时或 COM 错误
- ❌ pywin32 的 win32com.client — COM 启动失败
- ❌ comtypes — 同样 COM 错误
- ❌ LibreOffice soffice — 未安装

**正确方案：用 fpdf2 直接生成 PDF（不经过 PPTX 转换）**

```bash
pip install fpdf2
```

用 fpdf2 独立生成 PDF，内容和布局与 PPTX 保持一致，但不包含动画。
fpdf2 默认不支持中文，需要加载 Windows 字体：

```python
from fpdf import FPDF

class SlidePDF(FPDF):
    def __init__(self):
        super().__init__(orientation='L', unit='mm', format=(190.5, 338.67))  # 16:9

    def slide_bg(self, r, g, b):
        self.set_fill_color(r, g, b)
        self.rect(0, 0, self.w, self.h, 'F')

    def load_chinese_font(self):
        self.add_font('yahei', '', 'C:\\Windows\\Fonts\\msyh.ttc', uni=True)
        self.add_font('yahei', 'B', 'C:\\Windows\\Fonts\\msyhbd.ttc', uni=True)

pdf = SlidePDF()
pdf.load_chinese_font()
pdf.add_page()
pdf.set_font('yahei', '', 24)
pdf.set_text_color(255, 255, 255)
# ... 用 pdf.cell() / pdf.multi_cell() 排版
pdf.output('output.pdf')
```

## 动画辅助代码

在 PPT 生成脚本中使用以下函数为元素添加入场动画。根据场景风格选择合适的 effect 参数：

- **fade**（淡入）：万能，学术/通用场景首选
- **fly**（飞入）：适合创意展示、强调出场
- **wipe**（擦除）：适合晨会、章节标题
- **float**（浮动上升）：适合教学课件的知识点浮现
- **growShrink**（缩放）：适合数据强调、关键数字

```python
from pptx.oxml.ns import qn
from lxml import etree

def set_shape_alpha(shape, alpha_pct):
    solid = shape._element.find('.//' + qn('a:solidFill'))
    if solid is not None and len(solid) > 0:
        clr = solid[0]
        alpha_elem = clr.find(qn('a:alpha'))
        if alpha_elem is None:
            alpha_elem = etree.SubElement(clr, qn('a:alpha'))
        alpha_elem.set('val', str(int((100 - alpha_pct) * 1000)))

def add_appear_animation(slide, shape, delay_ms=0, duration_ms=800, effect='fade'):
    sp_id = shape.shape_id
    timing = slide.element.find(qn('p:timing'))
    if timing is None:
        timing = etree.SubElement(slide.element, qn('p:timing'))
    tnLst = timing.find(qn('p:tnLst'))
    if tnLst is None:
        tnLst = etree.SubElement(timing, qn('p:tnLst'))
    par = tnLst.find(qn('p:par'))
    if par is None:
        par = etree.SubElement(tnLst, qn('p:par'))
        cTn = etree.SubElement(par, qn('p:cTn'))
        cTn.set('id', '1'); cTn.set('dur', 'indefinite')
        cTn.set('restart', 'never'); cTn.set('nodeType', 'tmRoot')
        childTnLst = etree.SubElement(cTn, qn('p:childTnLst'))
        seq = etree.SubElement(childTnLst, qn('p:seq'))
        seq.set('concurrent', '1'); seq.set('nextAc', 'seek')
        seqCtn = etree.SubElement(seq, qn('p:cTn'))
        seqCtn.set('id', '2'); seqCtn.set('dur', 'indefinite')
        seqCtn.set('nodeType', 'mainSeq')
        etree.SubElement(seqCtn, qn('p:childTnLst'))
        etree.SubElement(seq, qn('p:prevCondLst'))
        etree.SubElement(seq, qn('p:nextCondLst'))
    mainSeq = par.find('.//' + qn('p:cTn') + '[@nodeType="mainSeq"]')
    seqChildTnLst = mainSeq.find(qn('p:childTnLst'))
    par_id = 100 + len(seqChildTnLst) * 3
    par_anim = etree.SubElement(seqChildTnLst, qn('p:par'))
    cTn1 = etree.SubElement(par_anim, qn('p:cTn'))
    cTn1.set('id', str(par_id)); cTn1.set('fill', 'hold')
    stCondLst = etree.SubElement(cTn1, qn('p:stCondLst'))
    etree.SubElement(stCondLst, qn('p:cond')).set('delay', '0')
    childTnLst1 = etree.SubElement(cTn1, qn('p:childTnLst'))
    par2 = etree.SubElement(childTnLst1, qn('p:par'))
    cTn2 = etree.SubElement(par2, qn('p:cTn'))
    preset_map = {'fade':'10','fly':'2','wipe':'22','float':'57','growShrink':'44'}
    subtype_map = {'fade':'0','fly':'4','wipe':'4','float':'1','growShrink':'0'}
    cTn2.set('id', str(par_id+1))
    cTn2.set('presetID', preset_map.get(effect,'10'))
    cTn2.set('presetClass', 'entr')
    cTn2.set('presetSubtype', subtype_map.get(effect,'0'))
    cTn2.set('fill', 'hold'); cTn2.set('grpId', '0')
    cTn2.set('nodeType', 'clickEffect' if delay_ms==0 else 'afterEffect')
    stCondLst2 = etree.SubElement(cTn2, qn('p:stCondLst'))
    etree.SubElement(stCondLst2, qn('p:cond')).set('delay', str(delay_ms))
    childTnLst2 = etree.SubElement(cTn2, qn('p:childTnLst'))
    set_el = etree.SubElement(childTnLst2, qn('p:set'))
    cBhvr1 = etree.SubElement(set_el, qn('p:cBhvr'))
    cTn3 = etree.SubElement(cBhvr1, qn('p:cTn'))
    cTn3.set('id', str(par_id+2)); cTn3.set('dur', '1'); cTn3.set('fill', 'hold')
    tgtEl1 = etree.SubElement(cBhvr1, qn('p:tgtEl'))
    etree.SubElement(tgtEl1, qn('p:spTgt')).set('spid', str(sp_id))
    to_el = etree.SubElement(set_el, qn('p:to'))
    etree.SubElement(to_el, qn('p:strVal')).set('val', 'visible')
    animEffect = etree.SubElement(childTnLst2, qn('p:animEffect'))
    animEffect.set('transition', 'in'); animEffect.set('filter', 'fade')
    cBhvr2 = etree.SubElement(animEffect, qn('p:cBhvr'))
    cTn4 = etree.SubElement(cBhvr2, qn('p:cTn'))
    cTn4.set('id', str(par_id+3)); cTn4.set('dur', str(duration_ms))
    tgtEl2 = etree.SubElement(cBhvr2, qn('p:tgtEl'))
    etree.SubElement(tgtEl2, qn('p:spTgt')).set('spid', str(sp_id))
    if effect == 'float':
        motion = etree.SubElement(childTnLst2, qn('p:anim'))
        motion.set('calcmode','lin'); motion.set('valueType','pt')
        cBhvr_m = etree.SubElement(motion, qn('p:cBhvr'))
        cTn_m = etree.SubElement(cBhvr_m, qn('p:cTn'))
        cTn_m.set('id', str(par_id+4)); cTn_m.set('dur', str(duration_ms))
        tgtEl_m = etree.SubElement(cBhvr_m, qn('p:tgtEl'))
        etree.SubElement(tgtEl_m, qn('p:spTgt')).set('spid', str(sp_id))
        attrLst = etree.SubElement(cBhvr_m, qn('p:attrNameLst'))
        etree.SubElement(attrLst, qn('p:attrName')).text = 'ppt_y'
        tavs = etree.SubElement(motion, qn('p:tavLst'))
        t1 = etree.SubElement(tavs, qn('p:tav')); t1.set('tm','0')
        etree.SubElement(etree.SubElement(t1, qn('p:val')), qn('a:fltVal')).set('val','80000')
        t2 = etree.SubElement(tavs, qn('p:tav')); t2.set('tm','100000')
        etree.SubElement(etree.SubElement(t2, qn('p:val')), qn('a:fltVal')).set('val','0')

def add_slide_transition(slide, transition_type='fade', duration_ms=1200):
    transition = slide.element.find(qn('p:transition'))
    if transition is None:
        transition = etree.SubElement(slide.element, qn('p:transition'))
    transition.set('spd','med'); transition.set('advClick','1')
    transition.set('advTm', str(duration_ms))
    for child in list(transition): transition.remove(child)
    tag = {'fade':qn('p:fade'),'push':qn('p:push'),'wipe':qn('p:wipe'),
           'split':qn('p:split')}.get(transition_type, qn('p:fade'))
    etree.SubElement(transition, tag)
```

## 修改 Prompt 模板

```
你是一个 PPT 制作专家。现在需要修改一个已有的 PPT。

## 修改目标
文件路径：<桌面路径/文件名.pptx>

## 修改要求
<把用户的具体修改要求原样填在这里>

## 执行步骤
1. 先用 python-pptx 读取现有 PPTX，理解其结构（页面数量、每页内容、配色、动画）
2. 按修改要求进行针对性修改，保留未提及的部分不变
3. 覆盖保存原 .pptx 文件
4. 用 fpdf2 重新生成对应的 .pdf 文件

## 技术规范
- 使用 python-pptx 读取和修改（pip install python-pptx lxml）
- 中文字体使用 Microsoft YaHei
- PDF 用 fpdf2 独立生成（pip install fpdf2），不经过 PPTX 转换
- fpdf2 加载中文字体：self.add_font('yahei', '', 'C:\\Windows\\Fonts\\msyh.ttc', uni=True)
- 修改后保持原文件的动画风格一致
- 幻灯片尺寸 13.333x7.5 英寸（16:9）
```

## 完成后

1. 验证桌面上 .pptx 和 .pdf 都存在且大小合理
2. 清理临时脚本文件
3. 输出结果摘要（文件名、页数、使用的场景风格）
