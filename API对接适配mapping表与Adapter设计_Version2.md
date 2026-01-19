# APIAI / GPTNB 平台能力与产品功能 mapping & adapter对接设计

## 1. 多功能聚合平台PRD与第三方API能力字段逐一对标表

| 核心产品功能           | 对应 API 板块      | APIAI 可对接接口                     | GPTNB 可对接接口                    | 主要字段映射            | 重要适配点备注                |
|---------------------|-----------------|----------------------------------|------------------------------------|----------------------|--------------------------|
| 多模型对话/AI聊天        | chat/completions  | `/v1/chat/completions`           | `/chat/completions` 等             | model, messages, temperature, top_p, stream | Provider名称、认证token、上下文格式统一 |
| 代码生成/写作/内容创作    | chat/completions  | `/v1/chat/completions`           | `/chat/completions`, `/completions`| model, prompt/messages, temperature, functions/format| 支持function call、代码块格式、tokens计数 |
| 多模态模型切换           | chat/completions, image/generate | model参数传递（如gpt-4, baichuan等）   | model参数/target_model切换           | model, key, api_base  | 统一Provider抽象与切换，key优先级叠加 |
| 文生图/图生图(AIGC绘画)   | image/generate     | `/v1/images/generations`, `/v1/sd`| `/image/generations`, `/image/sd`  | prompt, style, size, n, model, seed| 输出url格式、分辨率映射、批量图片组 |
| 局部重绘/高级编辑         | sd/inpainting,image/variation | `/v1/sd/inpaint`/`variation`      | `/image/sd/inpaint`（如有）        | mask, original_image, prompt, strength, steps | 文件上传方式匹配，输出格式一致化 |
| PPT/表格内容/文档生成      | document/generate  | `/v1/ppt/generate`, `/v1/excel`   | `/ppt/generate`, `/table/generate` | template_type, content, style, sheet_title   | 前端需将结构化内容填入模板PPT/Excel |
| 视频AI生成                | video/generate     | `/v1/video/generate`（火山、文心）  | `/video/generate`（部分模型）      | images, script, duration, style, transitions | 长耗时异步/回调、CDN地址管理 |
| 音频/AI配乐/语音合成       | audio/tts, audio/music | `/v1/audio/tts`, `/v1/music`     | `/audio/tts`, `/audio/music`       | prompt/text, style, duration, format        | 大文件合成，编码参数适配 |
| 智能体/流程编排           | agent/robot, function call/flow | `/v1/robot`, `/v1/agent-flow`    | `/agent/flow`, function_call       | intent, steps, functions, context           | 节点串联/并发，异常捕捉返回状态 |
| 多账号/密钥/计费/限额       | key, user, usage    | `/v1/key`, `/v1/user/usage`       | `/key`, `/usage`                   | key, stat, limits, usage, balance           | token计价单位、超限错误兼容 |
| 企业/私有化/安全           | 部署与账号管理板块    | 文档提供私有化/速率限制/部署接口        | 文档同上                            | install, monitor, qps, logs, roles          | 适配企业部署文档，带监控钩子 |

---

## 2. Adapter 实现示例（伪代码，包含核心 mapping 逻辑）

### 2.1 Provider/Adapter 抽象定义（TypeScript风格）

```typescript
interface AIProviderAdapter {
  // 文本/对话生成
  chat(options: ChatRequestOptions): Promise<ChatResponse>;

  // 图片生成
  generateImage(options: ImageRequestOptions): Promise<ImageResponse>;

  // PPT/表格生成
  generateDoc(options: DocGenOptions): Promise<DocGenResponse>;

  // 视频、音频生成
  generateVideo(options: VideoOptions): Promise<VideoResponse>;
  generateAudio(options: AudioOptions): Promise<AudioResponse>;

  // 智能体/函数流编排
  invokeAgentFlow(options: AgentFlowOptions): Promise<AgentResponse>;

  // 计费/密钥/用量管理
  getUsage(token: string): Promise<UsageStat>;
  checkQuota(token: string): Promise<boolean>;

  // ...更多API能力可扩展
}
```

