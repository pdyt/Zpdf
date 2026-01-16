# PDF解析API接口介绍

## 基础信息

### Base URL

所有接口请求均使用以下基地址：

`https://www.zhiyipdf.com`

### General Notes

1. **网络连接：** 请直连访问 API 接口。中国内地以外地区可能会遭遇网络波动，导致文件上传中断。
2. **数据保留：** 通过状态接口获取结果后，请尽快手动下载或通过下载接口获取文件到本地。服务器上仅临时保留7天的结果数据。
   - **图片 URL 过期：** 当使用 `images_as_url=true` 输出图片 URL 时，图片缓存文件会在服务器上保留 **30 天**，到期会被清理，请在有效期内自行保存。
3. **文件限制：** 
   *   单个文件大小 **<= 300MB**
   *   单个文件页数 **<= 800 页**

---

## 鉴权

### 获取 API Key

您首先需要获取 API Key。请联系平台管理员获取您的 API Key。

### 请求头格式

在 HTTP 请求头中加入 `Authorization` 字段，采用 Bearer Token 格式，或使用 `X-API-Key` 头：

| 名称            | 示例值          | 说明                                 |
| :-------------- | :-------------- | :----------------------------------- |
| `Authorization` | `Bearer sk-xxx` | 请替换 `sk-xxx` 为您的真实 API Key。 |
| `X-API-Key` | `sk-xxx` | 请替换 `sk-xxx` 为您的真实 API Key。 |

**或者** 通过 URL 参数传递：

| 参数名            | 示例值          | 说明                                 |
| :-------------- | :-------------- | :----------------------------------- |
| `api_key` | `sk-xxx` | 请替换 `sk-xxx` 为您的真实 API Key。 |

---

## 异步解析流程

ZhiyiPDF API 采用三步异步处理流程：**文件上传解析** → **轮询状态** → **下载结果**。

### 1. 文件上传解析

**POST /api/pdf-to-markdown-proxy/parse**

使用此接口上传 PDF 文件并开始解析任务。

#### 请求参数

| 名称                | 位置      | 类型     | 必选 | 说明                                                         |
| :------------------ | :-------- | :------- | :--- | :----------------------------------------------------------- |
| `file`              | FormData  | `file`   | 是   | PDF 文件（二进制流）。                                        |
| `table_mode`        | FormData  | `string` | 否   | 表格输出格式。可选值：`markdown`（转换为 Markdown 表格）、`image`（保留为图片）。默认值：`markdown`。 |
| `enable_translation` | FormData  | `string` | 否   | 是否启用翻译。可选值：`true`、`false`。默认值：`false`。     |
| `images_as_url`     | FormData  | `string` | 否   | 是否将图片以公网 URL 形式输出。可选值：`true`、`false`。默认值：`false`。当设为 `true` 时：接口最终仅返回 **Markdown 文件**（不再打包 ZIP），Markdown 内图片引用会被替换为 URL；图片可通过 `https://www.zhiyipdf.com/images/{api_key}/{task_id}/{image_name}` 直接访问。**图片缓存保留 30 天，过期会清理。** |
| `target_language`   | FormData  | `string` | 否   | 目标语言代码。仅在 `enable_translation=true` 时有效。可选值：`zh`（中文）、`en`（英文）、`ja`（日文）、`ko`（韩文）、`fr`（法文）、`de`（德文）、`es`（西班牙文）、`ru`（俄文）。默认值：`zh`。 |
| `output_options`    | FormData  | `string` | 否   | 翻译输出选项，仅在 `enable_translation=true` 时有效。多个选项用逗号分隔。可选值：`original`（原文）、`translated`（译文）、`bilingual`（双语）。默认值：`original`。可同时选择多个，例如：`original,translated,bilingual`。 |

#### 请求示例

**Windows (CMD/PowerShell):**
```cmd
curl -X POST "https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/parse?api_key=sk-xxx" ^
  -F "file=@document.pdf" ^
  -F "table_mode=markdown" ^
  -F "enable_translation=false"
```

**Windows (CMD/PowerShell) - 图片以URL输出（仅返回 Markdown）:**
```cmd
curl -X POST "https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/parse?api_key=sk-xxx" ^
  -F "file=@document.pdf" ^
  -F "table_mode=markdown" ^
  -F "enable_translation=false" ^
  -F "images_as_url=true"
```

**Linux/macOS:**
```bash
curl -X POST 'https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/parse?api_key=sk-xxx' \
  -F "file=@document.pdf" \
  -F "table_mode=markdown" \
  -F "enable_translation=false"
```

