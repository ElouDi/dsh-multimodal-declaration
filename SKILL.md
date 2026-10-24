---
name: dsh-multimodal-declaration
description: "在 DSH 模型列表配置中为实际支持图像输入的模型添加多模态能力声明（修复 does not declare image input）"
---

# DSH 多模态能力声明（Multimodal Capability Declaration）

给 DSH 模型列表上实际支持图像输入的模型，在配置文件中添加多模态（image input）能力声明，使 `read_image` 等图像工具可以正常调用。

## 背景机制（为什么声明会缺失）

DSH 核心在处理图像前有一道硬门禁（`assertImageCapableRoute`）：调用 `llm.resolveModelInfo(provider, model)` 取得该路由的 `inputModalities`，若不包含 `"image"` 则直接抛出：

```
model "<model-id>" does not declare image input
```

关键点：**这个声明与模型是否真的具备视觉能力无关，只取决于配置**。而且不同 provider 路由的声明来源不同，改错地方不会生效：

| 路由类型 | 声明来源 | 生效写法 |
| --- | --- | --- |
| pi-ai 型 provider（如 `providers.<name>.models` 直接在 patch 文件中定义） | 模型条目的 `input:` 字段 | `input: [text, image]` |
| 有自带目录的连接插件（如 workbuddy 类插件） | 插件保存的 `imageModelIds` 白名单（位于 `regions.<region>.imageModelIds`） | 把模型 id 加入 `imageModelIds` 列表 |

对插件型路由注意三点：

1. 插件运行时逻辑通常是 `multimodal === true ? ["text","image"] : ["text"]`，而 `multimodal` **只**由保存的 `imageModelIds` 决定。
2. 上游平台目录的 `supportsImages: true` 只是平台声明，**不等于**模型能看图；它甚至只是"否决项"（为 false 才拦截），永远不会主动把模型判定为可看图。
3. `imageToolEnabled` 是另一个独立开关（让模型主动调用看图/画图工具），与模态声明无关，不要混淆。

## 工作流程

### 第 1 步：定位配置文件

DSH 的 patch 配置（`cordis.patch.yml` / `cordis.patch.yaml`）通常位于：

- `DSH_HOME/profiles/<profile>/cordis.patch.yml`（profile 目录也可以在 `node_modules` 下找到已安装插件的实现，如 `dsh-connect-<name>/lib/index.js`）
- 如果不确定 DSH_HOME，查看环境变量 `DSH_*` 或运行时上下文中的提示

先通读配置，弄清当前激活的模型路由（provider + model id）属于哪一种类型。

### 第 2 步：确认门禁判定链（可选但推荐）

在已安装插件的 `lib/` 目录中 grep 以下关键词，确认该 provider 的模态判定来源：

- `inputModalities` / `assertImageCapableRoute` / `does not declare image input`
- `imageModelIds` / `multimodal`
- `input:`（pi-ai 型）

这一步可以避免"改了 pi-ai 的 `input` 字段但激活路由其实是插件白名单"这类无效修改。

### 第 3 步：收集证据，判定哪些模型真的支持图像

**无据不猜** —— 只有拿到证据才写声明。证据来源按可靠性排序：

1. **插件内置的厂商核验表**：很多连接插件带一张按精确 model id 匹配的原生模态表（常名为 `NATIVE_MODALITY_BY_MODEL_ID` 之类），值通常是 `multimodal` / `text` / `unknown`。这是最可靠的本地证据。注意按**精确 id** 匹配，族关系不能传递（`glm-5.3` 可看图不代表 `glm-5.3-flash` 也可以；`hy3` 不代表 `hy3-x`）。
2. **上游/厂商对模型的中文或英文描述**：如"原生多模态"、"多模态模型"、"vision"等字样。
3. **第三方聚合目录**：如 OpenRouter 的 `https://openrouter.ai/api/v1/models`（字段 `architecture.input_modalities`），可用同源厂商模型交叉验证。注意有些模型名字像多模态实际是纯文本，必须逐个确认而不是猜。
4. **用户确认**：都没有证据时问用户，或留空不写。

### 第 4 步：编辑配置

pi-ai 型 provider —— 给模型条目加 `input` 字段：

```yaml
providers:
  some-provider:
    models:
      - id: some-vision-model
        name: Some Vision Model
        contextWindow: 1000000
        maxTokens: 131072
        input:
          - text
          - image
```

插件型 provider —— 把模型 id 加入对应区域的 `imageModelIds`：

```yaml
        regions:
          <region>:
            imageModelIds:
              - some-vision-model
              - another-vision-model
            imageToolEnabled: false   # 独立开关，保持原值，不要擅自改
```

### 第 5 步：验证

1. **YAML 语法**：用任意 YAML 解析器（如 `require("yaml")` / `js-yaml` / python 的 `yaml.safe_load`）解析整个文件，确认无语法错误、顶层条目结构完好。
2. **内容核对**：统计带 `input: [...image]` 的条目数与不带的是否符合预期；`imageModelIds` 列出的每个 id 确实存在于插件目录中。
3. **运行态生效**：patch 文件的 `regions` 等属于 volatile 字段，DSH 的 HMR 监听文件改动后会就地重建目录，**无需重启**。可通过插件的 web 路由（如 `POST /plugins/<plugin>/__save` 回读运行态，或 `models/refresh`）确认新列表已被采纳；也可以直接对某个模型调用探测路由验证可用性。
4. **端到端验证**：重新调用 `read_image` 读一张真实图片，确认不再报 `does not declare image input`。

## 原则

- **无据不猜**：没有证据支持模型可看图时，不要添加声明。宁缺勿滥。
- **精确 id 匹配**：不通过模型族/命名相似度传递结论。
- **只改声明，不动开关**：`imageToolEnabled` 等独立功能开关保持原值。
- **改错位置等于没改**：先确认激活路由类型再动手。
- **验证闭环**：改完必须实际调用一次图像读取确认生效。

## 触发场景

- 用户报告 `does not declare image input` 或图像读取失败但模型实际支持视觉。
- 用户要求检查模型列表、为支持图像的模型补齐能力声明。
- 新装/新配置 DSH 后想启用多模态工作流。
