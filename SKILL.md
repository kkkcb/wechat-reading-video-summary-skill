---
name: "wechat-reading-video-summary"
description: "将视频（B站等）文案转录、总结，生成微信读书风格的精美HTML阅读页面。当用户要求总结视频并生成HTML阅读页面、制作视频笔记网页时调用。"
---

# 微信读书风格视频总结 HTML 生成器

## 概述

将视频（支持B站等平台）的文案进行转录、结构化总结，并生成一套精美的微信读书风格 HTML 阅读页面。整个流程分为四个阶段：**视频转录 → 内容总结与基础知识补充 → 技术图解与深度解析 → HTML 生成**。

---

## 阶段一：视频音频转录

### 步骤

1. **下载视频音频**：使用 `yt-dlp` 下载视频的音频轨道
   ```bash
   yt-dlp -x --audio-format m4a -o "audio.m4a" "<视频URL>"
   ```
   - 先尝试获取字幕列表：`yt-dlp --list-subs "<视频URL>"`
   - 如果有中文字幕，直接下载字幕文件即可，无需转录
   - 如果没有字幕，则下载音频进行转录
   - **B站反爬降级**：B站可能返回 HTTP 412，此时改用 WebSearch 搜索 `{视频标题} 文字稿/笔记/总结`，从多个博客源综合整理内容

2. **音频转录**：使用 `faster-whisper` 将音频转为文字
   ```python
   from faster_whisper import WhisperModel
   model = WhisperModel("base", device="cpu", compute_type="int8")
   segments, info = model.transcribe("audio.m4a", language="zh", beam_size=5)
   with open("transcript.txt", "w", encoding="utf-8") as f:
       for segment in segments:
           f.write(f"[{segment.start:.1f}s - {segment.end:.1f}s] {segment.text}\n")
   ```

3. **注意事项**：
   - Whisper 语音识别可能产生谐音错误（如 "Full Harness" 误识别为 "Four Harness"），生成内容后需要用户校验专有名词
   - 转录完成后保存到临时工作目录，不要放到用户的工作空间

---

## 阶段二：内容结构化总结与基础知识补充

### 原则

1. **必须基于实际转录内容**，不能仅凭视频描述或外部文章臆造
2. 将视频内容拆解为 5-8 个章节，每个章节有明确的主题
3. 每个章节下可包含多个子话题（用于侧边栏导航）
4. 如果视频引用了外部文章或资料，应在末尾添加附录，附上原文链接和简要讲解

### ⭐ 核心原则：基础知识必须补充

> **视频总结不仅是"记录视频说了什么"，更是"让读者真正理解这些内容"。**
> 视频中提到的每一个专业术语、核心概念、关键公式，都必须附带易于理解的基础知识解释。
> 读者应该是零基础也能看懂总结的程度。

#### 必须补充基础知识的场景

| 场景 | 示例 | 补充方式 |
|------|------|----------|
| 视频提及专业术语但未解释 | "归纳偏置"、"残差连接"、"Label Smoothing" | 添加 `deep-dive-card`，给出定义、直觉、类比 |
| 视频展示公式但未推导 | Self-Attention 公式、缩放因子 √d_k | 添加 `deep-dive-card`，给出数学推导步骤 |
| 视频提及对比但未展开 | RNN vs CNN vs Self-Attention | 添加 `compare-table` 或 `compare-grid`，逐维度对比 |
| 视频展示数值结果但未说明过程 | BLEU 分数、训练参数 | 添加 `step-list` 或 `matrix-display`，展示计算过程 |
| 视频涉及可视化概念 | 位置编码波形、归一化方向、架构图 | 添加内联 SVG 技术图解 |

#### 基础知识补充的深度标准