**Linux/macOS - 图片以URL输出（仅返回 Markdown）:**
```bash
curl -X POST 'https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/parse?api_key=sk-xxx' \
  -F "file=@document.pdf" \
  -F "table_mode=markdown" \
  -F "enable_translation=false" \
  -F "images_as_url=true"
```

#### 返回示例（成功 - 立即处理）

```json
{
  "success": true,
  "task_id": "12345",
  "status": "processing",
  "message": "PDF处理任务已启动",
  "points_deducted": 20,
  "remaining_points": 80
}
```

**说明：** 以上示例为 10 页 PDF 仅解析的情况，扣除积分为 `10 页 × 2 积分/页 = 20 积分`。如果启用翻译，则为 `10 页 × 3 积分/页 = 30 积分`。

#### 返回示例（成功 - 排队中）

```json
{
  "success": true,
  "task_id": "12345",
  "status": "waiting",
  "message": "任务已加入队列，等待处理",
  "points_deducted": 20,
  "remaining_points": 80,
  "queue_info": {
    "position": 1,
    "ahead_tasks": 3
  }
}
```

**说明：** 当您的 API Key 已有 3 个任务正在处理时，新提交的任务将进入等待队列。`queue_info` 字段显示：
- `position`: 当前任务在等待队列中的位置（从 1 开始）
- `ahead_tasks`: 前面有多少个任务（包括正在处理的任务和等待队列中排在前面的任务）

**并发限制：** 每个 API Key 最多同时处理 3 个任务。超过限制的任务将自动进入等待队列，当前面的任务完成时会自动开始处理。

#### 返回示例（积分不足）

```json
{
  "success": false,
  "message": "Insufficient points",
  "error_code": "insufficient_points",
  "points_required": 20,
  "current_points": 15
}
```

**说明：** 以上示例为 10 页 PDF 仅解析的情况，需要 `10 页 × 2 积分/页 = 20 积分`，但当前余额只有 15 积分。

#### 返回示例（文件格式错误）

```json
{
  "success": false,
  "message": "File is not a PDF file",
  "error_code": "parse_file_not_pdf"
}
```

#### 返回示例（文件过大）

```json
{
  "success": false,
  "message": "File size exceeds limit (300MB)",
  "error_code": "parse_file_too_large",
  "file_size": 314572800,
  "max_size": 314572800
}
```

#### 返回示例（页数超限）

```json
{
  "success": false,
  "message": "Page count exceeds limit (800 pages)",
  "error_code": "parse_page_limit_exceeded",
  "page_count": 1000,
  "max_pages": 800
}
```

**重要说明：** 当任务上传失败时（如文件格式错误、文件过大、页数超限等），**不会扣除积分**。积分仅在任务成功创建并准备处理时才会扣除。

#### 返回字段说明

| 字段              | 类型      | 说明                                       |
| :---------------- | :-------- | :----------------------------------------- |
| `success`          | `boolean` | 请求是否成功。                               |
| `task_id`          | `string`  | 任务 ID，用于后续查询状态和下载结果（仅在成功时返回）。         |
| `status`           | `string`  | 任务状态。可能值：`processing`（处理中）、`waiting`（排队中）。仅在成功时返回。   |
| `message`          | `string`  | 响应消息或错误信息。                                   |
| `error_code`       | `string`  | 错误代码（仅在失败时返回）。详见下方错误码说明。           |
| `points_deducted`  | `integer` | 本次扣除的积分数量（页数 × 单价，仅在成功时返回）。           |
| `remaining_points` | `integer` | 扣除后剩余的积分余额（仅在成功时返回）。                       |
| `points_required`  | `integer` | 所需积分数量（页数 × 单价，仅在积分不足时返回）。 |
| `current_points`   | `integer` | 当前积分余额（仅在积分不足时返回）。           |
| `queue_info`       | `object`  | 排队信息（仅在 `status=waiting` 时返回）。   |
| `queue_info.position` | `integer` | 当前任务在等待队列中的位置（从 1 开始）。   |
| `queue_info.ahead_tasks` | `integer` | 前面有多少个任务（包括正在处理的任务和等待队列中排在前面的任务）。   |

#### 积分消耗规则

| 服务类型       | 积分消耗（每页） |
| :------------- | :--------------- |
| 仅解析         | 2 积分           |
| 解析 + 翻译    | 3 积分           |

