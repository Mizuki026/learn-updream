# Learn Updream：AI 视频生成网站 MVP

> 文档状态：第一版产品与技术范围
>
> 当前阶段：使用模拟模型打通完整链路，之后接入真实视频模型
>
> MVP 核心：参考图片 + 提示词生成约 5 秒的视频

## 1. 项目目标

从零实现一个可运行的 AI 视频生成网站，以此学习 AI 视频产品从前端交互、后端接口、异步任务、模型调用、文件存储到部署运维的完整流程。

第一版允许用户上传一张参考图片，并输入描述画面如何运动的提示词。系统创建异步生成任务，展示任务状态，最终提供视频预览、下载和历史记录。

本项目的“从零实现”是指从零搭建 AI 视频应用及其工程链路，不包含从零训练基础视频生成模型。第一阶段使用统一模型适配器调用模拟模型，之后再接入第三方模型 API 或本地开源模型。

## 2. 目标用户与运行方式

第一版面向项目开发者本人，用于学习和验证完整流程。

- 第一阶段在本地运行。
- 第一阶段不实现注册和登录。
- 系统使用固定的本地测试用户 `local_user`。
- 任务和历史记录保存在数据库中，刷新页面后仍然存在。
- 后续公开部署时，再增加注册、登录、用户额度、权限和支付功能。

## 3. MVP 范围

### 3.1 第一版包含

- 上传一张参考图片。
- 输入视频运动或镜头描述提示词。
- 选择视频比例：`16:9`、`9:16` 或 `1:1`。
- 生成固定约 5 秒的视频。
- 创建并持久化异步任务。
- 展示排队、生成中、成功和失败状态。
- 成功后在线播放和下载视频。
- 失败后显示可理解的错误信息，并允许重新生成。
- 在历史记录中查看以前的任务。
- 使用 `MockVideoProvider` 模拟完整生成流程。
- 通过统一适配器为后续真实模型接入预留接口。

### 3.2 第一版不包含

- 纯文本生成视频。
- 无提示词的纯图片生成视频。
- 首尾帧、多参考图和角色一致性。
- 多镜头故事、分镜和智能画布。
- 视频编辑、延长、补帧和超分辨率。
- 数字人、配音和口型同步。
- 用户注册、登录和第三方登录。
- 积分、订阅、支付和账单。
- 作品社区、点赞、评论和分享。
- 多模型自动比价、路由和负载均衡。
- 从零训练视频生成基础模型。

## 4. 用户操作路径

```text
进入首页
  ↓
点击“开始创作”
  ↓
进入创作页
  ↓
上传一张参考图片
  ↓
输入提示词
  ↓
设置生成参数
  ├─ 视频比例：16:9 / 9:16 / 1:1
  ├─ 时长：固定 5 秒
  └─ 模型：默认模型
  ↓
点击“生成视频”
  ↓
系统创建任务并进入结果页
  ↓
查看任务状态
  ├─ 正在准备
  ├─ 排队中
  ├─ 生成中
  ├─ 生成成功
  └─ 生成失败
  ↓
预览生成结果
  ↓
下载视频或重新生成
  ↓
在“历史记录”中再次查看
```

## 5. 页面清单

### 5.1 首页

- 简要介绍产品。
- 展示一个主要操作入口：“开始创作”。
- 第一版不要求复杂营销内容和作品社区。

### 5.2 创作页

- 上传、替换和删除参考图片。
- 输入提示词。
- 选择视频比例。
- 展示固定时长和当前模型。
- 提交生成任务。
- 展示输入校验错误和提交错误。

### 5.3 任务结果页

- 展示任务状态和必要的状态说明。
- 生成中时定期查询最新状态。
- 失败时展示用户可理解的错误和重新生成按钮。
- 成功后播放和下载视频。
- 页面刷新后继续展示同一任务，而不是重新创建任务。

### 5.4 历史记录页

- 按创建时间倒序展示任务。
- 展示参考图、提示词、状态和创建时间。
- 支持进入任务详情。
- 成功任务支持播放和下载。
- 失败任务支持重新生成。

## 6. 系统职责边界

### 6.1 本系统负责