- **定义层**：一句话说清楚这个概念是什么
- **直觉层**：用日常类比或可视化帮助理解（如 Warmup = "开车先热身再加速"）
- **原理层**：给出数学公式或计算步骤，用 `matrix-display` 展示具体数值
- **对比层**：与相关概念对比，突出差异（如 BN vs LN）
- **代价层**：说明这个设计的 trade-off（如 Label Smoothing 损害 PPL 但提升 BLEU）

### 内容结构模板

```
- 封面（标题、副标题、来源信息）
- 目录卡片（列出所有章节标题）
- 第一章：背景引入 / 概念定义
  ↳ [deep-dive-card] 关键术语的详细解释
- 第二章：核心概念讲解
  ↳ [diagram-container] 概念可视化 SVG
  ↳ [deep-dive-card] 公式推导或数值示例
- 第三章：方案A的实践案例
  ↳ [compare-table] 与其他方案的对比
- 第四章：方案B的实践案例（含详细流程）
  ↳ [diagram-container] 流程图/架构图 SVG
- 第五章：概念溯源 / 时间线
- 第六章：总结与观点
- 附录：延伸阅读（如有参考资料）
```

---

## 阶段三：技术图解与深度解析设计

### 概述

在 HTML 生成之前，先规划好需要哪些技术图解和深度解析卡片。这一阶段确保图解和解析与内容章节精准匹配。

### 技术图解规范（内联 SVG）

#### 设计原则

1. **使用内联 SVG**：直接嵌入 HTML，不依赖外部文件，保证单文件可独立运行
2. **配色使用设计系统**：`#c0793c`(accent), `#2c2418`(text), `#6b5e4f`(secondary), `#9e9182`(tertiary), `#e4ddd3`(border), `#fffcf8`(card)
3. **必须设置 viewBox**：确保响应式缩放，外部容器 `overflow-x: auto` 支持移动端滚动
4. **中文标签**：SVG 中的文字标签使用中文
5. **颜色区分语义**：不同角色/模块用不同颜色（如 Encoder 橙色、Decoder 绿色）

#### 常见图解类型

| 图解类型 | 适用场景 | SVG 要素 |
|----------|----------|----------|
| **架构图** | 模型/系统结构 | 圆角矩形模块 + 连接箭头 + 颜色分区 |
| **流程图** | 计算步骤/数据流 | 节点矩形 + 有向箭头 + 标注 |
| **对比图** | 两种方案差异 | 左右/上下分栏 + 差异高亮 |
| **矩阵可视化** | 归一化方向、张量操作 | 网格 + 箭头标注方向 + 颜色填充 |
| **波形/曲线图** | 函数可视化、趋势 | polyline/path + 坐标轴 + 图例 |
| **全连接图** | Attention、图网络 | 节点圆圈 + 全连接线 + 权重标注 |

#### SVG 模板参考

```html
<div class="diagram-container">
  <svg viewBox="0 0 780 320" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
    <!-- 圆角矩形模块 -->
    <rect x="10" y="10" width="160" height="50" rx="8" fill="#fffcf8" stroke="#e4ddd3" stroke-width="1.5"/>
    <text x="90" y="40" text-anchor="middle" fill="#2c2418" font-size="14" font-family="Noto Sans SC, sans-serif">模块名称</text>

    <!-- 连接箭头 -->
    <defs><marker id="arrow" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6" fill="#9e9182"/>
    </marker></defs>
    <line x1="170" y1="35" x2="230" y2="35" stroke="#9e9182" stroke-width="1.5" marker-end="url(#arrow)"/>
  </svg>
  <div class="diagram-caption">图注说明文字</div>
</div>
```

### 深度解析卡片规范

#### deep-dive-card

用于对视频中的关键概念进行超出视频内容的深度解释。每张卡片应包含：

1. **标题标签**：用 `deep-dive-label` 标注主题
2. **定义**：一句话定义
3. **直觉/类比**：日常类比帮助理解
4. **原理/推导**：数学公式或逐步计算
5. **对比**：与相关概念的区别
6. **代价/Trade-off**：这种设计的利弊