---

### 2. 查看异步状态

**GET /api/pdf-to-markdown-proxy/status/{task_id}**

使用此接口轮询文件解析任务的状态，建议轮询频率为 1~3 秒/次。

#### 请求参数

| 名称      | 位置  | 类型     | 必选 | 说明                                |
| :-------- | :---- | :------- | :--- | :---------------------------------- |
| `task_id` | Path  | `string` | 是   | 文件上传解析成功后返回的任务 ID。     |

#### 请求示例

**Windows (CMD/PowerShell):**
```cmd
curl "https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/status/01920000-0000-0000-0000-000000000000?api_key=sk-xxx"
```

**Linux/macOS:**
```bash
curl 'https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/status/01920000-0000-0000-0000-000000000000?api_key=sk-xxx'
```

#### 返回示例（成功）

```json
{
  "success": true,
  "status": "completed",
  "progress": 100,
  "message": "处理完成",
  "result": {
    "task_id": "01920000-0000-0000-0000-000000000000",
    "download_url": "/api/pdf-to-markdown-proxy/download/01920000-0000-0000-0000-000000000000"
  }
}
```

#### 返回示例（处理中）

```json
{
  "success": true,
  "status": "processing",
  "progress": 45,
  "message": "正在解析第 23 页..."
}
```

#### 返回示例（排队中）

```json
{
  "success": true,
  "status": "waiting",
  "progress": 0,
  "message": "等待中（前面3个任务）",
  "queue_info": {
    "position": 1,
    "ahead_tasks": 3
  }
}
```

**说明：** 当任务处于排队状态时，`status` 为 `waiting`，`queue_info` 字段显示排队信息。当前面的任务完成时，系统会自动开始处理等待中的任务。

#### 返回示例（失败）

```json
{
  "success": false,
  "status": "failed",
  "message": "解析错误：文件格式不正确",
  "error_code": "parse_error"
}
```

**说明：** 任务处理失败时，会返回 `error_code` 字段，用于标识具体的错误类型。如果任务在处理过程中失败，已扣除的积分会自动返还。

#### 返回字段说明

| 字段           | 类型      | 状态                              | 含义                                       |
| :------------- | :-------- | :-------------------------------- | :----------------------------------------- |
| `success`      | `boolean` | 始终返回                           | 请求是否成功。                               |
| `status`       | `string`  | `waiting`, `processing`, `failed`, `completed` | 任务状态。`waiting` 表示排队中。                                   |
| `progress`     | `integer` | 0~100                             | 任务进度百分比。                             |
| `message`      | `string`  | 始终返回                           | 状态消息或错误信息。                         |
| `queue_info`   | `object`  | 仅当 `status=waiting`             | 排队信息对象。                               |
| `queue_info.position` | `integer` | 仅当 `status=waiting` | 当前任务在等待队列中的位置（从 1 开始）。   |
| `queue_info.ahead_tasks` | `integer` | 仅当 `status=waiting` | 前面有多少个任务。   |
| `result`       | `object`  | 仅当 `status=completed`           | 解析结果对象。                               |
| `result.download_url` | `string` | 仅当 `status=completed` | 下载链接（相对路径）。                       |

---

### 3. 下载结果

**GET /api/pdf-to-markdown-proxy/download/{task_id}**

使用此接口下载解析完成的结果文件。

#### 请求参数

| 名称      | 位置  | 类型     | 必选 | 说明                                |
| :-------- | :---- | :------- | :--- | :---------------------------------- |
| `task_id` | Path  | `string` | 是   | 已完成解析任务的 ID。                |

#### 请求示例

**Windows (CMD/PowerShell):**
```cmd
curl -o downloaded_file.zip "https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/download/01920000-0000-0000-0000-000000000000?api_key=sk-xxx"
```

**Linux/macOS:**
```bash
curl -o downloaded_file.zip 'https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/download/01920000-0000-0000-0000-000000000000?api_key=sk-xxx'
```

**参数说明：**
- `-o downloaded_file.zip`: 指定输出文件路径和文件名。请将 `downloaded_file.zip` 替换为您希望保存的文件路径，例如：
  - Windows: `-o "C:\Users\YourName\Downloads\result.zip"`
  - Linux/macOS: `-o "/home/username/Downloads/result.zip"`
  

**注意：** 如果不使用 `-o` 参数指定输出路径，curl 会将文件内容输出到终端，可能导致终端显示乱码。建议始终使用 `-o` 参数指定输出文件路径。

