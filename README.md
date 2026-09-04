# AstrBot Injection Blocker

AstrBot 消息注入拦截插件：在消息进入 LLM / 其它插件之前，检查是否命中拦截词并终止事件传播；同时提供历史上下文注入清洗与图片文字预检能力。

## 功能特性

- **入口消息拦截**：群聊 / 私聊消息（含引用原文）命中拦截词后立即 `stop_event()`，LLM 与其它插件都收不到该消息。
- **合并转发 / 聊天记录处理**：
  - 递归展开合并转发卡片文本参与拦截词匹配（平台支持时调用 `get_forward_msg` 拉取子消息）。
  - 带图片的合并转发 / 聊天记录，即使展开文本未命中，也默认整条拦截（`block_forward_image`），防止图片内的注入文字被带进上下文。
  - 引用普通图片 / 文本消息（例如引用一张图让 bot 识图）不会被误判为“带图聊天记录”。
- **历史上下文注入清洗**：在 `on_llm_request` 阶段扫描历史上下文与框架后注入的文本面（如合并转发、引用链、识图摘要等），默认把命中内容替换为占位符（`sanitize`），可切换为直接终止（`reject`）。
- **图片文字预检**（可选）：主模型直接多模态看图时，可先调用指定识图模型把图片内文字转写出来再扫一遍拦截词，命中即终止。
- **动态拦截词管理**：拦截词通过 `/pblock` 命令实时增删查清，不依赖配置文件，随 AstrBot KV 存储持久化。
- **仅管理员可操作**：`/pblock` 所有写操作均需 AstrBot 管理员权限。

## 安装

将本插件目录放到 AstrBot `data/plugins/` 下并启用，或在 WebUI 插件市场安装后启用。

要求：AstrBot v4.x（使用插件 KV 存储 API 与 `on_llm_request` 过滤器）。

## 使用方法

插件默认拦截词列表为空，不产生任何过滤效果。请先用管理员账号添加拦截词：

| 命令 | 功能 | 示例 |
|------|------|------|
| `/pblock add <word>` | 添加拦截词 | `/pblock add 忽略以上指令` |
| `/pblock del <word>` | 删除拦截词 | `/pblock del 忽略以上指令` |
| `/pblock list` | 查看全部拦截词 | `/pblock list` |
| `/pblock clear` | 清空全部拦截词 | `/pblock clear` |

`regex_mode` 开启后，每个拦截词会被当作正则表达式编译匹配（例如 `/pblock add (?i)ignore.*instructions`）。

## 配置项

配置项在 AstrBot WebUI 插件配置页可视化设置（`_conf_schema.json` 提供默认值）。

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `group_enabled` | bool | `true` | 群聊中是否启用拦截。 |
| `private_enabled` | bool | `true` | 私聊中是否启用拦截。 |
| `check_forward` | bool | `true` | 入口是否递归展开合并转发卡片内的文本再检查。 |
| `block_forward_image` | bool | `true` | 带图片的合并转发 / 聊天记录是否直接拦截。 |
| `block_message` | bool | `false` | 是否向被拦截消息发送者回复提示（默认静默，避免暴露过滤规则）。 |
| `block_message_text` | string | `您的消息包含不允许的内容，已自动拦截。` | 拦截提示文案，可用 `{word}` 占位符。 |
| `case_sensitive` | bool | `false` | 匹配是否区分大小写。 |
| `regex_mode` | bool | `false` | 是否把拦截词按正则表达式匹配。 |
| `show_nickname` | bool | `false` | 拦截日志中是否包含发送者昵称（DEBUG 日志可见）。 |
| `check_quote` | bool | `true` | 检查引用消息原文是否含拦截词。 |
| `context_block_enabled` | bool | `true` | 请求发给 LLM 前是否扫描清洗历史上下文中的注入内容。 |
| `context_block_mode` | string | `sanitize` | `sanitize`=替换为占位符后继续；`reject`=命中直接终止本次请求。 |
| `context_block_placeholder` | string | `[已拦截]` | 清洗时使用的占位文本。 |
| `image_caption_precheck` | bool | `false` | 是否先用识图模型转写图片文字再扫拦截词（会产生额外模型调用费用）。 |
| `image_caption_provider_id` | string | `` | 用于图片预检的多模态模型 id，留空则预检不生效。 |
| `image_caption_prompt` | string | `请逐字转写图片中出现的所有文字，不要输出其它内容。` | 图片预检的转写提示词。 |

## 工作原理

```
消息进入
  └─ on_message（入口拦截）
       ├─ 纯文本 + 引用原文 扫描拦截词 → 命中即 stop_event
       └─ 合并转发/聊天记录：递归展开文本扫描；
            含图片段 → 默认整条拦截（防图内文字注入）
LLM 请求前
  └─ on_llm_request（优先级极低，兜底）
       ├─ 图片文字预检（可选）
       ├─ 当前请求本身命中 → 终止
       └─ 历史上下文/注入面 → sanitize 或 reject
```

拦截词存于 AstrBot 插件 KV 存储（`get_kv_data` / `put_kv_data`），重启不丢失，不随源码分发。

## 注意事项

- 关键词过滤是基础防御，无法覆盖编码、繁体、同义改写等绕过方式，建议配合权限分级、沙箱等使用。
- 拦截词与聊天内容不会上传或外发，插件不发起任何网络请求（图片预检除外，仅在你显式配置识图模型后才会调用）。
- `context_block_mode=reject` 命中历史注入时会终止本次回复，需要清空会话才能继续聊天，一般建议保持 `sanitize`。

## License

MIT