```html
<div class="deep-dive-card">
  <div class="deep-dive-label">概念名称</div>
  <p><strong>定义：</strong> 一句话说清楚这个概念是什么。</p>
  <p><strong>直觉：</strong> 日常类比...</p>
  <p><strong>原理：</strong> 数学推导...</p>
  <div class="matrix-display">
    [1.0, 0.0]
    [0.0, 1.0]
  </div>
  <p><strong>Trade-off：</strong> 利弊分析...</p>
</div>
```

#### matrix-display

用于展示数值计算过程的中间矩阵：

```html
<div class="matrix-display">
第 1 步：输入矩阵 X（4×2）
┌         ┐
│  1   0  │
│  0   1  │
│  1   1  │
│  0   0  │
└         ┘
</div>
```

---

## 阶段四：HTML 页面生成

### 整体结构

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{页面标题}</title>
  <!-- Google Fonts -->
  <!-- CSS 样式 -->
</head>
<body>
  <!-- 阅读进度条 -->
  <div class="progress-bar" id="progressBar"></div>

  <!-- 左侧导航侧边栏 -->
  <nav class="sidebar" id="sidebar">...</nav>
  <div class="sidebar-overlay" id="sidebarOverlay"></div>
  <div class="sidebar-toggle" id="sidebarToggle">...</div>

  <!-- 主内容区 -->
  <div class="page-wrapper">
    <main class="content">
      <!-- 封面 -->
      <!-- 目录卡片 -->
      <!-- 各章节（含技术图解 + 深度解析） -->
      <!-- 附录（如有） -->
      <!-- 页脚 -->
    </main>
  </div>

  <script>
    // 进度条、章节动画、侧边栏、滚动追踪
  </script>
</body>
</html>
```

### 设计系统（微信读书风格）

#### CSS 变量

```css
:root {
  --bg-primary: #f6f1eb;       /* 页面底色：温暖米色 */
  --bg-card: #fffcf8;          /* 卡片底色：偏白暖色 */
  --bg-sidebar: #eee8df;       /* 侧边栏底色 */
  --text-primary: #2c2418;     /* 主文字：深棕色 */
  --text-secondary: #6b5e4f;   /* 次文字：中棕色 */
  --text-tertiary: #9e9182;    /* 辅助文字：浅棕色 */
  --accent: #c0793c;           /* 强调色：琥珀橙 */
  --accent-light: rgba(192,121,60,0.12);
  --accent-dark: #a8662e;
  --border: #e4ddd3;           /* 边框色 */
  --border-light: #ede7de;
  --tag-bg: #f0e8dc;           /* 标签背景 */
  --highlight: #fff3e0;        /* 高亮背景 */
  --shadow-sm: 0 1px 3px rgba(44,36,24,0.06);
  --shadow-md: 0 4px 12px rgba(44,36,24,0.08);
  --radius: 6px;
  --radius-lg: 10px;
  --transition: 0.25s cubic-bezier(0.22, 1, 0.36, 1);
}
```

#### 字体

- 标题：`'Noto Serif SC', serif`（宋体风格，增加阅读感）
- 正文：`'Noto Sans SC', -apple-system, BlinkMacSystemFont, sans-serif`
- Google Fonts 引入：`https://fonts.googleapis.com/css2?family=Noto+Serif+SC:wght@400;600;700&family=Noto+Sans+SC:wght@300;400;500;700&display=swap`

#### 页面容器

```css
.page-wrapper {
  max-width: 820px;
  margin: 0 auto;
  padding: 0 24px;
}
```

### 必备组件

以下组件必须实现，按需在内容中使用：

#### 1. 封面区域（cover-section）
- 居中布局，包含：badge 标签、主标题（serif 32px）、副标题、meta 信息（来源/时长/日期）
- 底部有装饰性分割线（40px 宽、accent 色）