或在浏览器中直接访问：

```
https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/download/01920000-0000-0000-0000-000000000000?api_key=sk-xxx
```

#### 返回说明

- 默认情况下成功时返回 ZIP 压缩文件，包含解析后的 Markdown 文件和相关资源（如图片）。
- 当上传解析时传入 `images_as_url=true`，下载接口将返回 **Markdown 文件（.md）**（不再返回 ZIP），图片会存放在公开路径，可通过：
  - `https://www.zhiyipdf.com/images/{api_key}/{task_id}/{image_name}` 访问。
- 文件通过 HTTP 流式下载；ZIP 场景 Content-Type 为 `application/zip`，Markdown 场景 Content-Type 为 `text/markdown`。

---

## 查询积分余额

**GET /api/pdf-to-markdown-proxy/balance**

使用此接口查询当前 API Key 的积分余额。

#### 请求参数

无

#### 请求示例

**Windows (CMD/PowerShell):**
```cmd
curl "https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/balance?api_key=sk-xxx"
```

**Linux/macOS:**
```bash
curl 'https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/balance?api_key=sk-xxx'
```

#### 返回示例（成功）

```json
{
  "success": true,
  "points": 98,
  "api_key": "sk-xxxx..."
}
```

**说明：** `api_key` 字段返回脱敏后的 API Key，仅显示前 8 个字符。

#### 返回示例（API Key 无效）

```json
{
  "success": false,
  "message": "Invalid API key"
}
```

#### 返回字段说明

| 字段      | 类型      | 说明                     |
| :-------- | :-------- | :----------------------- |
| `success` | `boolean` | 请求是否成功。             |
| `points`  | `integer` | 当前积分余额。             |
| `api_key` | `string`  | 查询的 API Key（脱敏显示）。 |
| `message` | `string`  | 错误消息（仅在失败时返回）。 |

---

## 错误码 (Error Codes)

### HTTP 状态码

| 状态码 | 含义              | 说明                                                  |
| :----- | :---------------- | :---------------------------------------------------- |
| `401`  | 未授权            | API Key 无效或未提供。                                 |
| `402`  | 积分不足          | 当前积分余额不足以完成本次操作。                        |
| `429`  | 超出 API 速率限制 | 需等待先前提交的任务完成。                              |

**注意：** 每个 API Key 最多同时处理 3 个任务。当您提交第 4 个任务时，该任务将进入等待队列（`status: "waiting"`），当前面的任务完成时会自动开始处理。您可以通过状态查询接口查看排队信息。
| `500`  | 服务器错误        | 服务器内部错误，具体的错误信息体现在 Response Body 中。 |

### 业务错误代码说明

| 错误代码                    | 原因                         | 解决方案                                           | 是否扣除积分 |
| :-------------------------- | :--------------------------- | :------------------------------------------------- | :---------- |
| `invalid_api_key`           | API Key 无效或不存在。        | 检查 API Key 是否正确，或联系管理员获取新的 API Key。 | ❌ 不扣除 |
| `insufficient_points`       | 可用的积分余额不足。          | 联系平台方获取更多积分。                             | ❌ 不扣除 |
| `no_file_found`             | 请求中未找到文件。            | 确保在 FormData 中包含 `file` 字段。                | ❌ 不扣除 |
| `parse_file_too_large`      | 单个文件大小超过限制。        | 当前允许单个文件大小 **<= 300MB**，请拆分 PDF。     | ❌ 不扣除 |
| `parse_page_limit_exceeded` | 单个文件页数超过限制。        | 当前允许单个文件页数 **<= 800 页**，请拆分 PDF。     | ❌ 不扣除 |
| `parse_file_not_pdf`        | 传入的文件不是 PDF 文件。     | 请确保解析后缀为 `.pdf` 的文件。                    | ❌ 不扣除 |
| `file_upload_failed`         | 文件上传到存储系统失败。       | 检查网络连接，稍后重试。                             | ❌ 不扣除 |
| `points_deduction_failed`   | 积分扣除操作失败。            | 联系技术支持。                                     | ❌ 不扣除 |
| `task_creation_failed`      | 任务创建失败。                | 联系技术支持。已扣除的积分会自动返还。                 | ✅ 已返还 |
| `parse_error`               | 解析错误。                   | 短暂等待后重试；若持续报错，请联系负责人。           | ✅ 已返还 |
| `parse_file_invalid`        | 解析文件错误或不合法。        | PDF 格式可能有问题或不规范。                         | ✅ 已返还 |
| `parse_timeout`             | 处理时间超过限制。            | 尝试切分 PDF 后再识别。                              | ✅ 已返还 |

