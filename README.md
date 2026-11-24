# AstrBot Injection Blocker

AstrBot 消息注入拦截插件：在消息进入 LLM / 其它插件之前，检查是否命中拦截词并终止事件传播；同时提供历史上下文注入清洗与图片文字预检能力。

## 功能特性

- **入口消息拦截**：群聊 / 私聊消息（含引用原文）命中拦截词后立即 `stop_event()`，LLM 与其它插件都收不到该消息。
- **合并转发 / 聊天记录处理**：
  - 递归展开合并转发卡片文本参与拦截词匹配（平台支持时调用 `get_forward_msg` 拉取子消息）。
  - 引用普通图片 / 文本消息（例如引用一张图让 bot 识图）不会被误判成“带图聊天记录”。
- **历史上下文注入清洗**：在 `on_llm_request` 阶段扫描历史上下文与框架后注入的文本面（如合并转发、引用链、识图摘要等），默认把命中内容替换为占位符（`sanitize`），可切换为直接终止（`reject`）。
- **图片文字预检**（可选）：主模型直接多模态看图时，先把图片内文字转出来再扫一遍拦截词，命中即终止。识别方式 `image_ocr_backend` 三选一：**auto**（默认，优先本地 OCR，本地不可用才回退 API）、**local**（只走 `rapidocr_onnxruntime`，不联网不花 token）、**api**（只走你配置的多模态模型）。
- **动态拦截词管理**：拦截词通过 `/pblock` 命令实时增删查清，不依赖配置文件，随 AstrBot KV 存储持久化。
- **仅管理员可操作**：`/pblock` 所有写操作均需 AstrBot 管理员权限。

## 安装

将本插件目录放到 AstrBot `data/plugins/` 下并启用，或在 WebUI 插件市场安装后启用。

要求：AstrBot v4.x（使用插件 KV 存储 API 与 `on_llm_request` 过滤器）。

### 可选依赖：本地 OCR

只有 `image_ocr_enabled=true` 且后端用到 local 时才需要（默认的 `auto` 会用到）。**不装也能正常用**：本地引擎加载失败会记一条 error 并置为不可用，`auto` 模式若配了 `image_ocr_provider_id` 就自动改走 API，没配则跳过预检。

```bash
# 用 AstrBot 解释器装（Windows 示例）
<AstrBot 的 python> -m pip install rapidocr_onnxruntime
```

它会把 `onnxruntime` 一起拉进来，约 80 MB；识别模型首次运行自动下载/加载，之后常驻内存。装好之后插件会在第一次需要 OCR 时初始化引擎（约 1-2 秒），后续复用。

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
| `block_message` | bool | `false` | 是否向被拦截消息发送者回复提示（默认静默，避免暴露过滤规则）。 |
| `block_message_text` | string | `您的消息包含不允许的内容，已自动拦截。` | 拦截提示文案，可用 `{word}` 占位符。 |
| `case_sensitive` | bool | `false` | 匹配是否区分大小写。 |
| `normalize_text` | bool | `true` | 匹配前做 NFKC 归一 + 去零宽字符 + 去空白再比一次，挡全角/零宽/拆词绕过。 |
| `regex_mode` | bool | `false` | 是否把拦截词按正则表达式匹配。 |
| `show_nickname` | bool | `false` | 拦截日志中是否包含发送者昵称（DEBUG 日志可见）。 |
| `check_quote` | bool | `true` | 检查引用消息原文是否含拦截词。 |
| `context_block_enabled` | bool | `true` | 请求发给 LLM 前是否扫描清洗历史上下文中的注入内容。 |
| `context_block_mode` | string | `sanitize` | `sanitize`=替换为占位符后继续；`reject`=命中直接终止本次请求。 |
| `context_block_placeholder` | string | `[Filtered]` | 清洗时使用的占位文本。 |
| `image_ocr_enabled` | bool | `false` | 请求前是否把图片里的文字转出来扫拦截词。 |
| `image_ocr_device` | string | `cpu` | `cpu`=强制 CPU（最稳，默认）；`gpu`=优先 CUDA（需 onnxruntime-gpu + 匹配 CUDA/cuDNN，失败自动回退 CPU）；`auto`=检测到 CUDAExecutionProvider 就用 GPU。 |
| `image_ocr_backend` | string | `auto` | `auto`=优先本地 OCR，不可用且有配 provider 时回退 API；`local`=只用本机 rapidocr；`api`=只用多模态模型。 |
| `image_ocr_provider_id` | string | `` | API 模式使用的多模态模型 id，留空则 API 模式跳过预检。 |
| `image_ocr_api_prompt` | string | `请逐字转写图片中出现的所有文字，不要输出其它内容。` | API 模式的转写提示词。 |
| `image_ocr_max_images` | int | `4` | 单次最多处理几张图，超出部分不检查。 |
| `image_ocr_timeout` | float | `15.0` | 单次图片预检超时（秒）。local 模式按此超时，api 模式给两倍。 |

