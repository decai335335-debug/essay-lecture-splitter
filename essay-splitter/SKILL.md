---
name: essay-lecture-splitter
description: 自动拆分申论讲义 Markdown 文件为独立题目文件并转换为 PDF。当用户需要拆分申论讲义、按题目整理公考学习材料、将大型讲义 MD 按题目分成独立文件、或把申论题目批量转 PDF 时使用。支持识别「第X讲」「真题/试题」「根据给定资料」等标记，自动判断材料-题目对应关系（一对一、一对多共享、多对多独立），生成规范命名的独立文件并用 Playwright 渲染中文 PDF。只要用户提到申论、讲义拆分、题目分文件、公考材料整理，就应当使用此 skill。
---

# 申论讲义拆分与 PDF 转换

这个 skill 把一个包含多个「讲」和多个题目的申论讲义 Markdown，拆分成一个题目一个文件（带对应材料），再批量转成 PDF。

## 工作流程总览

1. **分析源文件结构** —— 找出所有「讲」和「题目」的行号
2. **判断材料-题目对应关系** —— 一对一、一对多共享、还是多对多独立
3. **按判断结果提取内容** —— 生成独立 MD 文件
4. **批量转 PDF** —— 用 Playwright 渲染中文

下面每一步都有具体做法。

## 第一步：读源文件

用 Python 读，**必须指定 `encoding='utf-8'`**，否则 Windows 下中文会乱码：

```python
import sys
sys.stdout.reconfigure(encoding='utf-8')

with open(source_path, 'r', encoding='utf-8') as f:
    content = f.read()
    lines = content.split('\n')
```

不要用 PowerShell / `cat` 之类的直接读，会有编码问题。

## 第二步：识别「讲」和「题目」的边界

### 找「讲」
每个「讲」以 `## 第X讲` 开头：

```python
for i, line in enumerate(lines):
    if '## 第' in line and '讲' in line:
        print(f'行{i+1}: {line}')
```

### 找题目
题目以 `**` 包裹且通常包含「根据」：

```python
for i, line in enumerate(lines):
    stripped = line.strip()
    if stripped.startswith('**') and '根据' in stripped:
        print(f'行{i+1}: {stripped}')
```

记录下所有「讲」和「题目」的行号，下一步要用。

## 第三步：判断材料-题目对应关系（最核心）

每个「讲」里可能出现的三种情况：

### 类型 A：一个材料 → 一个题目
讲里只有一个「真题」或「试题」块，只有一道题。**输出 1 个文件。**

### 类型 B：一个材料 → 多个题目（共享）
讲里有多个题目，但后面的题目标注「同试题1材料」「同上材料」。**合并到 1 个文件。**

识别标志：在题目附近的行里搜索 `同试题` / `同上材料` / `同.*材料`。

### 类型 C：多个材料 → 多个题目（各自独立）
讲里有「真题1」「真题2」（或「试题1」「试题2」），每个后面都跟着自己独立的材料段落。**拆成多个文件。**

### 判断逻辑伪代码

```
对于每个「讲」：
  count = 数「真题X」或「试题X」标题的数量
  if count == 1:
      → 1 个文件（类型 A）
  else:
      if 后面的题目出现「同...材料」字样:
          → 1 个文件（类型 B）
      else:
          → count 个文件（类型 C）
```

**重要**：这一步的判断直接决定最终文件数，如果不确定，先把源文件对应区段打印给用户确认再动手。

## 第四步：提取每个文件的内容

### 边界确定
对每个将生成的文件，确定：
- **材料起始行**：第一个非标题（不以 `#` 开头）的实际内容行
- **材料结束行**：题目上一行
- **题目起始行**：题目那行
- **题目结束行**：遇到 `---` 分隔符或下一个块前

找「第一个非标题行」的技巧：

```python
first_mat_line = mat_start
for i in range(mat_start - 1, mat_start + 5):
    line = lines[i].strip()
    if line and not line.startswith('#'):
        first_mat_line = i + 1
        break
```

### 提取并拼接

```python
def extract_file(q_line, mat_start, lines):
    mat_lines = []
    for i in range(mat_start - 1, q_line - 1):
        line = lines[i].strip()
        if line and not line.startswith('#'):
            mat_lines.append(line)

    q_lines = []
    for i in range(q_line - 1, min(q_line + 10, len(lines))):
        line = lines[i].strip()
        if line.startswith('---'):
            break
        if line:
            q_lines.append(line)

    return '\n'.join(mat_lines) + '\n\n---\n\n' + '\n'.join(q_lines)
```

## 第五步：文件命名

格式：`原文档大标题_序号_题型_主题.md`

- **原文档大标题**：从源文件首个 `# 标题` 取（如「2025山东省考申论系统班讲义」）
- **序号**：两位数，`01`、`02`、`03`……
- **题型**：归纳概括 / 提出对策 / 综合分析 / 应用文 / 议论文（从题干判断）
- **主题**：题目内容的 5-10 字概括

示例：
- `2025山东省考申论系统班讲义_01_归纳概括_济钢无钢发展经验.md`
- `2025山东省考申论系统班讲义_02_归纳概括_X县根治生态伤疤经验.md`

## 第六步：批量转 PDF

**不要用 pandoc**——它对中文支持差。用 Playwright：

```bash
pip install playwright
playwright install chromium
```

然后调用 `scripts/md_to_pdf.py`（见本 skill 脚本目录）。用法：

```bash
python scripts/md_to_pdf.py <输入目录> <输出目录>
```

脚本会把每个 `.md` 渲染成同名 `.pdf`，用系统中文字体（Microsoft YaHei / SimHei）。

## 与用户对齐的要点

开工前跟用户确认：
1. 源文件路径
2. 输出目录
3. 是否也要生成 PDF（有的用户只要拆 MD）
4. 如果某个「讲」的对应关系你判断不准，把那一段贴给用户确认

## 常见坑

- **编码**：Python 读写都要 `encoding='utf-8'`，Windows 终端打印中文前加 `sys.stdout.reconfigure(encoding='utf-8')`
- **路径**：Windows 路径用原始字符串 `r'C:\...'` 或正斜杠
- **材料起始**：不要从 `## 第X讲` 标题行开始抓，要跳到第一个实际内容
- **共享材料的识别**：「同上材料」「同试题1材料」都要覆盖
- **PDF 中文**：用 Playwright + 中文字体 CSS，不要用 pandoc 默认 LaTeX 引擎
