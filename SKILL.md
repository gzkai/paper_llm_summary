---
name: arxiv_src
description: 用于精读 arxiv 论文。当用户提供 arxiv 链接（如 https://arxiv.org/abs/xxxx 或 https://arxiv.org/pdf/xxxx）时触发。工作流程：下载论文 LaTeX 源码压缩包 → 解压 → 读取 .tex 文件内容 → 精读并生成结构化中文总结 → 保存总结文件并提供下载链接。不直接读取 PDF，而是优先读取 LaTeX 源码以获得更准确的内容（公式、表格、章节结构）。
---

# arxiv_src Skill

## 工作流程

### 1. 提取 arxiv ID

从用户提供的链接中提取 arxiv ID（如 `2602.23978`）。

### 2. 下载源码

使用 Linux 系统命令下载并解压 LaTeX 源码：

```bash
# 1. 创建目标目录
mkdir -p /data/workspace/arxiv_papers/src/<arxiv_id>

# 2. 下载源码压缩包
curl -L -A "Mozilla/5.0" \
  "https://arxiv.org/src/<arxiv_id>" \
  -o /tmp/<arxiv_id>.tar.gz

# 3. 解压
tar -xzf /tmp/<arxiv_id>.tar.gz -C /data/workspace/arxiv_papers/src/<arxiv_id>/

# 4. 清理临时文件
rm /tmp/<arxiv_id>.tar.gz

# 5. 列出 .tex 文件（按大小降序）
find /data/workspace/arxiv_papers/src/<arxiv_id>/ -name "*.tex" \
  -exec du -k {} \; | sort -rn | head -20
```

> **注意**：若 `tar -xzf` 报错（部分源码包是 gzip 单文件而非 tar），改用：
> ```bash
> # 先用 file 命令检测类型
> file /tmp/<arxiv_id>.tar.gz
> # 若是单个 gzip 文件
> gunzip -c /tmp/<arxiv_id>.tar.gz > /data/workspace/arxiv_papers/src/<arxiv_id>/<arxiv_id>.tex
> ```

### 3. 读取源码内容

优先读取主 `.tex` 文件（通常是 `main.tex`、`paper.tex` 或文件最大的 `.tex` 文件）。

**读取策略**：
- 如果只有一个 `.tex` 文件，直接读取
- 如果有多个，先找包含 `\documentclass` 的主文件
- 若主文件用 `\input{}` 或 `\include{}` 引用其他文件，按需读取对应文件

**告知用户读取的文件**：每次精读时，在总结开头注明 "**来源文件**：`<文件路径>`"。

### 4. 生成精读总结

基于 LaTeX 源码内容，生成结构化中文总结，包含：

- **论文基本信息**：标题、作者、机构、发表场所、arxiv链接
- **研究背景与动机**：问题定义、现有方法不足
- **核心方法**：关键技术、算法、模型架构（尽可能详细地保留原文信息，保留重要公式）
- **实验结果**：数据集、对比方法、核心指标（用表格呈现）
- **核心贡献总结**
- **未来工作**（如有）

**公式格式要求**：
- 行内公式使用：$` a=1 `$
- 块级公式使用：
``` math
  a = 1
```
优先使用最强兼容版格式，保证兼容github网站的显示

### 5. 保存总结

将总结保存到：`/data/workspace/arxiv_papers/summaries/<arxiv_id>_summary.md`

然后使用 `display_download_links` 工具提供下载链接。

---

## 注意事项

- 若源码下载失败（如论文未提供源码），回退到读取 PDF（使用 pdfplumber）
- 源码解压后可能包含图片、bib 文件等，只需关注 `.tex` 文件
- 保持总结文件与 PDF 精读格式一致，方便用户对比阅读