#### 2. 目录卡片（toc-card）
- 白底卡片，列出所有章节编号和标题
- 可点击跳转到对应章节

#### 3. 章节容器（chapter）
- 每个章节用 `<section class="chapter" id="chX">` 包裹
- 章节头：圆形编号 + 标题（serif 24px）
- 章节间用 `divider-accent` 分隔（渐变线条）

#### 4. 段落文字（para）
- 正文字体 15.5px，行高 1.85，颜色 text-secondary
- 段间距 16px

#### 5. 对比表格（compare-table）
- 用于展示两种方案的对比
- 表头用 accent-light 背景，数据行交替底色

#### 6. 流程图（flow-container）
- 垂直流程图，节点间用箭头连接
- 节点类型：
  - `flow-user`：用户/外部输入（橙色左边框）
  - `flow-planner`：规划角色（蓝色左边框）
  - `flow-generator`：执行角色（绿色左边框）
  - `flow-evaluator`：评估角色（红色左边框）
  - `flow-done`：完成状态（绿色背景）
- 支持循环框（flow-loop-box）：虚线边框包裹循环逻辑区域

#### 7. 阶段块（phase-block）
- 左侧彩色边框的卡片，用于展示分阶段实施流程
- 包含阶段编号、标题、正文
- 支持自定义左边框颜色

#### 8. 循环步骤图（loop-diagram）
- 编号步骤列表，用于展示循环/迭代流程
- 步骤编号用圆形标记，循环步骤用 ⟳ 符号

#### 9. 功能时间线（feature-timeline）
- 水平排列的功能点进度条
- 三种状态：done（绿色✓）、active（橙色⟳ 脉冲动画）、pending（灰色○）
- 功能点之间用连接线串联

#### 10. 对比卡片（compare-grid / compare-card）
- 两列对比布局，用于展示两种方案的差异
- 标题区分 good（绿色）和 bad（红色）标签
- 每张卡片内用 metric 行展示各维度对比

#### 11. 引用块（quote-block）
- 左侧 3px accent 色竖线
- 引用文字 + 来源标注

#### 12. 高亮卡片（highlight-card）
- 白底卡片，带图标标题 + 正文
- 用于强调关键设计、注意事项等

#### 13. 步骤列表（step-list / step-item）
- 编号圆形标记 + 标题 + 描述文字
- 用于列举要点

#### 14. 时间线（timeline）
- 垂直时间线，左侧圆点 + 日期 + 内容
- 用于展示事件发展脉络

#### 15. 结论块（conclusion-block）
- 居中大号图标 + 标题 + 正文
- 用于章节或全文的总结性陈述

#### 16. 附录卡片（appendix-card）
- 紫色主题（#7c4dff），区别于正文章节
- 包含：来源标签、文章标题（可点击跳转原文）、元信息、核心思想高亮框、简要讲解要点列表
- 每个要点用编号圆形标记

#### 17. 图解容器（diagram-container）⭐ 新增
- SVG 技术图解的外部容器
- 白底圆角卡片，带边框和阴影
- `overflow-x: auto` 支持移动端横向滚动
- 底部配图注说明文字 `diagram-caption`

```css
.diagram-container {
  background: var(--bg-card);
  border-radius: var(--radius-lg);
  padding: 24px;
  margin: 24px 0;
  border: 1px solid var(--border-light);
  box-shadow: var(--shadow-sm);
  overflow-x: auto;
}
.diagram-caption {
  text-align: center;
  font-size: 13px;
  color: var(--text-tertiary);
  margin-top: 12px;
  font-style: italic;
}
```

#### 18. 深度解析卡片（deep-dive-card）⭐ 新增
- 用于对视频中关键概念进行超出视频内容的深度解释
- 左侧 4px accent 色边线，渐变背景
- 标题带 🔍 图标前缀
- 可嵌套 `matrix-display` 组件展示数值计算