- 接收并校验提示词、参考图片和生成参数。
- 将参考图片保存到本系统的文件存储。
- 创建并持久化本地任务。
- 通过统一模型适配器提交生成任务。
- 定时查询模型厂商的任务状态。
- 将不同厂商状态转换为系统统一状态。
- 下载并转存生成成功的视频。
- 向前端提供任务状态和结果地址。
- 保存和查询生成历史。
- 记录错误和必要的运行日志。

### 6.2 模型厂商负责

- 接收生成参数。
- 执行视频生成。
- 返回厂商任务 ID。
- 返回任务状态、错误信息和临时结果地址。

### 6.3 安全边界

- 前端只能调用本系统后端，不能直接调用模型厂商。
- 模型 API Key 只能保存在后端环境变量中。
- 前端不能获得模型 API Key 和厂商内部任务信息。

## 7. 核心数据流程

```text
用户提交图片和提示词
  ↓
后端校验并保存参考图片
  ↓
后端创建 pending 任务
  ↓
后台任务调用 VideoModelProvider
  ↓
获得 providerTaskId，状态变为 queued
  ↓
后台定期查询模型状态
  ↓
模型开始生成，状态变为 processing
  ↓
模型返回临时视频地址
  ↓
后端下载并转存视频
  ↓
转存成功后状态变为 succeeded
  ↓
前端播放本系统保存的视频
```

模型已经生成、但结果尚未成功转存时，任务不能进入 `succeeded`。

## 8. 参考图片与输入限制

第一版暂定以下限制，接入真实模型时可以根据厂商要求调整：

- 支持格式：JPEG、PNG、WebP。
- 最大文件体积：10 MB。
- 最小尺寸：256 × 256。
- 最大尺寸：4096 × 4096。
- 每次只能上传一张图片。
- 提示词为必填项，去除首尾空格后不能为空。
- 提示词长度暂定为 1～1000 个字符。
- 后端必须校验文件真实类型，不能只信任扩展名或前端校验。
- 上传失败时不能创建生成任务。

## 9. 任务状态

### 9.1 状态定义

| 状态值 | 中文名称 | 含义 | 用户界面 |
| --- | --- | --- | --- |
| `pending` | 等待提交 | 本地任务已创建，尚未提交给模型 | 正在准备 |
| `queued` | 排队中 | 已提交给模型，等待模型处理 | 排队中 |
| `processing` | 生成中 | 模型正在生成，或系统正在转存结果 | 生成中 |
| `succeeded` | 生成成功 | 视频已生成并成功保存到本系统 | 播放、下载 |
| `failed` | 生成失败 | 提交、生成、查询或文件转存发生不可恢复错误 | 显示错误、允许重试 |

第一版不提供主动取消任务，因此暂不启用 `cancelled` 状态。

### 9.2 状态转换

```text
pending
  ├─ queued
  └─ failed

queued
  ├─ processing
  └─ failed

processing
  ├─ succeeded
  └─ failed
```

正常路径：

```text
pending → queued → processing → succeeded
```

失败路径：

```text
pending → failed
queued → failed
processing → failed
```

### 9.3 状态处理规则

- 用户提交后，系统立即创建 `pending` 任务。
- 成功提交给模型并获得厂商任务 ID 后，状态变为 `queued`。
- 模型开始处理后，状态变为 `processing`。
- 视频生成并转存到本系统存储后，状态变为 `succeeded`。
- 任意步骤发生不可恢复错误时，状态变为 `failed`。
- `succeeded` 和 `failed` 是终态，正常情况下不能再次变化。
- 重新生成会创建新任务，不会修改原终态任务。
- 页面刷新不能改变任务状态。
- 后端负责查询模型状态，前端只展示本系统后端返回的状态。
- `progress` 允许为 `null`，因为部分模型不提供可信的进度百分比。

### 9.4 厂商状态映射示例

| 厂商状态示例 | 系统统一状态 |
| --- | --- |
| `submitted`、`waiting`、`queued` | `queued` |
| `running`、`generating`、`processing` | `processing` |
| `completed`、`success` | 转存完成后为 `succeeded` |
| `error`、`rejected`、`failed` | `failed` |

## 10. 任务数据模型

每个任务至少记录以下数据：

- 任务 ID。
- 用户 ID。
- 生成类型。
- 提示词。
- 输入图片地址。
- 模型厂商和模型名称。
- 视频比例和时长。
- 当前统一状态。
- 模型厂商任务 ID 和厂商原始状态。
- 生成进度。
- 结果视频地址。
- 错误代码、用户错误信息和厂商原始错误。
- 尝试次数和来源任务 ID。
- 幂等键。
- 创建、更新、开始生成和完成时间。

