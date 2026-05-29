# 草稿管理 (Draft Management)

在蚁小二生态中，存在两种“草稿”能力，需按目标选择：
- 保存到**蚁小二草稿箱**（`action: save-draft`）
- 保存到**目标平台草稿箱**（`action: publish` + `pubType: 0`）

## 触发场景 (Trigger)
- **意图辨析**：
  - **蚁小二草稿**：当用户希望先录入内容，待以后手动检查或由他人审核，不希望立即启动任何平台推送流程时。
  - **平台草稿**：当内容已比较完善，但由于平台规则（如视频号必须在手机端二次确认）或用户希望在平台后台进行最后的 SEO、话题优化时。
- **典型提示词**：
  - **蚁小二草稿**：“帮我把这个视频存为蚁小二的草稿”、“暂时不发布，先存草稿”、“存为 YXE 草稿”。
  - **平台草稿**：“把内容推送到抖音的草稿箱里”、“存为小红书草稿”、“推送到平台草稿箱”。
- **强制约束**：
  - 若用户说“存草稿”但未说明位置，Agent **必须询问用户** 明确意图（是存为“蚁小二草稿”还是“平台草稿”），严禁自行猜测或默认选择。
  - **蚁小二草稿**不消耗发布配额。
  - **平台草稿**消耗一次云发布/本机发布配额。

## 执行逻辑 (Logic Flow)
1. **意图深度研判**：根据提示词关键字选取模式。
2. **蚁小二草稿模式 (`action: save-draft`)**：
   - 构造 `action: "save-draft"`。
   - 不进入 `publish` 路径，不使用 `isDraft: true` 作为替代方案。
   - `contentPublishForm.pubType` 可设为 1 (发布) 或 0 (草稿)，后端仅做云端存储。
3. **平台草稿模式 (`action: publish` + `pubType: 0`)**：
   - 根级别注入 `action: "publish"`。
   - `accountForms` 内每个账号的 `contentPublishForm.pubType` 必须设为 `0`。
4. **指令执行**：调用 `node scripts/api.ts --payload='{...}'`。
5. **状态反馈**：告知用户草稿保存位置及对应的 `taskSetId`。

## 快速区分 (Draft Types)

| 类型 | 蚁小二草稿 (YiXiaoEr Draft) | 平台草稿 (Platform Draft) |
| :--- | :--- | :--- |
| **定义** | 仅保存在蚁小二系统数据库中 | 发送到目标平台（如抖音、B站）的草稿箱 |
| **是否触发任务** | 否 (仅存储) | 是 (执行推送流程) |
| **调用 Action** | `save-draft` | `publish` |
| **核心参数** | `action: "save-draft"` | `contentPublishForm.pubType: 0` |
| **主要用途** | 跨端同步编辑、团队预审 | 将内容预推送到平台后台，方便手动二次微调 |

---

## 存为蚁小二草稿 (`action: save-draft`)

当需要将任务暂存到蚁小二系统的草稿列表，而不启动发布流程时使用。

禁止使用 `action: "publish"` + `isDraft: true` 代替本模式；`isDraft: true` 只能视为旧字段或标记位，不能作为阻止平台推送的安全边界。

### 参数列表 (Key Parameters)
| 字段名 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| `action` | `string` | **是** | 固定值：`save-draft` |
| `publishType` | `string` | **是** | `video`、`article` 或 `imageText` |
| `platforms` | `string[]` | **是** | 草稿关联的平台。后端仍要求有平台信息 |
| `publishArgs.accountForms` | `Array` | **是** | 草稿关联账号表单。接口通常仍要求账号；若不能省略账号，必须先通过 `accounts` 查询并填入 `platformAccountId` |

### 图文草稿强制 DTO 规则

保存图文到蚁小二草稿箱时，`publishType` 必须使用 `imageText`，图片字段不能只放在平台透传层。

必须同时补齐以下字段：

```text
coverKey
publishArgs.content
publishArgs.accountForms[].images
publishArgs.accountForms[].coverKey
publishArgs.accountForms[].cover
publishArgs.accountForms[].contentPublishForm
```

字段放置规则：

- `publishArgs.content`：图文正文的通用内容，纯文本优先；用于蚁小二列表、通用图文 DTO 和非平台透传场景。
- `contentPublishForm.title`：平台侧标题；如果目标平台有标题长度限制，必须按平台限制预校验。
- `contentPublishForm.description`：平台侧正文 / 描述；可按平台文档使用 HTML，例如 `<p>`、`<topic>`。
- `publishArgs.content` 和 `contentPublishForm.description` 应表达同一份正文，不要一个是旧稿、一个是新稿。
- 如果只填写 `contentPublishForm.description`，草稿列表或通用图文层可能缺少正文摘要；如果只填写 `publishArgs.content`，部分平台侧预览可能缺少描述。
- `accountForms[].images`：图文图片主数组，必须放在账号表单外层。
- `accountForms[].cover`：主封面对象，通常使用第一张图。
- `accountForms[].coverKey`：必须与 `cover.key` 一致。
- 根级 `coverKey`：通常也使用第一张图的 `key`。
- `contentPublishForm`：平台透传层，可放 `formType`、`title`、`description`、`pubType` 以及平台特有字段。
- 可以按平台文档在 `contentPublishForm.images` 中冗余放一份图片数组，但不能只放这一处。