```css
.deep-dive-card {
  background: linear-gradient(135deg, #fff8f0, #fffcf8);
  border: 1px solid var(--border);
  border-left: 4px solid var(--accent);
  border-radius: 0 var(--radius-lg) var(--radius-lg) 0;
  padding: 24px 28px;
  margin: 24px 0;
  box-shadow: var(--shadow-md);
}
.deep-dive-card .deep-dive-label {
  font-size: 13px;
  font-weight: 700;
  color: var(--accent);
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 12px;
  display: flex;
  align-items: center;
  gap: 6px;
}
.deep-dive-card .deep-dive-label::before {
  content: '🔍';
}
.deep-dive-card p {
  font-size: 14.5px;
  color: var(--text-secondary);
  line-height: 1.85;
  margin-bottom: 10px;
}
.deep-dive-card p:last-child { margin-bottom: 0; }
```

#### 19. 矩阵显示（matrix-display）⭐ 新增
- 用于展示数值计算的中间矩阵
- 等宽字体，支持横向滚动
- 配合 `deep-dive-card` 使用

```css
.matrix-display {
  background: var(--bg-card);
  border: 1px solid var(--border-light);
  border-radius: var(--radius);
  padding: 16px 20px;
  margin: 12px 0;
  font-family: 'Courier New', monospace;
  font-size: 13px;
  line-height: 1.8;
  overflow-x: auto;
  white-space: pre;
}
```

### 侧边栏导航

#### 结构

```html
<nav class="sidebar" id="sidebar">
  <div class="sidebar-header">
    <div class="sidebar-title">目录导航</div>
    <div class="sidebar-close" onclick="toggleSidebar()">✕</div>
  </div>
  <div class="sidebar-body">
    <!-- 每个章节一个 group -->
    <div class="sidebar-chapter-group" data-chapter="ch1">
      <div class="sidebar-chapter-link" onclick="toggleSub(this)">
        <span class="sidebar-chapter-num">01</span>章节标题
      </div>
      <ul class="sidebar-sub-list">
        <li><a class="sidebar-sub-link" onclick="navTo(this, 'ch1')">子标题1</a></li>
        <li><a class="sidebar-sub-link" onclick="navTo(this, 'ch1')">子标题2</a></li>
      </ul>
    </div>
  </div>
  <div class="sidebar-progress">
    <div class="sidebar-progress-label">
      <span>阅读进度</span>
      <span id="sidebarProgressPct">0%</span>
    </div>
    <div class="sidebar-progress-bar">
      <div class="sidebar-progress-fill" id="sidebarProgressFill"></div>
    </div>
  </div>
</nav>
```

#### 行为

- **切换按钮**：固定在左侧中间，点击展开/收起侧边栏
- **展开/收起**：点击章节标题展开该章子标题列表，同时收起其他章节
- **子标题跳转**：点击子标题平滑滚动到对应章节区域
- **滚动追踪**：滚动时自动高亮当前所在章节，展开对应子标题列表
- **阅读进度**：底部显示百分比进度条
- **遮罩层**：侧边栏展开时显示半透明遮罩，点击遮罩关闭侧边栏
- **键盘快捷键**：Escape 关闭侧边栏
- **移动端适配**：侧边栏宽度 260px，覆盖式弹出

### JavaScript 功能

#### 1. 阅读进度条
```javascript
function updateProgress() {
  const scrollTop = document.documentElement.scrollTop || document.body.scrollTop;
  const scrollHeight = document.documentElement.scrollHeight - document.documentElement.clientHeight;
  const progress = (scrollTop / scrollHeight) * 100;
  progressBar.style.width = progress + '%';
  sidebarProgressFill.style.width = progress + '%';
  sidebarProgressPct.textContent = Math.round(progress) + '%';
}
window.addEventListener('scroll', updateProgress);
```