### 2.2 APIAI Adapter（具体适配，mapping字段）

```typescript
class APIAIAdapter implements AIProviderAdapter {
  constructor(private apiKey: string, private apiBase: string = 'https://apiai.apifox.cn/v1/') {}

  async chat(options: ChatRequestOptions) {
    const resp = await fetch(this.apiBase + 'chat/completions', {
      method: 'POST',
      headers: { Authorization: `Bearer ${this.apiKey}`, 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: options.model,
        messages: options.messages,
        temperature: options.temperature,
        // 其它参数按需适配mapping
      })
    });
    // 字段映射/解包标准化
    const raw = await resp.json();
    return {
      text: raw.choices?.[0]?.message?.content ?? '',
      usage: raw.usage,
      id: raw.id,
      provider: 'APIAI',
    };
  }
  // 其它接口同理:
  async generateImage(opts) { /* ...调用 image/generate，字段对照上表 ...*/ }
  async generateDoc(opts) { /* ...ppt/excel生成... */ }
  // ...
}
```

### 2.3 GPTNB Adapter（与APIAI一致的接口抽象）

```typescript
class GPTNBAdapter implements AIProviderAdapter {
  constructor(private apiKey: string, private apiBase: string = 'https://api.gptnb.ai/v1/') {}

  async chat(options: ChatRequestOptions) {
    const resp = await fetch(this.apiBase + 'chat/completions', {
      method: 'POST',
      headers: { Authorization: `Bearer ${this.apiKey}`, 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: options.model,
        messages: options.messages,
        temperature: options.temperature
      })
    });
    const raw = await resp.json();
    return {
      text: raw.choices?.[0]?.message?.content ?? '',
      usage: raw.usage,
      id: raw.id,
      provider: 'GPTNB',
    };
  }
  // 其它接口同理:
  async generateImage(opts) { /* ...调用 image/generations ...*/ }
  async generateDoc(opts) { /* ...ppt/table生成... */ }
  // ...
}
```

---

### 3. 通用字段说明（产品与API字段对照）

| 产品参数/能力            | API标准字段            | 备注                               |
|---------------------|-------------------|----------------------------------|
| 模型名称/类型            | "model"           | gpt-4/gpt-3.5/Spark/文心等           |
| prompt/prompt_list  | "prompt"/"messages"| 支持列表、字符串或角色设定             |
| 温度                  | "temperature"      | 控制输出多样性                         |
| 图片分辨率/风格           | "size", "style"      | 图像API专用，例: "1024x1024"/"二次元"   |
| 多轮上下文               | "messages"          | 聊天历史格式化标准化                    |
| 批量生成数               | "n"                 | 批量图片/视频/txt数                     |
| 文件上传（重绘/变体等）      | "image", "mask"      | 文件流base64/URL/ID                     |
| 输出/导出格式             | "format"            | "png"/"svg"/"pdf"/"mp3"/"pptx"等        |
| 计费统计/用量             | "usage"/"stat"      | API返回内带，按调用/耗时/token等单位计数  |
| 错误处理/超限             | "error"             | 统一mapping后端异常与前端提示             |
| 认证/密钥                | headers/参数         | 前端传入优先覆盖全局配置                 |
| 多Provider/分流能力        | "provider"/"channel" | 前端/后端模型切换 用于弹窗和回退策略      |

---

### 4. 高级适配设计建议
- 对所有API调用实现响应/错误统一封装，包括`error_code`标准化和降级处理。
- 支持API自注册结构（注册新聚合/第三方接口只需填一份schema, 新建Adapter类）。
- 所有参数/返回需支持schema校验与版本比对，保证后续第三方字段变动时热升级。
- 支持API级别、用户级别多key限额及额度可见化。
- 所有Provider适配输出都遵循统一interafce，供前端/流程编排/计费/监控等模块共用。

---

如需针对某一功能（如视频生成、文案生成、知识库问答）展开更深入mapping与后端实现DEMO，请指定细分模块！