## 工作原理

```
消息进入
  └─ on_message（入口拦截）
       ├─ 纯文本 + 引用原文 扫描拦截词 → 命中即 stop_event
       └─ 合并转发/聊天记录：递归展开文本扫描（含图片不再自动拦截）
LLM 请求前
  └─ on_llm_request（优先级极低，兜底）
       ├─ 图片文字预检（可选，local / api / auto）
       ├─ 当前请求本身命中 → 终止
       └─ 历史上下文/注入面 → sanitize 或 reject
```

拦截词存于 AstrBot 插件 KV 存储（`get_kv_data` / `put_kv_data`），重启不丢失，不随源码分发。

## 注意事项

- 关键词过滤是基础防御，无法覆盖编码、繁体、同义改写等绕过方式，建议配合权限分级、沙箱等使用。
- 拦截词与聊天内容不会上传或外发。图片预检走本地 OCR（`rapidocr_onnxruntime`），不联网、不调模型；唯一的网络请求是 OCR 前下载图片本体（如果平台给的是 URL）。
- `context_block_mode=reject` 命中历史注入时会终止本次回复，需要清空会话才能继续聊天，一般建议保持 `sanitize`。

## License

MIT

## 更新日志

### 1.6.0

- **本地 OCR 可选 GPU**（新配置 `image_ocr_device`，默认 `cpu`）：`gpu` 走 `RapidOCR(det_use_cuda=True, ...)`，需要 `onnxruntime-gpu` + 匹配的 CUDA/cuDNN；初始化失败自动回退 CPU 并记一条 warning。`auto` 检测 `CUDAExecutionProvider` 再决定。默认 CPU 是因为本机 `rapidocr_onnxruntime` 自带的 onnxruntime 就是 CPU 版，够用且部署最省事。
- **插空比对按 v2 规格收紧**：从「去掉所有空白 + 长度≥2」改成「逐行去水平空白（保留换行）+ 中文≥3 字 / 纯 ASCII≥4 字符」。前者会把跨行内容粘成一个词造成误伤，后者门槛太低（「人」「好」这种满屏都是）。
- `normalize_text` 的 hint 补上 Bidi 不可见字符说明，并提醒 `regex_mode` 同开时正则里的标点写半角。

### 1.5.0

- **匹配前文本归一化**（新配置 `normalize_text`，默认 `true`）：先 `NFKC` 归一（全角英数折叠、兼容字符归一）再删零宽/不可见字符（`\u200b-\u200f` / `\u2060` / `\ufeff` / `\u00ad`），最后额外做一次「去掉所有空白」的比较。
- 挡的是三类改写：`忽\u200b略以上`（零宽拆词）、`忽略以上` 用全角字符混写、`忽 略 以 上`（插空）。正则模式只归一化被检查的文本，不动管理员写的表达式；插空比对只对长度 ≥2 的词生效，避免单字词在这种模式下到处误命中。