#### 2. 章节入场动画
```javascript
const chapters = document.querySelectorAll('.chapter');
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      entry.target.classList.add('visible');
    }
  });
}, { threshold: 0.08 });
chapters.forEach(ch => observer.observe(ch));
```

#### 3. 滚动追踪当前章节
```javascript
const chapterIds = ['ch1', 'ch2', 'ch3', 'ch4', 'ch5', 'ch6', 'appendix'];
function updateActiveChapter() {
  const scrollTop = window.scrollY + 120;
  let currentChapter = chapterIds[0];
  chapterIds.forEach(id => {
    const el = document.getElementById(id);
    if (el && el.offsetTop <= scrollTop) currentChapter = id;
  });
  // 更新侧边栏高亮...
}
window.addEventListener('scroll', updateActiveChapter);
```

#### 4. TOC 链接导航
```javascript
// 确保 scroll-margin-top 防止锚点偏移
// 所有章节 section 设置 scroll-margin-top: 20px
document.querySelectorAll('.toc-link, .sidebar-sub-link').forEach(link => {
  link.addEventListener('click', (e) => {
    const href = link.getAttribute('href') || link.dataset.href;
    if (href && href.startsWith('#')) {
      e.preventDefault();
      const target = document.querySelector(href);
      if (target) target.scrollIntoView({ behavior: 'smooth', block: 'start' });
    }
  });
});
```

### 章节入场动画 CSS

```css
.chapter {
  opacity: 0;
  transform: translateY(20px);
  transition: opacity 0.6s ease, transform 0.6s ease;
}
.chapter.visible {
  opacity: 1;
  transform: translateY(0);
}
```

### 移动端适配

```css
@media (max-width: 768px) {
  .cover-title { font-size: 26px; }
  .page-wrapper { padding: 0 16px; }
  .compare-grid { grid-template-columns: 1fr; }
  .feature-track { flex-wrap: wrap; }
}
```

---

## 质量检查清单

生成 HTML 后，逐项检查：

### 内容质量
- [ ] 所有专有名词拼写正确（尤其注意 Whisper 转录的谐音错误）
- [ ] 视频中每个专业术语都有基础知识解释（deep-dive-card）
- [ ] 深度解析卡片覆盖了定义、直觉、原理、对比、代价五个层次
- [ ] 对比卡片的数据准确
- [ ] 附录文章链接可点击跳转到原文

### 技术图解
- [ ] 所有关键概念配有 SVG 技术图解
- [ ] SVG 使用 viewBox 属性，支持响应式缩放
- [ ] SVG 配色使用设计系统颜色
- [ ] 图解标签使用中文
- [ ] 不同模块用颜色区分语义

### 交互功能
- [ ] 侧边栏章节编号和子标题与正文内容一致
- [ ] 所有章节 id 与侧边栏 data-chapter 对应
- [ ] 章节间的 divider-accent 分隔线完整
- [ ] 流程图箭头方向和连接关系正确
- [ ] 移动端布局正常（无水平溢出）
- [ ] 滚动进度条和章节追踪功能正常
- [ ] 侧边栏展开/收起/跳转功能正常
- [ ] scroll-margin-top 防止锚点偏移

### 文件规范
- [ ] HTML 文件是完整可独立运行的单一文件（无外部依赖）
- [ ] 文件大小不超过 200KB
- [ ] SVG 内联嵌入，不引用外部图片文件

---

## 文件管理规范

- **转录临时文件**（audio.m4a、transcript.txt、transcribe.py）→ 保存到临时工作目录
- **最终 HTML 文件** → 保存到用户工作空间目录
- HTML 文件命名格式：`{主题关键词}-summary.html`

---

## 典型触发场景

- 用户说"总结这个视频"并提供了视频链接
- 用户说"把这个视频做成笔记网页"
- 用户说"生成一个视频总结的HTML阅读页面"
- 用户提供了视频链接并要求"微信读书风格"的输出
