# DSH Multimodal Declaration（DSH 多模态能力声明）

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Platform: DSH](https://img.shields.io/badge/platform-DeepSeek%20Harness-blue)](https://github.com/deepseek-ai)
[![Type: Agent Skill](https://img.shields.io/badge/type-Agent%20Skill-green)](./SKILL.md)

一个用于 **DeepSeek Harness（DSH）** 的 Agent Skill：在 DSH 模型列表的配置文件中，为实际支持图像输入的模型添加多模态（image input）能力声明，修复 `read_image` 等图像工具报错

```
model "<model-id>" does not declare image input
```

的问题。

---

## 为什么需要这个 Skill

当你把 **OpenRouter、Vercel（AI Gateway）、Workbuddy** 等 API 反代/接入到 DSH 上使用时，会遇到一个常见困扰：很多模型（如各种 VL / Vision / Omni 系列模型）本身具备多模态视觉能力，但在 DSH 中调用 `read_image` 读取图片时却被拦截报错。

**这不是模型没有能力，而是 DSH 配置里缺少能力声明。** DSH 核心在处理图像前有一道硬门禁（`assertImageCapableRoute`），它会检查当前模型路由声明的 `inputModalities` 是否包含 `"image"`，不包含就直接拒绝。而不同接入方式下，这个声明的存放位置完全不同——改错地方等于没改。本 Skill 就是把"定位 → 确认判定链 → 收集证据 → 修改声明 → 验证闭环"这条完整流程固化下来，让 Agent 可以可靠、可复现地完成修复。

## 适用范围

本 Skill 适用于以下场景：

- 将 **OpenRouter** 的模型通过 pi-ai 型 provider 配置接入 DSH（`providers.<name>.models` 的 `input:` 字段声明）
- 将 **Vercel AI Gateway** 等 OpenAI 兼容 API 反代接入 DSH
- 通过 **Workbuddy 类连接插件**（`dsh-connect-*`）接入的模型（`regions.<region>.imageModelIds` 白名单声明）
- 用户报告 `does not declare image input` 或图像读取失败、但所用模型实际支持视觉输入
- 新装/新配置 DSH 后想启用多模态工作流，需要批量为支持图像的模型补齐能力声明

**不适用于**：模型本身确实不支持图像输入的情况——本 Skill 遵循"无据不猜"原则，不会为纯文本模型伪造声明。

## 适用条件

| 条件 | 要求 |
| --- | --- |
| DSH 版本 | **DeepSeek Harness ≥ 2026.10**（需具备：`assertImageCapableRoute` 图像门禁、`cordis.patch.yml` patch 配置、`regions` volatile 字段 HMR 热更新、`llm.resolveModelInfo()` 模态解析链；更早版本未经验证，`cordis.patch.*` 之前的旧配置体系不适用） |
| 操作系统 | 跨平台（Windows / macOS / Linux），Skill 内不含平台特定命令 |
| 权限 | 需要能读写 DSH profile 目录（通常为 workspace-write 级别） |
| 验证依赖 | 任一 YAML 解析器（`yaml` / `js-yaml` / `pyyaml`），可选 `curl` 或 `web_fetch` 用于查询 OpenRouter 公开目录 |

## 安装

### 方法一：git clone（推荐）

```bash
# Linux / macOS
git clone https://github.com/ElouDi/dsh-multimodal-declaration.git \
  "$DSH_HOME/skills/dsh-multimodal-declaration"

# Windows (PowerShell)
git clone https://github.com/ElouDi/dsh-multimodal-declaration.git `
  "$env:DSH_HOME\skills\dsh-multimodal-declaration"
```

> 如果 `$DSH_HOME` 未设置，默认位置通常是 `~/.dsh`（即 Windows 下的 `%USERPROFILE%\.dsh`）。安装完成后目录结构应为：
>
> ```
> ~/.dsh/skills/dsh-multimodal-declaration/
> ├── SKILL.md      # skill 主体（必须）
> └── README.md     # 本文件
> ```

### 方法二：手动复制

从 GitHub 下载仓库 zip 或复制文件，将 `SKILL.md` 放到：

```
DSH_HOME/skills/dsh-multimodal-declaration/SKILL.md
```

### 安装后验证

重启 DSH（或开启新会话）后，询问 Agent "你有哪些可用的 skill"，确认列表中出现 `dsh-multimodal-declaration` 即安装成功。

## 如何让 DSH 使用本 Skill

安装后 Agent 在遇到相关问题时会**自动路由**到此 Skill；也可以显式触发：

**自动触发（推荐）** —— 直接描述问题或任务：

```text
帮检查目前 DSH 的模型列表，把实际具有 image 能力的模型都在配置文件中添加相关声明。
```

```text
read_image 报错 does not declare image input，帮我修复。
```

```text
我把 OpenRouter 反代到了 DSH 上，某个支持看图的模型读不了图片，用 skill 处理一下。
```

**显式调用** —— 在对话中点名 skill：

```text
用 dsh-multimodal-declaration 这个 skill，检查并补齐所有多模态模型的能力声明。
```

只要对话内容涉及"图像读取失败""模型能力声明""`does not declare image input`""多模态配置"等关键词，Agent 就会加载并按本 Skill 的五步流程执行。

## 使用方法

安装后无需手动操作。典型触发方式：

- 直接对 Agent 说："帮检查目前 DSH 的模型列表，把实际具有 image 能力的模型都在配置文件中添加相关声明。"
- Agent 报出 `does not declare image input` 错误时自动应用。

Skill 内部会按五步流程执行：

1. **定位配置** — 找到 `cordis.patch.yml`，判断激活路由类型
2. **确认判定链** — grep 插件源码确认该 provider 的模态声明来源
3. **收集证据** — 四级证据链（插件内置核验表 > 上游模型描述 > 第三方目录交叉验证 > 用户确认），坚持"无据不猜"与"精确 id 匹配"
4. **编辑配置** — 按路由类型写入 `input: [text, image]` 或 `imageModelIds` 白名单
5. **验证闭环** — YAML 语法校验 → 内容核对 → 运行态 HMR 回读 → 实际调用 `read_image` 端到端确认

## 核心知识（本 Skill 的技术依据）

- DSH 的图像门禁只看**配置声明**，与模型真实能力无关
- pi-ai 型 provider 看模型条目的 `input:` 字段；连接插件型看 `imageModelIds` 白名单，且运行时 `multimodal` 标志**只**由该白名单决定
- 上游平台的 `supportsImages: true` 只是展示性声明（甚至只是否决项），永远不会主动让模型获得图像能力
- `imageToolEnabled` 是独立的画图工具开关，与模态声明无关，本 Skill 不会擅自修改
- `regions` 等 volatile 字段的改动经 DSH 的 HMR 就地生效，**无需重启**

## 隐私说明

本 Skill 不含任何特定机器的路径、用户名、API Key 或模型配置。所有示例使用占位符（`some-vision-model`、`<profile>`、`<region>` 等），可安全地在任何机器上使用与分享。

## 制作说明

- **Harness**：DeepSeek Harness（DSH）
- **制作模型**：GLM-5.3-Flash（由智谱 AI 提供），运行于 DeepSeek Harness Agent 运行时
- **来源**：从一个真实故障修复 session 的完整流程中提炼固化而成
- **Skill 结构**：遵循 DSH 原生 Agent Skill 规范（`SKILL.md` + YAML front matter），未使用任何外部依赖

## License

本项目以 [MIT License](./LICENSE) 发布。

## 免责声明

本 Skill 按现状提供（AS IS），不附带任何明示或默示的担保。使用前请自行备份 DSH 配置文件；对 DSH 配置的修改可能导致模型行为变化，请仔细阅读本 Skill 的"核心知识"一节后再执行。