**重要说明：**
- 所有上传阶段的错误（文件格式、大小、页数等验证失败）**不会扣除积分**。
- 如果任务创建成功但后续处理失败，已扣除的积分**会自动返还**。
- 只有在任务成功创建并准备处理时，积分才会被扣除。

---

## 参数说明

### 表格输出格式 (`table_mode`)

| 值        | 说明                                                         |
| :-------- | :----------------------------------------------------------- |
| `markdown` | 将表格转换为 Markdown 格式的表格。适合需要编辑和处理的场景。 |
| `image`   | 将表格保留为图片。适合表格结构复杂或需要保持原始样式的场景。 |

### 翻译参数

#### 目标语言 (`target_language`)

支持以下语言代码：

| 语言代码 | 语言名称   |
| :------- | :--------- |
| `zh`     | 中文       |
| `en`     | 英文       |
| `ja`     | 日文       |
| `ko`     | 韩文       |
| `fr`     | 法文       |
| `de`     | 德文       |
| `es`     | 西班牙文   |
| `ru`     | 俄文       |

#### 翻译输出选项 (`output_options`)

| 值           | 说明                                                         |
| :----------- | :----------------------------------------------------------- |
| `original`   | 仅输出原文（原始 Markdown 内容）。                           |
| `translated` | 仅输出译文（翻译后的 Markdown 内容）。                       |
| `bilingual`  | 输出双语版本（原文和译文对照）。                             |

**注意：** 可以同时选择多个选项，用逗号分隔。例如：`original,translated,bilingual` 将同时生成三种输出。

---

## 完整示例

### 示例 1：仅解析 PDF（表格转换为 Markdown）

**Windows (CMD/PowerShell):**
```cmd
REM 1. 上传文件并开始解析
curl -X POST "https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/parse?api_key=sk-xxx" ^
  -F "file=@document.pdf" ^
  -F "table_mode=markdown" ^
  -F "enable_translation=false"

REM 响应（假设PDF为10页）：
REM {
REM   "success": true,
REM   "task_id": "01920000-0000-0000-0000-000000000000",
REM   "status": "processing",
REM   "points_deducted": 20,
REM   "remaining_points": 80
REM }

REM 2. 轮询状态（每 2 秒查询一次）
curl "https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/status/01920000-0000-0000-0000-000000000000?api_key=sk-xxx"

REM 3. 下载结果
curl -o result.zip "https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/download/01920000-0000-0000-0000-000000000000?api_key=sk-xxx"
```

**Linux/macOS:**
```bash
# 1. 上传文件并开始解析
curl -X POST 'https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/parse?api_key=sk-xxx' \
  -F "file=@document.pdf" \
  -F "table_mode=markdown" \
  -F "enable_translation=false"

# 响应（假设PDF为10页）：
# {
#   "success": true,
#   "task_id": "01920000-0000-0000-0000-000000000000",
#   "status": "processing",
#   "points_deducted": 20,
#   "remaining_points": 80
# }

# 2. 轮询状态（每 2 秒查询一次）
curl 'https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/status/01920000-0000-0000-0000-000000000000?api_key=sk-xxx'

# 3. 下载结果
curl -o result.zip 'https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/download/01920000-0000-0000-0000-000000000000?api_key=sk-xxx'
```

### 示例 2：解析并翻译为英文（表格保留为图片）

**Windows (CMD/PowerShell):**
```cmd
REM 1. 上传文件并开始解析
curl -X POST "https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/parse?api_key=sk-xxx" ^
  -F "file=@document.pdf" ^
  -F "table_mode=image" ^
  -F "enable_translation=true" ^
  -F "target_language=en" ^
  -F "output_options=original,translated"

REM 响应（假设PDF为10页）：
REM {
REM   "success": true,
REM   "task_id": "01920000-0000-0000-0000-000000000000",
REM   "status": "processing",
REM   "points_deducted": 30,
REM   "remaining_points": 70
REM }

REM 2. 轮询状态（每 2 秒查询一次）
curl "https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/status/01920000-0000-0000-0000-000000000000?api_key=sk-xxx"

REM 3. 下载结果
curl -o result.zip "https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/download/01920000-0000-0000-0000-000000000000?api_key=sk-xxx"
```