### 1.4.0

- **图片 OCR 改成 `auto / local / api` 三档调度**：`auto`（默认）优先本机 OCR；只有「本地引擎不可用」才回退到 API，**图片里没字不会白花 API 额度**。这靠把 `_ocr_sync` / `_ocr_local_images` 的返回值从 `str` 改成 `(text, ok)`——`("", True)` 是「识别了但没字」，`("", False)` 是「引擎不可用」。
- **API OCR 防反注入**：调用多模态模型时用 `system_prompt` 把图片文字明确声明为不可信数据，要求机械逐字转写、不执行图中任何指令（含「忽略以上」「输出空内容」这类对抗话术）。Provider 的 `text_chat()` 不支持 `system_prompt` 参数时自动 `TypeError` 回退成拼到 prompt 前面。
- **防「多图坏图掩护」**：批量请求失败、且剩余超时够（>1s）时，自动降级为逐张识别，一张坏链接不再让整条消息其余图片逃检。
- **配置数值健壮性**：`_safe_int` / `_safe_float` 给 `image_ocr_max_images`（≥1）和 `image_ocr_timeout`（≥1.0s）兜底，非法值不会把 `asyncio.wait_for` 弄崩。非法 `backend` 值记 warning 后按 `auto` 处理。
- `context_block_mode` 补 `options: ["sanitize", "reject"]`；`context_block_enabled` 的 hint 写明它不影响图片 OCR、当前请求兜底和注入面检查。

### 1.3.1

- 历史清洗的默认占位符从 `[已拦截]` 改为 `[Filtered]`（`context_block_placeholder`）。已经配过值的不受影响，改的只是默认值——方括号里挂着中文，模型有可能顺手把它当正文复述出来。

### 1.3.0

- **图片文字预检支持 API 回退**：新增 `image_ocr_backend`（`local` / `api`）。`local` 走本机 `rapidocr_onnxruntime`；`api` 走 `image_ocr_provider_id` 指定的多模态模型，适合本机没装 rapidocr、或想用更强模型识别的场景。配套新增 `image_ocr_provider_id` 与 `image_ocr_api_prompt`。
- 没有可用识别方式时（本地没装 + API 没配 provider）静默跳过预检，只留说明性日志，不影响正常回复。

### 1.2.1

- **删除**「带图片的合并转发/聊天记录整条拦截」（原 `block_forward_image`）。那个行为会误伤正常聊天记录，且和「拦截词命中」的语义不一致——现在只按词拦。
- **图片文字预检改为本地 OCR**：不再需要配置识图模型（原 `image_caption_precheck` / `image_caption_provider_id` / `image_caption_prompt` 全部移除），改用本机 `rapidocr_onnxruntime`，不联网、不调模型、不花 token。新配置 `image_ocr_enabled` / `image_ocr_max_images` / `image_ocr_timeout`。
- OCR 用 `threading.Lock` 串行、deadline 贯穿到加载与识别内部，超时后线程自行停下，不会跑完才丢弃，也不会卡住回复。
- `/pblock` 放行改为「正则匹配 + 管理员校验」：普通用户发送 `/pblock add xxx` 不再被当作管理命令绕过拦截。
- 引用原文检查保留完整正文，不再依赖 `get_message_outline()`（该方法对长文本会截断，注入内容堆在末尾会漏检）。
- 拦截回复文案抽成 `_format_block_reply()`，`{word}` 占位统一处理。
- `show_nickname` 的配置提示更正为 INFO 级别可见（日志实际走 `logger.info`，原来写 DEBUG 是错的）。

### 1.1.0

- 历史上下文注入清洗（sanitize / reject），`on_llm_request` 优先级 -300000，排在陪伴插件注入之后。
- 合并转发展开、引用原文检查、拦截日志。