示例：

```json
{
  "id": "task_001",
  "userId": "local_user",
  "type": "image_to_video",
  "prompt": "橘猫在雨中的街道向前奔跑，镜头缓慢跟随",
  "inputImageUrl": "/storage/images/image_001.png",
  "provider": "mock",
  "model": "mock-video-v1",
  "ratio": "16:9",
  "duration": 5,
  "status": "processing",
  "progress": null,
  "providerTaskId": "provider_123",
  "providerStatus": "running",
  "resultVideoUrl": null,
  "errorCode": null,
  "errorMessage": null,
  "providerError": null,
  "attemptCount": 1,
  "retryOfTaskId": null,
  "idempotencyKey": "7bd7a2a2-3282-4a43-ae35-892e68195683",
  "createdAt": "2026-08-19T10:00:00+08:00",
  "updatedAt": "2026-08-19T10:00:05+08:00",
  "startedAt": "2026-08-19T10:00:03+08:00",
  "completedAt": null
}
```

## 11. 网站后端接口

前端使用本系统的 `taskId`。`providerTaskId` 仅用于后端与模型厂商通信，不向前端暴露。

### 11.1 上传参考图片

```http
POST /api/uploads/images
Content-Type: multipart/form-data
```

成功响应：

```json
{
  "imageUrl": "/storage/images/image_001.png",
  "width": 1280,
  "height": 720,
  "contentType": "image/png"
}
```

### 11.2 创建生成任务

```http
POST /api/video-tasks
Content-Type: application/json
Idempotency-Key: 7bd7a2a2-3282-4a43-ae35-892e68195683
```

请求：

```json
{
  "type": "image_to_video",
  "prompt": "橘猫在雨中的街道向前奔跑，镜头缓慢跟随",
  "imageUrl": "/storage/images/image_001.png",
  "ratio": "16:9",
  "duration": 5,
  "model": "mock-video-v1"
}
```

响应：

```json
{
  "id": "task_001",
  "status": "pending",
  "createdAt": "2026-08-19T10:00:00+08:00"
}
```

### 11.3 查询单个任务

```http
GET /api/video-tasks/{taskId}
```

### 11.4 查询历史记录

```http
GET /api/video-tasks?page=1&pageSize=20
```

历史记录按创建时间倒序分页返回。

### 11.5 重新生成

```http
POST /api/video-tasks/{taskId}/retry
```

重新生成会复制原任务输入并创建一个新任务，通过 `retryOfTaskId` 关联原任务。

## 12. 模型适配器接口

业务代码不能直接依赖某一家模型厂商，所有模型实现必须遵守统一接口。

```typescript
type VideoTaskStatus = "queued" | "processing" | "succeeded" | "failed";

interface CreateVideoInput {
  type: "image_to_video";
  prompt: string;
  imageUrl: string;
  ratio: "16:9" | "9:16" | "1:1";
  duration: 5;
  model: string;
}

interface CreateVideoResult {
  providerTaskId: string;
  status: "queued" | "processing";
}

interface VideoTaskResult {
  providerTaskId: string;
  status: VideoTaskStatus;
  progress: number | null;
  videoUrl: string | null;
  errorCode: string | null;
  errorMessage: string | null;
  rawStatus?: string;
}

interface VideoModelProvider {
  createTask(input: CreateVideoInput): Promise<CreateVideoResult>;
  getTask(providerTa  skId: string): Promise<VideoTaskResult>;
}
```

计划实现：

```text
VideoModelProvider
  ├─ MockVideoProvider       模拟模型
  ├─ JimengVideoProvider     即梦或火山引擎模型
  ├─ RunwayVideoProvider     Runway 模型
  └─ LocalVideoProvider      本地开源模型
```

### 12.1 提交任务示例

提交成功：

```json
{
  "providerTaskId": "provider_task_123",
  "status": "queued"
}
```

提交失败：

```json
{
  "errorCode": "PROVIDER_UNAVAILABLE",
  "errorMessage": "模型服务暂时无法接受生成任务"
}
```

### 12.2 查询任务示例

生成中：