**Linux/macOS:**
```bash
# 1. 上传文件并开始解析
curl -X POST 'https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/parse?api_key=sk-xxx' \
  -F "file=@document.pdf" \
  -F "table_mode=image" \
  -F "enable_translation=true" \
  -F "target_language=en" \
  -F "output_options=original,translated"

# 响应（假设PDF为10页）：
# {
#   "success": true,
#   "task_id": "01920000-0000-0000-0000-000000000000",
#   "status": "processing",
#   "points_deducted": 30,
#   "remaining_points": 70
# }

# 2. 轮询状态（每 2 秒查询一次）
curl 'https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/status/01920000-0000-0000-0000-000000000000?api_key=sk-xxx'

# 3. 下载结果
curl -o result.zip 'https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/download/01920000-0000-0000-0000-000000000000?api_key=sk-xxx'
```

### 示例 3：解析并翻译为日文（输出双语版本）

**Windows (CMD/PowerShell):**
```cmd
REM 1. 上传文件并开始解析
curl -X POST "https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/parse?api_key=sk-xxx" ^
  -F "file=@document.pdf" ^
  -F "table_mode=markdown" ^
  -F "enable_translation=true" ^
  -F "target_language=ja" ^
  -F "output_options=bilingual"

REM 响应（假设PDF为10页）：
REM {
REM   "success": true,
REM   "task_id": "01920000-0000-0000-0000-000000000000",
REM   "status": "processing",
REM   "points_deducted": 30,
REM   "remaining_points": 70
REM }

REM 2. 轮询状态
curl "https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/status/01920000-0000-0000-0000-000000000000?api_key=sk-xxx"

REM 3. 下载结果
curl -o result.zip "https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/download/01920000-0000-0000-0000-000000000000?api_key=sk-xxx"
```

**Linux/macOS:**
```bash
# 1. 上传文件并开始解析
curl -X POST 'https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/parse?api_key=sk-xxx' \
  -F "file=@document.pdf" \
  -F "table_mode=markdown" \
  -F "enable_translation=true" \
  -F "target_language=ja" \
  -F "output_options=bilingual"

# 响应（假设PDF为10页）：
# {
#   "success": true,
#   "task_id": "01920000-0000-0000-0000-000000000000",
#   "status": "processing",
#   "points_deducted": 30,
#   "remaining_points": 70
# }

# 2. 轮询状态
curl 'https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/status/01920000-0000-0000-0000-000000000000?api_key=sk-xxx'

# 3. 下载结果
curl -o result.zip 'https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/download/01920000-0000-0000-0000-000000000000?api_key=sk-xxx'
```

### 示例 4：查询积分余额

**Windows (CMD/PowerShell):**
```cmd
REM 查询当前 API Key 的积分余额
curl "https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/balance?api_key=sk-xxx"

REM 响应：
REM {
REM   "success": true,
REM   "points": 98,
REM   "api_key": "sk-xxxx..."
REM }
```

**Linux/macOS:**
```bash
# 查询当前 API Key 的积分余额
curl 'https://www.zhiyipdf.com/api/pdf-to-markdown-proxy/balance?api_key=sk-xxx'

# 响应：
# {
#   "success": true,
#   "points": 98,
#   "api_key": "sk-xxxx..."
# }
```

---

## 注意事项

1. **积分扣除时机：** 
   - 积分在任务成功创建并准备处理时扣除。
   - 如果任务上传失败（如文件格式错误、文件过大等），**不会扣除积分**。
   - 如果任务创建成功但后续处理失败，已扣除的积分**会自动返还**。
   - 如有异常，请联系我们。

2. **并发限制和排队：** 
   - 每个 API Key 最多同时处理 3 个任务
   - 超过限制的任务将自动进入等待队列
   - 当前面的任务完成时，等待中的任务会自动开始处理
   - 您可以通过状态查询接口查看排队信息和前面有多少个任务

3. **任务状态轮询：** 建议使用 1~3 秒的轮询间隔，避免过于频繁的请求导致服务器压力过大。

4. **文件下载：** 解析完成后，请尽快下载结果文件。服务器仅保留 7 天，过期后将无法下载。

5. **错误处理：** 当遇到错误时，请根据返回的错误码和消息进行相应处理。对于网络错误，建议实现重试机制。

6. **API Key 安全：** 请妥善保管您的 API Key，不要在公开代码或日志中暴露。建议使用环境变量或安全的配置管理方式。

---

## 技术支持

如有问题或需要帮助，请联系平台管理员或技术支持团队。