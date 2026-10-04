---
name: ecom-image-workflow
title: 跨境电商商品图工作流（线路一 + 线路二）
description: 处理跨境电商商品图片和竞品参考图片：先识别图片来源与商品类目，再按路线执行。自有商品图走“视觉识别 → 事实与推断分离 → 类目化 Prompt → 详情图生成 → 质检报告”；竞品参考图走“竞品识别 → 可借鉴卖点提炼 → 差异化 Prompt → 商品图生成 → 对比报告”。当用户提供商品图或竞品图，并提到生成详情图、重制主图、识别图片生成提示词、分析竞品、提炼卖点、跑线路一或跑线路二时使用。
version: 2.0.0
agent_created: true
trigger_when:
  - 用户提供自有商品图并要求“生成详情图”“重制商品图”“跑线路一”或“按工作流处理这张图”
  - 用户提供竞品参考图并要求“分析竞品”“提炼卖点”“生成差异化商品图”或“跑线路二”
  - 用户提到“电商图片工作流”“识别图片生成提示词”，但未明确路线时先执行统一路由判断
---

# 跨境电商商品图工作流

## 适用场景

用户提供自有商品实拍图或竞品参考图，希望走“识别 → 关键信息 → Prompt → 生图 → 质检报告”流水线，产出可用于 listing 的图片。统一入口必须先判断 `sourceType`，再选择路线；不要把两条路线的结论混在一起。

## 执行步骤

### 0. 统一路由与前置确认

- 若用户没给图片路径，先索要：要求直接拖图进对话，或给出本机绝对路径。
- ImageGen 生图约消耗 5–10 积分/张。用户首次使用时提醒一句即可，不必每次确认。
- 根据用户描述和图片内容设置 `sourceType`：`own_product`（自有商品图）、`competitor_reference`（竞品参考图）或 `mixed`（两者都有）。
- 用户未说明来源时询问一句；若图片明显包含多个品牌或竞品陈列，默认 `competitor_reference`，并标注这是推断。
- `own_product` 执行线路一；`competitor_reference` 执行线路二；`mixed` 先分别识别，再分别执行两条路线。

### 1. 视觉识别（Read 图片）

用 Read 工具直接读取图片（多模态视觉能力），输出结构化识别结果，字段固定为：

```text
sourceType / productType / category / subcategory / visibleFacts / inferences / uncertainties
visualElements / colorScheme / material / style / composition / background / lighting / resolution
```

事实与推断必须分开：

- `visibleFacts` 只记录图片中直接可见或用户明确提供的内容，例如颜色、结构、材质纹理、文字、Logo、配件和拍摄条件。
- `inferences` 记录模型根据视觉线索做出的判断，例如目标市场、价格带、消费人群、使用场景和潜在卖点；每项必须带 `confidence`（0–1）和 `evidence`。
- 无法确认的内容放入 `uncertainties`，使用 `unknown`；禁止把推断写成事实。看不清的 Logo、文字、成分和认证不得猜测。

推荐结构：

```json
{
  "sourceType": "own_product",
  "category": {"value": "apparel", "confidence": 0.96},
  "visibleFacts": [{"field": "color", "value": "black", "evidence": "主面料呈黑色"}],
  "inferences": [{"field": "targetMarket", "value": "US casualwear", "confidence": 0.62, "evidence": "版型与场景线索"}],
  "uncertainties": ["Logo text is not legible"]
}
```

### 2. 类目识别与策略选择

先将 `category` 归一化，再选择 Prompt 策略；无法可靠归类时使用 `generic`，不要套用服装模板。

| 类目 | 类目策略重点 |
|---|---|
| `apparel` / `footwear` / `bags` | 模特或平铺、版型、面料、缝线、穿着状态、细节特写 |
| `beauty` / `personal_care` | 包装、膏体或液体质地、使用动作、洁净背景、成分视觉化（仅使用已知信息） |
| `electronics` | 产品结构、接口、屏幕状态、功能场景、尺寸比例、材质反光控制 |
| `home` / `furniture` / `kitchen` | 空间关系、使用场景、尺寸感、材质、收纳或功能演示 |
| `food` / `beverage` | 包装、食物状态、份量、食用场景、食品安全和文字保真 |
| `generic` | 主体、结构、材质、颜色、比例、使用场景和平台构图 |