```json
{
  "providerTaskId": "provider_task_123",
  "status": "processing",
  "progress": null,
  "videoUrl": null,
  "errorCode": null,
  "errorMessage": null
}
```

生成成功：

```json
{
  "providerTaskId": "provider_task_123",
  "status": "succeeded",
  "progress": 100,
  "videoUrl": "https://model-provider.example/video.mp4",
  "errorCode": null,
  "errorMessage": null
}
```

生成失败：

```json
{
  "providerTaskId": "provider_task_123",
  "status": "failed",
  "progress": null,
  "videoUrl": null,
  "errorCode": "CONTENT_REJECTED",
  "errorMessage": "输入内容未通过模型服务的安全检查"
}
```

## 13. MockVideoProvider 行为

初期首先实现 `MockVideoProvider`，它不调用真实模型，不产生 API 费用，但必须模拟真实异步流程。

```text
提交任务
  ↓
返回 providerTaskId 和 queued
  ↓
等待数秒
  ↓
返回 processing
  ↓
等待数秒
  ↓
返回固定测试视频和 succeeded
```

Mock Provider 还需要支持可控失败，例如当提示词包含约定测试词 `mock-fail` 时返回 `failed`，以便验证失败页面、错误提示和重试流程。

Mock Provider 和真实 Provider 必须实现同一个接口，切换 Provider 时不修改页面业务逻辑。

## 14. 后台查询、超时与恢复

- 前端只轮询本系统后端，不轮询模型厂商。
- 后端负责提交模型任务和查询厂商状态。
- `queued` 和 `processing` 任务需要由后台工作进程继续处理。
- 查询间隔采用逐步退避，避免频繁调用厂商接口。
- 单次网络或查询失败不立即将任务标记为 `failed`。
- 连续查询失败达到上限后，记录统一错误。
- 超过最大生成时间后，将任务标记为 `POLLING_TIMEOUT`。
- 服务重启后，后台工作进程必须扫描并恢复未完成任务。
- 同一任务不能被多个工作进程重复提交。
- 下载或转存视频失败时应进行有限次数重试。

第一版的具体查询间隔、超时时间和重试次数在实现时配置为环境变量或集中配置项，不散落在业务代码中。

## 15. 异常路径

- 提示词为空或超过长度限制：拒绝提交并提示用户修改。
- 未上传参考图片：拒绝提交。
- 图片格式、体积或尺寸不合规：提示用户重新上传。
- 图片上传失败：不创建生成任务。
- 重复点击提交：通过按钮状态和幂等键避免重复任务。
- 模型暂时不可用：显示服务繁忙，允许稍后重试。
- 内容安全审核失败：说明输入不符合要求，不自动重试。
- 模型生成失败：显示可理解的原因并允许重新生成。
- 页面刷新或关闭后重新打开：任务继续存在，并继续查询状态。
- 后端服务重启：未完成任务可以恢复处理。
- 厂商视频链接过期：系统使用已经转存的结果，不永久依赖厂商链接。
- 结果转存失败：任务不能显示成功，系统尝试有限次数重试。

## 16. 错误代码

| 错误代码 | 含义 | 是否适合自动重试 |
| --- | --- | --- |
| `INVALID_INPUT` | 输入参数不合法 | 否 |
| `INVALID_IMAGE` | 图片格式、尺寸或内容不合法 | 否 |
| `CONTENT_REJECTED` | 内容安全审核未通过 | 否 |
| `PROVIDER_UNAVAILABLE` | 模型厂商暂时不可用 | 是 |
| `PROVIDER_RATE_LIMITED` | 模型厂商调用频率受限 | 是 |
| `GENERATION_FAILED` | 视频生成失败 | 视具体错误而定 |
| `POLLING_TIMEOUT` | 查询生成结果超时 | 是 |
| `TRANSFER_FAILED` | 结果视频转存失败 | 是 |
| `INTERNAL_ERROR` | 系统内部错误 | 是 |

前端展示用户可理解的错误信息；日志中保存厂商原始状态和错误，但不能向用户暴露密钥、内部堆栈或敏感信息。

## 17. 重试与幂等规则