标题和正文校验规则：

- 标题必须来自用户确认的发布预览或交付包，不得临时改写核心含义。
- 标题超过平台限制时，必须先给出缩短版本并让用户确认。
- 抖音图文标题不得大于 20 个字符。
- 正文必须来自用户确认的内容或交付包；为适配平台可以做格式转换，但不得新增未确认事实、医疗 / 法律 / 金融结论或风险承诺。
- `publishArgs.content` 建议使用纯文本，保留换行和话题。
- `contentPublishForm.description` 可使用平台支持的 HTML 包装同一份正文。
- 话题可以放在正文末尾；如平台要求结构化话题，再按平台文档补充。

常见错误：

```json
{
  "publishArgs": {
    "accountForms": [
      {
        "contentPublishForm": {
          "images": [
            { "key": "img_key_1", "width": 1080, "height": 1440, "size": 200000 }
          ]
        }
      }
    ]
  }
}
```

这个结构可能导致蚁小二草稿列表里看不到图片，因为图文主 DTO 没有收到 `accountForms[].images`、`cover` 和 `coverKey`。

标题 / 正文也有类似问题：

```json
{
  "publishArgs": {
    "accountForms": [
      {
        "contentPublishForm": {
          "title": "平台标题",
          "description": "<p>平台正文</p>"
        }
      }
    ]
  }
}
```

这个结构可能导致通用图文草稿缺少正文内容，因为 `publishArgs.content` 没有填写。正确做法是：`publishArgs.content` 放纯文本正文，`contentPublishForm.description` 放平台格式化正文，二者内容保持一致。

正确结构：

```json
{
  "action": "save-draft",
  "publishType": "imageText",
  "platforms": ["抖音"],
  "coverKey": "img_key_1",
  "desc": "图文草稿说明",
  "publishArgs": {
    "content": "图文正文。#话题",
    "accountForms": [
      {
        "platformAccountId": "DOUYIN_ACCOUNT_ID",
        "images": [
          { "key": "img_key_1", "width": 1080, "height": 1440, "size": 200000, "format": "png" },
          { "key": "img_key_2", "width": 1080, "height": 1440, "size": 200000, "format": "png" }
        ],
        "coverKey": "img_key_1",
        "cover": { "key": "img_key_1", "width": 1080, "height": 1440, "size": 200000, "format": "png" },
        "contentPublishForm": {
          "formType": "task",
          "pubType": 1,
          "title": "图文标题",
          "description": "<p>图文正文。#话题</p>",
          "images": [
            { "key": "img_key_1", "width": 1080, "height": 1440, "size": 200000, "format": "png" },
            { "key": "img_key_2", "width": 1080, "height": 1440, "size": 200000, "format": "png" }
          ]
        }
      }
    ]
  }
}
```

注意：

- 这是蚁小二草稿，`contentPublishForm.pubType` 可以用 `1`；它不会触发平台发布。
- 保存蚁小二草稿时，不要为了“草稿”把 `action` 改成 `publish`。
- 若用户要求“抖音草稿箱”，才是平台草稿，应走 `action: "publish"` + `pubType: 0`。
- 抖音图文标题不得大于 20 个字符；超过时必须先缩短并经用户确认。
- 所有图片必须先上传得到 `key`，不能直接填本地路径或外部 URL。

### 调用指令 (Command)

```bash
node scripts/api.ts --payload='{
  "action": "save-draft",
  "publishType": "video",
  "platforms": ["抖音", "视频号"],
  "desc": "这是一个蚁小二草稿",
  "publishArgs": {
    "accountForms": [
      {
        "platformAccountId": "67fb2f1735eeb3cf31db3d65",
        "video": { "key": "v-xxxxxx" },
        "coverKey": "c-xxxxxx",
        "contentPublishForm": {
          "pubType": 1,
          "title": "这是一个蚁小二草稿"
        }
      }
    ]
  }
}'
```

---

## 存为平台草稿 (`pubType: 0`)

当需要启动发布流程，但最终结果是在第三方平台后台看到草稿时使用。

### 调用指令 (Command)

```bash
node scripts/api.ts --payload='{
  "action": "publish",
  "publishType": "video",
  "platforms": ["抖音"],
  "publishArgs": {
    "accountForms": [
      {
        "platformAccountId": "acc_vid_003",
        "video": { "key": "v_key" },
        "coverKey": "c_key",
        "contentPublishForm": {
          "pubType": 0,
          "title": "存入抖音草稿箱的内容"
        }
      }
    ]
  }
}'
```