输出 `categoryStrategy`，至少包含 `category`、`visualFocus`、`sceneType`、`compositionRule` 和 `avoid`。

### 3. 提取关键信息

从识别结果提炼：

- `visualElements`：必须保留进 Prompt 的可见视觉元素（面料、结构、Logo、车线、接口等）。
- `colorScheme`：主色调、搭配色和需要避免的颜色漂移。
- `composition`：原图构图和可执行的构图建议。
- `mood`：氛围关键词，只使用与图片或用户要求一致的词。
- `keySellingPoints`：核心卖点（2–4 条）；推断出来的卖点必须标注置信度，不能伪装成已证实功能。

### 4. 生成英文 Prompt

根据 `sourceType` 和 `categoryStrategy` 拼装英文电商摄影 Prompt。先写通用保真约束，再写类目专属内容；不要强行填入不适用的模特、面料或搭配单品。

```text
Professional e-commerce [category] photography, faithful representation of the supplied product,
[主体与使用状态], [必须保留的可见视觉元素], [类目策略中的场景与构图],
[背景与光线], [平台规格], clean commercial look, high detail,
do not alter product color, structure, proportions, components, or visible branding,
do not invent text, logo, certification, material, feature, or accessory.
```

要点：

- 自有商品图：保留用户确认的品牌名、联名、Logo 和其它视觉元素；图片中看不清的文字不得补写。
- 竞品参考图：只提炼可借鉴的构图、色彩、场景和卖点，不复制竞品 Logo、品牌名、独特包装或受保护设计；生成结果必须改为用户自己的商品信息。
- 保留原图配色与产品结构，除非用户明确要求改变；把用户要求的改变单独写入 `requestedChanges`。
- Prompt 默认可直接执行；若会改变商品身份、品牌元素或平台合规状态，先向用户确认。

### 5. 按路线执行

**线路一：自有商品图 → 详情图**

执行“事实识别 → 类目策略 → listing 图片 Prompt → ImageGen”。输出商品识别、保真约束、生成图和报告。

**线路二：竞品参考图 → 差异化商品图**

执行“竞品事实识别 → 可借鉴元素 → 优势/劣势推断 → 用户商品差异化卖点 Prompt → ImageGen”。竞品分析只能用于抽象构图、色彩、场景和卖点方向，不得复制品牌识别元素或独特受保护设计。

### 6. AI 生图（ImageGen）

用 DeferExecuteTool 调用 `ImageGen`（若 schema 未加载，先 ToolSearch 加载）：

```text
params: {
  prompt: <上一步的英文 Prompt>,
  size: "1024x1536",        // 竖版详情图；主图可用 1024x1024
  quality: "high",
  style: "photographic",
  background: "opaque",
  output_dir: "<工作区>/outputs/<run-id>"
}
```

工具参数以当前 ImageGen schema 为准；schema 未加载时先 ToolSearch，不要假定所有参数都受支持。

### 7. 保存结果与报告

在 `<工作区>/outputs/<run-id>/` 下产出：

1. `workflow_result.json`：`sourceType`、`recognition`、`categoryStrategy`、`keyInfo`、`prompt`、`generatedImages`、`uncertainties` 和错误状态。
2. `workflow_report.html`：原图/参考图与生成图对比、事实/推断分栏、类目策略、Prompt 摘要和路线说明。

最后用 `present_files` 依次展示报告 HTML、生成图 PNG、结果 JSON、源图或参考图。

## 输出话术模板

最终回复需包含：路线判断、商品或竞品识别结论（一两句）、识别事实与推断摘要、类目策略、Prompt 和生图结果摘要、产出文件列表、下一步建议。不要把线路一和线路二写成同一组结论。

## 边界与注意

- 图片读取失败（路径不存在、格式不支持、图片过小或主体不可见）时直接告知用户，不要用文字模拟代替。
- ImageGen 失败时报告错误并保留前置识别 JSON，告知用户可稍后从生图步骤重试。
- 线路一覆盖自有商品详情图；线路二覆盖竞品参考分析与差异化商品图，二者由 `sourceType` 明确分流。
- 不得猜测看不清的品牌、Logo、文字、成分、认证、功能或价格。
- 配套的 Web 应用（index.html + server.js）与本 Skill 独立；Skill 不依赖该应用运行。