- `failed` 任务不能重新变为 `processing`。
- 用户点击“重新生成”时创建一个新任务。
- 新任务通过 `retryOfTaskId` 关联原任务。
- 同一次创建请求使用 `Idempotency-Key` 保证幂等性。
- 短时间内重复发送相同幂等键时返回同一个任务。
- 厂商限流、短暂网络错误等临时错误可以由系统自动重试。
- 参数错误、图片错误和内容安全错误不自动重试。
- 自动重试必须设置最大次数，不能无限重试。

## 18. 文件存储规则

- 开发阶段可以使用本地目录存储输入图片和结果视频。
- 存储接口需要预留未来迁移到 S3、OSS 或 COS 的可能性。
- 文件名由系统生成，不能直接使用用户上传的原始文件名。
- 数据库保存稳定的本系统文件地址，不永久保存厂商临时链接作为结果地址。
- 删除策略、保留时间和对象存储生命周期不属于第一版功能，但需要避免临时文件无限增长。

## 19. 安全要求

- API Key 只保存在后端环境变量或密钥服务中。
- `.env` 和真实密钥不得提交到 Git 仓库。
- 前端不能直接调用模型厂商。
- 后端必须重新校验所有前端输入。
- 后端必须校验上传文件的真实类型和体积。
- 文件名和存储路径由系统生成，防止路径穿越。
- 对提示词和图片执行模型厂商要求的内容安全规则。
- 不向前端返回厂商完整错误、内部异常堆栈或系统路径。
- 日志不能记录 API Key 或完整鉴权请求头。

## 20. 非功能目标

第一版不追求大规模并发，但需要满足以下基本要求：

- 页面刷新后数据不丢失。
- 后端接口返回结构保持一致。
- 模型厂商故障不会导致主服务崩溃。
- 核心操作有必要日志，能够定位任务失败阶段。
- 配置通过环境变量或集中配置管理。
- 模型、数据库和存储实现可替换，不与页面强耦合。
- 主要业务逻辑具备基础自动化测试。

## 21. MVP 验收标准

满足以下条件，即视为第一版 MVP 完成：

- 用户可以上传一张合规的参考图片。
- 用户可以输入提示词并选择视频比例。
- 后端可以创建并持久化任务。
- 重复提交不会意外创建多个相同任务。
- `MockVideoProvider` 可以模拟排队、生成、成功和失败。
- 页面可以展示任务当前状态。
- 页面刷新后任务仍然存在。
- 生成成功后可以在线播放和下载视频。
- 生成失败后可以看到可理解的错误信息。
- 失败任务可以通过创建新任务重新生成。
- 历史记录页可以查看以前的任务。
- 结果视频保存在本系统存储中，而不是永久依赖厂商链接。
- 后端重启后，未完成任务可以继续处理或被正确恢复。
- 切换模型 Provider 时，不需要修改前端页面的业务逻辑。
- 密钥不会出现在前端代码、接口响应和 Git 仓库中。

## 22. 实现顺序

### 里程碑 1：项目骨架

- 创建前端和后端项目。
- 配置基础开发环境。
- 实现健康检查。
- 确定统一配置方式。

### 里程碑 2：本地任务闭环

- 建立任务数据模型和数据库。
- 实现图片上传和本地存储。
- 实现创建任务、查询任务和历史记录接口。
- 实现 `MockVideoProvider`。
- 实现后台状态推进和测试视频返回。

### 里程碑 3：前端页面闭环

- 实现创作页。
- 实现任务结果页和状态轮询。
- 实现历史记录页。
- 实现播放、下载、失败提示和重新生成。

### 里程碑 4：可靠性

- 增加幂等处理。
- 增加后台重试、超时和任务恢复。
- 增加输入校验、错误映射和基础测试。
- 验证刷新页面和重启服务后的行为。

### 里程碑 5：接入真实模型

- 选择第一个真实模型厂商。
- 实现对应的 `VideoModelProvider`。
- 配置密钥和内容安全要求。
- 验证提交、查询、转存、失败和限流场景。
- 记录单次生成的耗时和成本。

## 23. 第一项开发任务

第一项开发任务不是立即调用真实模型，而是创建项目骨架并完成一个最小技术验证：

1. 前端能够调用后端健康检查接口。
2. 后端能够连接数据库。
3. 后端能够创建一条本地 `pending` 任务。
4. 页面能够通过任务 ID 查询并展示这条任务。

完成后再实现 `MockVideoProvider`，逐步打通完整异步链路。
