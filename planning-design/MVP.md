# learn\-updream

# MVP



## 产品目标

用户上传一张参考图片，并输入描述图片如何运动的提示词，
系统生成约 5 秒的视频，支持查看进度、在线播放、下载和历史记录。


## 目标用户与运行方式

第一版面向项目开发者本人，用于学习和验证 AI 视频生成网站的完整流程。

- 第一阶段在本地运行
- 第一阶段不实现注册和登录
- 使用固定的本地测试用户
- 历史记录保存在数据库中
- 后续部署上线时再加入用户注册、登录、额度和权限系统


## 用户操作路径

进入首页

↓

点击“开始创作”

↓

选择生成方式

└─文＋图生成视频

↓

输入提示词

↓

上传参考图

↓

设置生成参数

├─视频比例：16:9 / 9:16 / 1:1

├─时长：5秒

└─模型：默认模型

↓

点击“生成视频”

↓

系统创建任务

↓

查看状态

├─排队中

├─生成中

├─生成成功

└─生成失败

↓

预览生成结果

↓

下载视频或重新生成

↓

在“历史记录”中再次查看



## 第一版功能与不做的功能

### 

功能：根据提示词＋参考图生成视频

不做的功能：（仅有）提示词生成视频、（仅有）参考图生成视频



## 页面清单



1. 创作页

输入提示词、上传图片、设置参数并提交任务。

2. 任务结果页

   展示排队进度、错误信息和最终视频

3. 历史记录页

   展示以前的生成任务，并支持播放和下载



## 异常路径

- 提示词为空：不能提交

- 图片格式或体积不合规：提示用户重新上传

- 模型生成失败：显示原因并允许重试

- 页面刷新：任务不能消失，继续查询生成状态

- 视频链接过期：系统从自己的存储读取，而不是永久依赖模型厂商链接



## 任务状态定义

### 定义

| 状态值       | 中文名称 | 含义                       | 用户界面           |
| ------------ | -------- | -------------------------- | ------------------ |
| `pending`    | 等待提交 | 任务已创建，尚未提交给模型 | 正在准备           |
| `queued`     | 排队中   | 已提交，等待模型处理       | 排队中             |
| `processing` | 生成中   | 模型正在生成视频           | 生成中             |
| `succeeded`  | 生成成功 | 视频已生成并保存           | 播放、下载         |
| `failed`     | 生成失败 | 模型调用或文件保存失败     | 显示错误、允许重试 |
| `cancelled`  | 已取消   | 用户或系统取消了任务       | 已取消             |

### 状态转换

pending

↓

queued

↓

processing

├─ succeeded

├─ failed

└─ cancelled



正常路径

pending → queued → processing → succeeded



失败路径

pending → failed

queued → failed

processing → failed



取消路径

queued → cancelled

processing → cancelled



### 每个任务需要记录的数据

任务 ID

用户 ID

生成类型：文生视频/图生视频

提示词

参考图片地址

模型名称

视频比例

视频时长

当前状态

模型厂商任务 ID

生成进度

结果视频地址

错误代码

错误信息

创建时间

开始生成时间

完成时间



可以表示成这样的数据：

```JSON
{
  "id": "task_001",
  "type": "text_to_video",
  "prompt": "一只橘猫在雨中的城市街道奔跑",
  "model": "default-video-model",
  "ratio": "16:9",
  "duration": 5,
  "status": "processing",
  "progress": 40,
  "providerTaskId": "provider_123",
  "videoUrl": null,
  "errorCode": null,
  "errorMessage": null,
  "createdAt": "2026-08-19T10:00:00+08:00",
  "completedAt": null
}
```



### 状态处理规则

- 用户提交后，系统立即创建`pending`任务

- 成功提交给模型后，变为`queued`

- 模型开始处理后，变为`processing`

- 视频生成并转存成功后，变为`succeeded`

- 任意步骤发生不可恢复的错误，变为`failed`

- 页面页面刷新不能改变任务状态

- 后端负责查询模型状态，前端只展示后端的返回状态

- `succeeded`、`failed`和`cancelled`属于终态，正常情况下不再变化



## 模型接口定义



### 1\.提交视频生成任务

接口名称：createVideoTask

请求数据：

```JSON
{
  "type": "text_to_video",
  "prompt": "一只橘猫在雨中的城市街道中奔跑",
  "imageUrl": null,
  "ratio": "16:9",
  "duration": 5,
  "model": "default-video-model"
}
```

字段说明：

提交成功后返回：

```JSON
{
  "providerTaskId": "provider_task_123",
  "status": "queued"
}
```

提交失败后返回：

```JSON
{
  "errorCode": "SUBMIT_FAILED",
  "errorMessage": "模型暂时无法接受生成任务"
}
```

### 2\.查询视频生成状态

接口名称：getVideoTask

请求数据：

```JSON
{
  "providerTaskId": "provider_task_123"
}
```

生成中返回：

```JSON
{
  "providerTaskId": "provider_task_123",
  "status": "processing",
  "progress": 40,
  "videoUrl": null,
  "errorMessage": null
}
```

生成成功返回：

```JSON
{
  "providerTaskId": "provider_task_123",
  "status": "succeeded",
  "progress": 100,
  "videoUrl": "https://model-provider.example/video.mp4",
  "errorMessage": null
}
```

生成失败返回：

```JSON
{
  "providerTaskId": "provider_task_123",
  "status": "failed",
  "progress": null,
  "videoUrl": null,
  "errorMessage": "提示词不符合内容安全要求"
}
```

### 3\.统一任务状态

不同模型厂商使用的状态名称可能不同，因此系统需要转换成自己的统一状态：

### 4\.模型适配器

系统不应该让业务代码直接依赖某一家模型厂商，而是定义统一接口：
```TypeScript
interface VideoModelProvider {
  createTask(input: CreateVideoInput): Promise<CreateVideoResult>;
  getTask(providerTaskId: string): Promise<VideoTaskResult>;
}
```

未来接入不同模型时分别实现：
VideoModelProvider
├─ MockVideoProvider       模拟模型
├─ JimengVideoProvider     即梦模型
├─ RunwayVideoProvider     Runway 模型
└─ LocalVideoProvider      本地开源模型


## 系统职责边界

本系统负责：

- 接收提示词和参考图片
- 校验、上传并保存参考图片
- 创建本地任务记录
- 调用视频模型厂商接口
- 定时查询生成状态
- 将厂商状态转换为统一状态
- 下载并保存生成结果
- 向前端提供任务状态和视频地址
- 保存生成历史

模型厂商负责：

- 接收生成参数
- 执行视频生成
- 返回厂商任务 ID
- 返回任务状态、错误信息和临时视频地址


## 核心数据流程

前端不能直接调用模型厂商，也不能接触厂商 API Key。

用户提交图片和提示词
→ 图片上传到本系统存储
→ 后端创建 pending 任务
→ 后端调用 VideoModelProvider
→ 获得厂商任务 ID
→ 后台轮询厂商任务状态
→ 厂商生成成功
→ 后端下载并转存视频
→ 本地任务变为 succeeded
→ 前端播放本系统保存的视频


## MVP验收标准
模型接口满足以下条件即算完成：
- 能提交一条提示词+参考图生成视频任务
- 提交后能获得厂商任务ID
- 能通过厂商任务ID查询状态
- 成功后能获得视频地址
- 失败后能获得可读的错误信息
- 模拟模型和接口模型使用相同的接口模式

初期我们先实现 MockVideoProvider。它不调用真实模型，而是模拟：
提交任务
   ↓
等待几秒
   ↓
状态从 queued 变为 processing
   ↓
返回一个测试视频