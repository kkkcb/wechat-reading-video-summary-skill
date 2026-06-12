# 📖 微信读书风格视频总结 HTML 生成器

将视频（B站、YouTube、微信视频号等）文案转录、结构化总结，生成精美的**微信读书风格 HTML 阅读页面**。

## 特性

- 🎨 **微信读书风格**：温暖米色主题、Noto Serif/Sans 字体、卡片式布局
- 📊 **SVG 技术图解**：内联嵌入，支持响应式缩放，中文标签
- 🔍 **深度解析卡片**：专业术语 5 层深度解释（定义→直觉→原理→对比→代价）
- 📱 **移动端适配**：响应式布局，侧边栏导航
- 📜 **阅读进度条**：顶部进度条 + 侧边栏进度同步
- 🧩 **18 个组件**：封面、目录、章节、对比表、流程图、时间线、附录等
- ⚡ **单文件输出**：HTML 完整自包含，无外部依赖（除 Google Fonts）

## 四阶段工作流

```
视频转录 → 内容总结与基础知识补充 → 技术图解与深度解析 → HTML 生成
```

1. **阶段一：视频转录** — yt-dlp 下载音频 + faster-whisper 转文字（支持字幕直接下载）
2. **阶段二：内容总结** — 结构化拆解为 5-8 个章节，补充基础知识（专业术语、公式推导、概念对比）
3. **阶段三：图解设计** — 规划 SVG 技术图解和 deep-dive-card 深度解析卡片
4. **阶段四：HTML 生成** — 微信读书风格完整页面，18 个组件按需使用

## 核心原则

> **视频总结不仅是"记录视频说了什么"，更是"让读者真正理解这些内容"。**

每个专业术语都附带：
- **定义层**：一句话解释
- **直觉层**：日常类比帮助理解
- **原理层**：数学公式或计算步骤
- **对比层**：与相关概念的区别
- **代价层**：设计的 trade-off

## 设计系统

| 变量 | 值 | 说明 |
|------|------|------|
| `--bg-primary` | `#f6f1eb` | 温暖米色底色 |
| `--bg-card` | `#fffcf8` | 偏白暖色卡片 |
| `--accent` | `#c0793c` | 琥珀橙强调色 |
| `--text-primary` | `#2c2418` | 深棕主文字 |
| `--text-secondary` | `#6b5e4f` | 中棕次文字 |

## 示例文件（examples/）

| 文件 | 视频来源 | 说明 |
|------|----------|------|
| `01-harness-engineering-summary.html` | Harness Engineering 概念视频 | AI 工程化流程详解 |
| `02-claude-code-self-verification-summary.html` | X/Twitter 视频 | Claude Code 自我验证能力 |
| `03-transformer-paper-reading-summary.html` | B站 李沐论文精读 | Transformer 架构论文详解 |
| `04-astigmatism-summary.html` | 微信视频号 PROEM视力训练 | 散光科普与健康建议 |
| `05-anthropic-claude-mike-krieger-summary.html` | YouTube Every 播客 | Anthropic CPO 深度对话 |

直接在浏览器中打开任意 HTML 文件即可查看效果。

## 使用方式

将 `SKILL.md` 放入项目的 `.trae/skills/wechat-reading-video-summary/` 目录，然后在 SOLO 中提供视频链接即可触发。

支持的视频来源：
- **Bilibili**（bilibili.com）
- **YouTube**（youtube.com）
- **微信视频号**（weixin.qq.com/sph/）

## 文件结构

```
.
├── SKILL.md              # Skill 完整定义（CSS、JS、组件规范、工作流）
├── README.md             # 本文件
└── examples/             # 生成的示例 HTML 文件
    ├── 01-harness-engineering-summary.html
    ├── 02-claude-code-self-verification-summary.html
    ├── 03-transformer-paper-reading-summary.html
    ├── 04-astigmatism-summary.html
    └── 05-anthropic-claude-mike-krieger-summary.html
```

## License

MIT
