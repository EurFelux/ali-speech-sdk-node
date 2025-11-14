# ASR 统一接口设计文档

## 1. API 对比分析

### 1.1 三个 API 概览

| 特性 | Paraformer | FunASR | Qwen-ASR |
|------|-----------|---------|----------|
| **API 端点** | `/api/v1/services/audio/asr/transcription` | `/api/v1/services/audio/asr/transcription` | `/api/v1/services/aigc/multimodal-generation/generation` |
| **认证方式** | Bearer Token | Bearer Token | Bearer Token |
| **调用模式** | 异步任务 | 异步任务 | 同步调用 |
| **音频输入** | 仅 URL | 仅 URL | URL 或本地文件路径 |
| **文件大小限制** | ≤2GB | ≤2GB | 未明确限制 |
| **时长限制** | ≤12小时 | ≤12小时 | 未明确限制 |
| **批量处理** | 最多100个URL | 最多100个URL | 单个文件 |

### 1.2 核心差异

#### **1. 调用模式差异**

**Paraformer & FunASR (异步任务模式):**
```
1. 提交任务 → 获得 task_id
2. 轮询查询 task_id → 获取任务状态
3. 任务完成 → 从 transcription_url 获取结果
```

**Qwen-ASR (同步调用模式):**
```
1. 发送请求 → 直接返回识别结果
```

#### **2. 请求格式差异**

**Paraformer & FunASR:**
```json
{
  "model": "paraformer-v2",
  "file_urls": ["http://example.com/audio.wav"],
  "vocabulary_id": "vocab-id",
  "diarization_enabled": true
}
```

**Qwen-ASR:**
```json
{
  "model": "qwen3-asr-flash",
  "messages": [
    {
      "role": "system",
      "content": [{"text": "系统提示"}]
    },
    {
      "role": "user",
      "content": [
        {"audio": "http://example.com/audio.wav"}
      ]
    }
  ]
}
```

#### **3. 功能特性对比**

| 功能 | Paraformer | FunASR | Qwen-ASR |
|------|-----------|---------|----------|
| **多语言支持** | ✅ V2支持18种语言 | ❌ 主要中文 | ✅ 18种语言 |
| **说话人分离** | ✅ diarization_enabled | ✅ diarization_enabled | ❌ |
| **热词定制** | ✅ vocabulary_id | ✅ vocabulary_id | ✅ context (更强大) |
| **时间戳** | ✅ 词/句/段落级 | ✅ 词/句级 | ❌ |
| **语气词过滤** | ✅ disfluency_removal_enabled | ❌ | ❌ |
| **ITN文本规范化** | ❌ | ❌ | ✅ (仅 Qwen3-ASR) |
| **情感检测** | ❌ | ❌ | ✅ (仅 Qwen3-ASR) |
| **上下文增强** | ❌ | ❌ | ✅ 最多10000 tokens |

### 1.3 共同点

1. **认证方式统一**: 都使用 `Authorization: Bearer {api-key}`
2. **支持相似的音频格式**: aac、amr、flac、mp3、mp4、wav、webm 等
3. **都支持热词/上下文定制**
4. **结果都包含文本转写**
5. **都基于 DashScope 平台**

### 1.4 重要限制：不支持 Base64/Buffer 直传

**关键发现：三个 API 都不支持 Base64 编码或 Buffer 直传**

- **Paraformer & FunASR**: 仅支持公网可访问的 HTTP/HTTPS URL
- **Qwen-ASR**: 支持公网 URL 或本地文件**绝对路径**

**影响：**
- 无法直接处理前端录音的 Base64 数据
- 无法处理内存中的音频 Buffer
- Paraformer/FunASR 必须先将音频上传到可公网访问的位置

**解决方案：**
SDK 提供可选的 OSS 自动上传功能：
- 用户配置 OSS 后，SDK 自动上传 Base64/Buffer 到 OSS 并生成临时 URL
- 如果未配置 OSS，使用 Base64/Buffer 将抛出运行时错误
- Qwen-ASR 可以通过写入临时本地文件来支持 Buffer（无需 OSS）

---

## 2. 统一接口设计

### 2.1 设计原则

1. **抽象差异**: 隐藏异步/同步调用的差异，提供统一的 Promise-based API
2. **参数归一化**: 将不同 API 的参数映射到统一的接口
3. **类型安全**: 充分利用 TypeScript 提供类型提示和校验
4. **可扩展性**: 支持添加新的 ASR 服务而不破坏现有代码
5. **错误处理**: 统一错误格式和错误处理机制

### 2.2 架构设计

```
┌─────────────────────────────────────────────────────────┐
│              AliSpeechClient (统一入口)                   │
│  - asr: ASRService                                      │
│  - tts: TTSService (未来)                               │
│  - config: ClientConfig                                 │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
┌───────▼──────────┐    ┌─────────▼────────┐
│   ASRService     │    │   TTSService     │ (未来)
│- transcribe()    │    │- synthesize()    │
│- transcribeBatch()│   └──────────────────┘
│- _handleAudio()  │
└────────┬─────────┘
         │
         │ 使用
         ▼
┌─────────────────┐
│  ASRProvider    │ (抽象基类)
│- transcribe()   │
└────────┬────────┘
         │
  ┏━━━━━┻━━━━━━━━━━━━━━━━━┓
  ┃                        ┃
┌─▼────────────┐  ┌────────▼───────┐  ┌──────────▼────────┐
│ Paraformer   │  │    FunASR      │  │    QwenASR        │
│ Provider     │  │   Provider     │  │   Provider        │
│- transcribe()│  │- transcribe()  │  │- transcribe()     │
│- submitTask()│  │- submitTask()  │  │- directCall()     │
│- pollTask()  │  │- pollTask()    │  └───────────────────┘
└──────────────┘  └────────────────┘
      │                  │
      └────────┬─────────┘
               │
      ┌────────▼────────┐
      │  OSSUploader    │ (共享工具类)
      │- upload()       │
      │- cleanup()      │
      └─────────────────┘
```

### 2.2.1 音频输入处理流程

SDK 会根据音频输入类型和 Provider 要求自动处理：

```
用户输入 audio
    │
    ├─ 类型检测
    │   ├─ HTTP/HTTPS URL → 直接使用
    │   ├─ file:// 协议 → 本地文件路径
    │   ├─ Base64 字符串 (data: 开头) → 需要转换
    │   ├─ 普通字符串路径 → 本地文件路径
    │   └─ Buffer → 需要处理
    │
    ├─ Provider 需求检测
    │   ├─ Paraformer/FunASR → 必须是公网 HTTP/HTTPS URL
    │   └─ Qwen-ASR → 支持 HTTP/HTTPS URL 或本地绝对路径
    │
    └─ 处理策略
        ├─ HTTP/HTTPS URL → 直接传递给 Provider
        │
        ├─ file:// 协议
        │   ├─ Qwen → 转换为绝对路径直接使用
        │   └─ Paraformer/FunASR → 读取文件并上传到 OSS
        │       ├─ 已配置 OSS → 上传并返回 URL
        │       └─ 未配置 OSS → 抛出 OSS_NOT_CONFIGURED 错误
        │
        ├─ 普通路径字符串 (相对/绝对路径)
        │   ├─ Qwen → 转换为绝对路径直接使用
        │   └─ Paraformer/FunASR → 读取文件并上传到 OSS
        │       ├─ 已配置 OSS → 上传并返回 URL
        │       └─ 未配置 OSS → 抛出 OSS_NOT_CONFIGURED 错误
        │
        ├─ Buffer
        │   ├─ Qwen → 写入临时本地文件
        │   └─ Paraformer/FunASR → 上传到 OSS
        │       ├─ 已配置 OSS → 上传并返回 URL
        │       └─ 未配置 OSS → 抛出 OSS_NOT_CONFIGURED 错误
        │
        └─ Base64 (data: 开头) → 解码为 Buffer，按 Buffer 处理
```

### 2.3 核心接口定义

#### **Zod Schema 定义（运行时验证）**

本 SDK 使用 **Zod v4** 进行运行时类型验证，所有类型定义都应从 Zod schema 推断。

```typescript
import { z } from "zod";

// 支持的语言 Schema
export const ASRLanguageSchema = z.enum([
	"zh",   // 中文（含方言）
	"en",   // 英语
	"ja",   // 日语
	"ko",   // 韩语
	"es",   // 西班牙语
	"fr",   // 法语
	"de",   // 德语
	"ru",   // 俄语
	"pt",   // 葡萄牙语
	"it",   // 意大利语
	"ar",   // 阿拉伯语
	"tr",   // 土耳其语
	"vi",   // 越南语
	"th",   // 泰语
	"id",   // 印尼语
	"ms",   // 马来语
	"hi",   // 印地语
	"fa"    // 波斯语
]);

// Provider Schema
export const ASRProviderSchema = z.enum(["paraformer", "funasr", "qwen"]);

// 模型 Schemas
export const ParaformerModelSchema = z.enum([
	"paraformer-v2",
	"paraformer-8k-v2",
	"paraformer-v1",
	"paraformer-8k-v1"
]);

export const FunASRModelSchema = z.enum([
	"fun-asr",
	"fun-asr-mtl-2025-08-25"
]);

export const QwenASRModelSchema = z.enum([
	"qwen3-asr-flash",
	"qwen-audio-asr"
]);

// 音频格式 Schema
export const AudioFormatSchema = z.enum([
	"aac", "amr", "avi", "flac", "flv",
	"m4a", "mkv", "mov", "mp3", "mp4",
	"mpeg", "ogg", "opus", "wav", "webm",
	"wma", "wmv"
]);

// 情感类型 Schema
export const EmotionSchema = z.enum([
	"happy",
	"sad",
	"angry",
	"neutral",
	"surprised",
	"fearful",
	"disgusted"
]);

// 从 Schema 推断 TypeScript 类型
export type ASRLanguage = z.infer<typeof ASRLanguageSchema>;
export type ASRProvider = z.infer<typeof ASRProviderSchema>;
export type ParaformerModel = z.infer<typeof ParaformerModelSchema>;
export type FunASRModel = z.infer<typeof FunASRModelSchema>;
export type QwenASRModel = z.infer<typeof QwenASRModelSchema>;
export type AudioFormat = z.infer<typeof AudioFormatSchema>;
export type Emotion = z.infer<typeof EmotionSchema>;
```

#### **ASRTranscriptionRequest Schema 和类型**

```typescript
// 基础请求 Schema
export const ASRTranscriptionRequestSchema = z.object({
	// 必需参数
	audio: z.union([
		z.string(),                      // URL, file://, 本地路径, Base64
		z.instanceof(Buffer)
	]),
	provider: ASRProviderSchema,

	// 可选通用参数
	language: ASRLanguageSchema.optional(),
	model: z.string().optional(),        // 灵活的模型字符串

	// 热词/上下文
	customVocabulary: z.array(z.string()).optional(),
	vocabularyId: z.string().optional(),
	context: z.string().max(10000, "上下文最多10000字符").optional(),

	// 识别增强功能
	enableDiarization: z.boolean().optional(),
	enableTimestamp: z.boolean().optional(),
	enableITN: z.boolean().optional(),
	enableEmotionDetection: z.boolean().optional(),
	removeDisfluency: z.boolean().optional(),

	// 高级选项
	channelId: z.array(z.number().int().nonnegative()).optional(),
	samplingRate: z.enum(["8000", "16000"]).transform(Number).optional(),

	// 轮询配置
	pollingInterval: z.number().int().min(100).max(60000).optional(),
	maxPollingAttempts: z.number().int().min(1).max(1000).optional()
});

// Provider 特定的 Schema (使用 discriminatedUnion)
export const ParaformerTranscriptionRequestSchema = ASRTranscriptionRequestSchema.extend({
	provider: z.literal("paraformer"),
	model: ParaformerModelSchema.optional(),
	language: ASRLanguageSchema.optional(),
	enableDiarization: z.boolean().optional(),
	enableTimestamp: z.boolean().optional(),
	removeDisfluency: z.boolean().optional(),
	vocabularyId: z.string().optional()
});

export const FunASRTranscriptionRequestSchema = ASRTranscriptionRequestSchema.extend({
	provider: z.literal("funasr"),
	model: FunASRModelSchema.optional(),
	enableDiarization: z.boolean().optional(),
	enableTimestamp: z.boolean().optional(),
	channelId: z.array(z.number().int().nonnegative()).optional(),
	vocabularyId: z.string().optional()
}).omit({ language: true });  // FunASR 不支持 language

export const QwenASRTranscriptionRequestSchema = ASRTranscriptionRequestSchema.extend({
	provider: z.literal("qwen"),
	model: QwenASRModelSchema.optional(),
	language: ASRLanguageSchema.optional(),
	context: z.string().max(10000).optional(),
	enableITN: z.boolean().optional(),
	enableEmotionDetection: z.boolean().optional()
}).omit({ enableDiarization: true, enableTimestamp: true });  // Qwen 不支持这些

// 联合 Schema (用于运行时验证)
export const ASRTranscriptionRequestUnionSchema = z.discriminatedUnion("provider", [
	ParaformerTranscriptionRequestSchema,
	FunASRTranscriptionRequestSchema,
	QwenASRTranscriptionRequestSchema
]);

// 导出推断类型
export type ASRTranscriptionRequest = z.infer<typeof ASRTranscriptionRequestSchema>;
export type ParaformerTranscriptionRequest = z.infer<typeof ParaformerTranscriptionRequestSchema>;
export type FunASRTranscriptionRequest = z.infer<typeof FunASRTranscriptionRequestSchema>;
export type QwenASRTranscriptionRequest = z.infer<typeof QwenASRTranscriptionRequestSchema>;
```

#### **ASRTranscriptionResponse (统一响应接口)**

```typescript
// 情感类型
type Emotion = "happy" | "sad" | "angry" | "neutral" | "surprised" | "fearful" | "disgusted";

interface ASRTranscriptionResponse {
	// 基本结果
	text: string;                        // 完整转写文本

	// 层级结果 (如果可用)
	sentences?: ASRSentence[];           // 句子级结果 (Paraformer/FunASR)
	words?: ASRWord[];                   // 词级结果 (Paraformer/FunASR)

	// 元数据
	metadata: {
		provider: ASRProvider;           // 使用的服务
		model: string;                   // 使用的模型
		language?: ASRLanguage;          // 检测到的语言
		duration?: number;               // 音频时长(秒)
		taskId?: string;                 // 任务ID (异步任务)
	};

	// 增强功能结果
	speakers?: ASRSpeaker[];             // 说话人信息 (Paraformer/FunASR)
	emotion?: Emotion;                   // 情感标注 (Qwen专用)

	// 原始响应 (调试用)
	raw?: unknown;
}

interface ASRSentence {
	text: string;
	startTime?: number;                  // 毫秒
	endTime?: number;
	speakerId?: string;
	confidence?: number;
}

interface ASRWord {
	text: string;
	startTime?: number;
	endTime?: number;
	confidence?: number;
}

interface ASRSpeaker {
	id: string;
	segments: Array<{
		startTime: number;
		endTime: number;
		text: string;
	}>;
}
```

### 2.4 使用示例

#### **基础使用 - URL 音频**

```typescript
import { AliSpeechClient } from "ali-speech-sdk";

const client = new AliSpeechClient({
	apiKey: "your-api-key",
	region: "cn-beijing" // or "ap-southeast-1"
});

// 示例1: 使用 Paraformer 进行多语言识别
const result = await client.asr.transcribe({
	audio: "https://example.com/audio.mp3",
	provider: "paraformer",
	model: "paraformer-v2",
	language: "zh",
	enableDiarization: true,
	enableTimestamp: true
});

console.log(result.text);
console.log(result.speakers);

// 示例2: 使用 Qwen-ASR 进行上下文增强识别（本地文件）
const result2 = await client.asr.transcribe({
	audio: "./local-audio.wav",  // Qwen 支持本地文件路径
	provider: "qwen",
	model: "qwen3-asr-flash",
	context: "这是一段关于人工智能的演讲，涉及大语言模型、深度学习等术语。",
	enableITN: true,
	enableEmotionDetection: true
});

console.log(result2.text);
console.log(result2.emotion);

// 示例3: 批量识别 (仅 Paraformer/FunASR)
const results = await client.asr.transcribeBatch({
	audios: [
		"https://example.com/audio1.mp3",
		"https://example.com/audio2.mp3"
	],
	provider: "funasr",
	enableTimestamp: true
});
```

#### **使用本地文件路径**

```typescript
import { AliSpeechClient } from "ali-speech-sdk";

const client = new AliSpeechClient({
	apiKey: "your-api-key",
	region: "cn-beijing"
});

// 示例1: 使用 file:// 协议（Qwen）
const result1 = await client.asr.transcribe({
	audio: "file:///absolute/path/to/audio.mp3",
	provider: "qwen"
});

// 示例2: 使用相对路径（Qwen）
const result2 = await client.asr.transcribe({
	audio: "./audio/sample.wav",
	provider: "qwen"
});

// 示例3: 使用绝对路径（Qwen）
const result3 = await client.asr.transcribe({
	audio: "/Users/username/Documents/audio.mp3",
	provider: "qwen"
});
```

#### **本地文件自动上传 OSS（Paraformer/FunASR）**

```typescript
import { AliSpeechClient } from "ali-speech-sdk";

// 配置 OSS 以支持本地文件自动上传
const client = new AliSpeechClient({
	apiKey: "your-api-key",
	region: "cn-beijing",
	oss: {
		accessKeyId: "your-oss-access-key-id",
		accessKeySecret: "your-oss-access-key-secret",
		bucket: "your-bucket-name",
		region: "oss-cn-beijing",
		tempPrefix: "asr-temp/",
		tempUrlExpiration: 3600,
		autoCleanup: true
	}
});

// 示例1: file:// 协议 - SDK 自动读取并上传到 OSS
const result1 = await client.asr.transcribe({
	audio: "file:///Users/username/audio.mp3",
	provider: "paraformer",
	enableDiarization: true
});

// 示例2: 相对路径 - SDK 自动读取并上传到 OSS
const result2 = await client.asr.transcribe({
	audio: "./recordings/meeting.wav",
	provider: "funasr"
});

// 示例3: 绝对路径 - SDK 自动读取并上传到 OSS
const result3 = await client.asr.transcribe({
	audio: "/tmp/audio-sample.mp3",
	provider: "paraformer",
	model: "paraformer-v2"
});
```

#### **使用 Buffer/Base64 - 需要配置 OSS**

```typescript
import { AliSpeechClient } from "ali-speech-sdk";
import * as fs from "fs";

// 配置 OSS 以支持 Buffer/Base64 自动上传
const client = new AliSpeechClient({
	apiKey: "your-dashscope-api-key",
	region: "cn-beijing",
	oss: {
		accessKeyId: "your-oss-access-key-id",
		accessKeySecret: "your-oss-access-key-secret",
		bucket: "your-bucket-name",
		region: "oss-cn-beijing",
		tempPrefix: "asr-temp/",           // 临时文件前缀
		tempUrlExpiration: 3600,            // URL 1小时过期
		autoCleanup: true                   // 自动清理临时文件
	}
});

// 示例1: 使用 Buffer (Paraformer)
const audioBuffer = fs.readFileSync("./audio.mp3");
const result = await client.asr.transcribe({
	audio: audioBuffer,                     // SDK 自动上传到 OSS
	provider: "paraformer",
	enableDiarization: true
});

// 示例2: 使用 Base64
const base64Audio = audioBuffer.toString("base64");
const result2 = await client.asr.transcribe({
	audio: base64Audio,                     // SDK 自动解码并上传
	provider: "funasr"
});

// 示例3: Qwen 使用 Buffer（无需 OSS，自动写入本地临时文件）
const clientWithoutOSS = new AliSpeechClient({
	apiKey: "your-api-key"
});

const result3 = await clientWithoutOSS.asr.transcribe({
	audio: audioBuffer,                     // SDK 写入临时本地文件
	provider: "qwen",                       // 仅 Qwen 支持
	enableITN: true
});
```

#### **错误处理**

```typescript
const client = new AliSpeechClient({
	apiKey: "your-api-key"
	// 未配置 OSS
});

// 错误1: 使用 Buffer/本地文件但未配置 OSS
try {
	const audioBuffer = fs.readFileSync("./audio.mp3");
	const result = await client.asr.transcribe({
		audio: audioBuffer,
		provider: "paraformer"  // Paraformer 需要 URL
	});
} catch (error) {
	if (error.code === "OSS_NOT_CONFIGURED") {
		console.error("使用 Buffer/Base64/本地文件 + Paraformer/FunASR 需要配置 OSS");
		console.error("解决方案: 1) 配置 OSS 或 2) 切换到 Qwen-ASR provider");
	}
}

// 错误2: 本地文件不存在
try {
	const result = await client.asr.transcribe({
		audio: "./non-existent-file.mp3",
		provider: "qwen"
	});
} catch (error) {
	if (error.code === "FILE_NOT_FOUND") {
		console.error(`音频文件未找到: ${error.message}`);
	}
}

// 错误3: 使用 file:// 协议但文件不存在
try {
	const result = await client.asr.transcribe({
		audio: "file:///path/to/missing.mp3",
		provider: "paraformer"
	});
} catch (error) {
	if (error.code === "FILE_NOT_FOUND") {
		console.error("文件不存在，无法上传到 OSS");
	} else if (error.code === "OSS_NOT_CONFIGURED") {
		console.error("需要配置 OSS 才能使用本地文件");
	}
}
```

#### **类型安全示例**

```typescript
import { AliSpeechClient } from "ali-speech-sdk";

const client = new AliSpeechClient({ apiKey: "xxx" });

// ✅ 正确：Paraformer 支持多语言和说话人分离
const result1 = await client.asr.transcribe({
	audio: "https://example.com/audio.mp3",
	provider: "paraformer",
	model: "paraformer-v2",
	language: "zh",                      // ✅ 类型安全：只能是支持的语言代码
	enableDiarization: true              // ✅ Paraformer 支持
});

// ✅ 正确：Qwen 支持上下文增强和情感检测
const result2 = await client.asr.transcribe({
	audio: "./audio.wav",
	provider: "qwen",
	model: "qwen3-asr-flash",            // ✅ 类型安全：只能是 Qwen 模型
	language: "en",                      // ✅ 支持
	context: "This is a medical conversation...",
	enableITN: true,                     // ✅ Qwen 支持
	enableEmotionDetection: true         // ✅ Qwen 支持
});

// ❌ 编译错误：FunASR 不支持 language 参数
const result3 = await client.asr.transcribe({
	audio: "https://example.com/audio.mp3",
	provider: "funasr",
	language: "zh",                      // ❌ TypeScript 错误
	enableDiarization: true
});

// ❌ 编译错误：Qwen 不支持说话人分离
const result4 = await client.asr.transcribe({
	audio: "./audio.wav",
	provider: "qwen",
	enableDiarization: true              // ❌ TypeScript 错误
});

// ❌ 编译错误：无效的语言代码
const result5 = await client.asr.transcribe({
	audio: "https://example.com/audio.mp3",
	provider: "paraformer",
	language: "invalid-lang"             // ❌ TypeScript 错误
});

// ❌ 编译错误：采样率只能是 8000 或 16000
const result6 = await client.asr.transcribe({
	audio: "https://example.com/audio.mp3",
	provider: "paraformer",
	samplingRate: 44100                  // ❌ TypeScript 错误
});
```

---

## 3. 实现策略

### 3.1 统一客户端入口

```typescript
class AliSpeechClient {
	private config: ClientConfig;
	public readonly asr: ASRService;
	// 未来扩展
	// public readonly tts: TTSService;

	constructor(config: ClientConfig) {
		// 验证配置
		const validationResult = ClientConfigSchema.safeParse(config);
		if (!validationResult.success) {
			throw new Error(
				`Invalid client configuration: ${validationResult.error.message}`
			);
		}

		this.config = validationResult.data;
		this.asr = new ASRService(this.config);
		// this.tts = new TTSService(this.config);
	}
}

// 使用
const client = new AliSpeechClient({ apiKey: "xxx" });
await client.asr.transcribe({ ... });
// await client.tts.synthesize({ ... });
```

### 3.2 ASRService 服务层

```typescript
class ASRService {
	private config: ClientConfig;
	private providers: Map<string, ASRProvider>;
	private ossUploader?: OSSUploader;

	constructor(config: ClientConfig) {
		this.config = config;

		// 初始化 Providers
		this.providers = new Map([
			["paraformer", new ParaformerProvider(config)],
			["funasr", new FunASRProvider(config)],
			["qwen", new QwenASRProvider(config)]
		]);

		// 初始化 OSS (如果配置了)
		if (config.oss) {
			this.ossUploader = new OSSUploader(config.oss);
		}
	}

	// 函数重载：根据 provider 提供不同的类型约束
	async transcribe(request: ParaformerTranscriptionRequest): Promise<ASRTranscriptionResponse>;
	async transcribe(request: FunASRTranscriptionRequest): Promise<ASRTranscriptionResponse>;
	async transcribe(request: QwenASRTranscriptionRequest): Promise<ASRTranscriptionResponse>;
	async transcribe(request: ASRTranscriptionRequest): Promise<ASRTranscriptionResponse>;
	async transcribe(request: ASRTranscriptionRequest): Promise<ASRTranscriptionResponse> {
		// 1. 运行时验证请求参数
		const validationResult = ASRTranscriptionRequestUnionSchema.safeParse(request);
		if (!validationResult.success) {
			throw new ASRError(
				"Invalid transcription request",
				"INVALID_REQUEST",
				request.provider,
				undefined,
				validationResult.error
			);
		}

		const validatedRequest = validationResult.data;

		// 2. 处理音频输入
		const audioUrl = await this._handleAudioInput(
			validatedRequest.audio,
			validatedRequest.provider
		);

		// 3. 获取对应的 Provider
		const provider = this.providers.get(validatedRequest.provider);
		if (!provider) {
			throw new ASRError(
				`Unknown provider: ${validatedRequest.provider}`,
				"INVALID_PROVIDER"
			);
		}

		// 4. 调用 Provider
		const result = await provider.transcribe({
			...validatedRequest,
			audio: audioUrl
		});

		// 5. 清理临时文件 (如果启用)
		if (this.config.oss?.autoCleanup && this.ossUploader) {
			await this.ossUploader.cleanup(audioUrl);
		}

		return result;
	}

	async transcribeBatch(request: ASRBatchRequest): Promise<ASRTranscriptionResponse[]> {
		// 批量处理逻辑
		const promises = request.audios.map(audio =>
			this.transcribe({ ...request, audio })
		);
		return Promise.all(promises);
	}

	private async _handleAudioInput(
		audio: string | Buffer,
		provider: string
	): Promise<string> {
		// 1. HTTP/HTTPS URL - 直接返回
		if (typeof audio === "string" && /^https?:\/\//i.test(audio)) {
			return audio;
		}

		// 2. file:// 协议
		if (typeof audio === "string" && audio.startsWith("file://")) {
			const filePath = this._fileUrlToPath(audio);
			return this._handleLocalFile(filePath, provider);
		}

		// 3. Base64 (data: 开头)
		if (typeof audio === "string" && audio.startsWith("data:")) {
			const buffer = this._toBuffer(audio);
			return this._handleBuffer(buffer, provider);
		}

		// 4. Buffer
		if (Buffer.isBuffer(audio)) {
			return this._handleBuffer(audio, provider);
		}

		// 5. 普通字符串路径（相对或绝对路径）
		if (typeof audio === "string") {
			return this._handleLocalFile(audio, provider);
		}

		throw new ASRError("Invalid audio input type", "INVALID_AUDIO_INPUT");
	}

	private async _handleLocalFile(filePath: string, provider: string): Promise<string> {
		// 转换为绝对路径
		const absolutePath = path.isAbsolute(filePath)
			? filePath
			: path.resolve(process.cwd(), filePath);

		// 检查文件是否存在
		if (!fs.existsSync(absolutePath)) {
			throw new ASRError(
				`Audio file not found: ${absolutePath}`,
				"FILE_NOT_FOUND"
			);
		}

		// Qwen: 直接使用本地路径
		if (provider === "qwen") {
			return absolutePath;
		}

		// Paraformer/FunASR: 需要上传到 OSS
		if (!this.ossUploader) {
			throw new ASRError(
				"OSS configuration is required to use local files with Paraformer/FunASR",
				"OSS_NOT_CONFIGURED"
			);
		}

		// 读取文件并上传
		const buffer = await fs.promises.readFile(absolutePath);
		return this.ossUploader.upload(buffer, path.basename(absolutePath));
	}

	private async _handleBuffer(buffer: Buffer, provider: string): Promise<string> {
		// Qwen: 写入临时本地文件
		if (provider === "qwen") {
			return this._writeTempFile(buffer);
		}

		// Paraformer/FunASR: 上传到 OSS
		if (!this.ossUploader) {
			throw new ASRError(
				"OSS configuration is required to use Buffer/Base64 with Paraformer/FunASR",
				"OSS_NOT_CONFIGURED"
			);
		}

		return this.ossUploader.upload(buffer);
	}

	private _fileUrlToPath(fileUrl: string): string {
		// file:///absolute/path -> /absolute/path
		// file://relative/path -> relative/path
		const url = new URL(fileUrl);
		return decodeURIComponent(url.pathname);
	}

	private _toBuffer(audio: string | Buffer): Buffer {
		if (Buffer.isBuffer(audio)) {
			return audio;
		}
		// Base64
		const base64Data = audio.startsWith("data:")
			? audio.split(",")[1]
			: audio;
		return Buffer.from(base64Data, "base64");
	}

	private async _writeTempFile(buffer: Buffer): Promise<string> {
		// 写入临时文件并返回绝对路径
		const tmpPath = path.join(os.tmpdir(), `asr-${Date.now()}.audio`);
		await fs.promises.writeFile(tmpPath, buffer);
		return tmpPath;
	}
}
```

### 3.3 Provider 适配层

每个 Provider 实现抽象基类 `ASRProvider`：

```typescript
abstract class ASRProvider {
	protected config: ASRProviderConfig;

	constructor(config: ASRProviderConfig) {
		this.config = config;
	}

	abstract transcribe(
		request: ASRTranscriptionRequest
	): Promise<ASRTranscriptionResponse>;

	protected abstract mapRequest(request: ASRTranscriptionRequest): unknown;
	protected abstract mapResponse(response: unknown): ASRTranscriptionResponse;
}
```

#### **ParaformerProvider / FunASRProvider (异步任务模式)**

```typescript
class ParaformerProvider extends ASRProvider {
	async transcribe(request: ASRTranscriptionRequest): Promise<ASRTranscriptionResponse> {
		// 1. 参数映射
		const apiRequest = this.mapRequest(request);

		// 2. 提交任务
		const task = await this.submitTask(apiRequest);

		// 3. 轮询任务状态
		const result = await this.pollTaskStatus(task.taskId, {
			interval: request.pollingInterval ?? 2000,
			maxAttempts: request.maxPollingAttempts ?? 300
		});

		// 4. 获取并解析结果
		const transcription = await this.fetchTranscriptionResult(result.transcriptionUrl);

		// 5. 响应映射
		return this.mapResponse(transcription);
	}

	private async pollTaskStatus(taskId: string, options: PollOptions): Promise<TaskResult> {
		for (let attempt = 0; attempt < options.maxAttempts; attempt++) {
			const status = await this.getTaskStatus(taskId);

			if (status.taskStatus === "SUCCEEDED") {
				return status;
			}

			if (status.taskStatus === "FAILED") {
				throw new ASRError(`Task failed: ${status.message}`);
			}

			await this.sleep(options.interval);
		}

		throw new ASRError("Polling timeout");
	}
}
```

#### **QwenASRProvider (同步调用模式)**

```typescript
class QwenASRProvider extends ASRProvider {
	async transcribe(request: ASRTranscriptionRequest): Promise<ASRTranscriptionResponse> {
		// 1. 参数映射
		const apiRequest = this.mapRequest(request);

		// 2. 直接调用
		const response = await this.call(apiRequest);

		// 3. 响应映射
		return this.mapResponse(response);
	}

	protected mapRequest(request: ASRTranscriptionRequest) {
		return {
			model: request.model ?? "qwen3-asr-flash",
			messages: [
				{
					role: "system",
					content: [{ text: request.context ?? "You are a helpful assistant." }]
				},
				{
					role: "user",
					content: [
						{ audio: request.audio }
					]
				}
			],
			parameters: {
				language: request.language,
				enable_inverse_text_normalization: request.enableITN
			}
		};
	}
}
```

### 3.2 错误处理

```typescript
class ASRError extends Error {
	constructor(
		message: string,
		public code: string,
		public provider?: string,
		public taskId?: string,
		public cause?: unknown
	) {
		super(message);
		this.name = "ASRError";
	}
}

// 统一错误码
enum ASRErrorCode {
	AUTHENTICATION_FAILED = "AUTHENTICATION_FAILED",
	INVALID_AUDIO_FORMAT = "INVALID_AUDIO_FORMAT",
	INVALID_AUDIO_INPUT = "INVALID_AUDIO_INPUT",    // 无效的音频输入类型
	INVALID_REQUEST = "INVALID_REQUEST",            // 请求参数验证失败
	AUDIO_TOO_LARGE = "AUDIO_TOO_LARGE",
	AUDIO_TOO_LONG = "AUDIO_TOO_LONG",
	FILE_NOT_FOUND = "FILE_NOT_FOUND",              // 本地文件不存在
	TASK_FAILED = "TASK_FAILED",
	POLLING_TIMEOUT = "POLLING_TIMEOUT",
	NETWORK_ERROR = "NETWORK_ERROR",
	OSS_NOT_CONFIGURED = "OSS_NOT_CONFIGURED",      // 使用 Buffer/Base64/本地文件 但未配置 OSS
	OSS_UPLOAD_FAILED = "OSS_UPLOAD_FAILED",        // OSS 上传失败
	UNKNOWN_ERROR = "UNKNOWN_ERROR"
}
```

### 3.4 配置管理

```typescript
// OSS 配置 Schema
const OSSConfigSchema = z.object({
	accessKeyId: z.string().min(1, "OSS Access Key ID 不能为空"),
	accessKeySecret: z.string().min(1, "OSS Access Key Secret 不能为空"),
	bucket: z.string().min(1, "OSS Bucket 名称不能为空"),
	region: z.string().min(1, "OSS Region 不能为空"),
	endpoint: z.string().url().optional(),
	secure: z.boolean().default(true),
	tempPrefix: z.string().default("ali-speech-sdk/temp/"),
	tempUrlExpiration: z.number().int().min(60).max(86400).default(3600),
	autoCleanup: z.boolean().default(true)
});

// 客户端配置 Schema
const ClientConfigSchema = z.object({
	apiKey: z.string().min(1, "API Key 不能为空"),
	region: z.enum(["cn-beijing", "ap-southeast-1"]).default("cn-beijing"),

	// 默认设置
	defaultProvider: ASRProviderSchema.optional(),
	timeout: z.number().int().min(1000).max(600000).optional(),

	// 异步任务轮询配置
	pollingInterval: z.number().int().min(100).max(60000).default(2000),
	maxPollingAttempts: z.number().int().min(1).max(1000).default(300),

	// OSS 配置
	oss: OSSConfigSchema.optional(),

	// HTTP 客户端配置
	httpClient: z.object({
		proxy: z.string().url().optional(),
		retryAttempts: z.number().int().min(0).max(10).default(3),
		retryDelay: z.number().int().min(100).max(10000).default(1000)
	}).optional()
});

// 导出类型
export type OSSConfig = z.infer<typeof OSSConfigSchema>;
export type ClientConfig = z.infer<typeof ClientConfigSchema>;

// AliSpeechClient 构造函数中验证配置
class AliSpeechClient {
	apiKey: string;
	region?: "cn-beijing" | "ap-southeast-1";

	// 默认设置
	defaultProvider?: "paraformer" | "funasr" | "qwen";
	timeout?: number;                        // 请求超时(ms)

	// 异步任务轮询配置
	pollingInterval?: number;                // 默认 2000ms
	maxPollingAttempts?: number;             // 默认 300次 (10分钟)

	// OSS 配置 (可选，用于自动上传 Base64/Buffer)
	oss?: {
		accessKeyId: string;
		accessKeySecret: string;
		bucket: string;
		region: string;
		endpoint?: string;                   // 自定义端点
		secure?: boolean;                    // 是否使用 HTTPS，默认 true
		// 临时文件配置
		tempPrefix?: string;                 // 临时文件路径前缀，默认 "ali-speech-sdk/temp/"
		tempUrlExpiration?: number;          // 临时 URL 过期时间(秒)，默认 3600 (1小时)
		autoCleanup?: boolean;               // 识别完成后自动删除临时文件，默认 true
	};

	// HTTP 客户端配置
	httpClient?: {
		proxy?: string;
		retryAttempts?: number;
		retryDelay?: number;
	};
}
```

---

## 4. 推荐的实现优先级

### Phase 1: 核心基础设施
1. ✅ 实现类型定义 (`src/types/index.ts`)
2. ✅ 实现认证模块 (`src/auth/index.ts`)
3. ✅ 实现错误处理 (`src/error/index.ts`)
4. ✅ 实现 `ASRProvider` 抽象基类
5. ✅ 实现 `AliSpeechClient` 统一客户端入口

### Phase 2: ASR Provider 实现 (仅 URL 支持)
6. ✅ 实现 `ParaformerProvider` (异步任务模式参考)
7. ✅ 实现 `FunASRProvider` (复用异步任务逻辑)
8. ✅ 实现 `QwenASRProvider` (同步调用模式)

### Phase 3: ASRService + 音频输入处理
9. ✅ 实现 `ASRService` 服务层
10. ✅ 实现音频输入类型检测 (URL/Buffer/Base64)
11. ✅ 实现 `OSSUploader` 模块 (支持 Buffer/Base64 上传)
12. ✅ 实现 Qwen 本地临时文件写入
13. ✅ 添加 OSS_NOT_CONFIGURED 错误处理
14. ✅ 实现批量识别功能

### Phase 4: 高级功能和优化
15. ✅ 添加重试和错误恢复机制
16. ⏳ 实现热词管理 API
17. ⏳ 添加进度回调和事件系统
18. ⏳ 实现流式识别 (如果 API 支持)

### Phase 5: TTS 和其他服务 (未来)
19. ⏳ 实现 `TTSService`
20. ⏳ 添加更多语音服务

---

## 5. 技术决策记录

### 5.1 为什么使用统一客户端而非独立的 ASR 客户端？

**决策**: 对外暴露统一的 `AliSpeechClient`，ASR 作为 `client.asr` 子模块

**理由**:
1. **可扩展性**: 未来可以无缝添加 TTS、实时识别等服务
   ```typescript
   client.asr.transcribe()
   client.tts.synthesize()
   client.realtime.connect()
   ```
2. **配置共享**: API Key、OSS、Region 等配置在所有服务间共享，避免重复配置
3. **一致的用户体验**: 所有阿里云语音服务使用统一的初始化方式
4. **更清晰的命名空间**: 避免函数名冲突，`client.asr.transcribe()` 比 `client.transcribe()` 更明确

**权衡**:
- ✅ 更好的可维护性和扩展性
- ✅ 配置集中管理
- ❌ API 调用路径稍长（多一层 `.asr`）
- ✅ 但更符合大型 SDK 的设计模式（参考 AWS SDK、阿里云 SDK）

### 5.2 严格的类型系统设计

**决策**: 使用 **Zod v4** 进行运行时验证 + TypeScript 编译时类型检查

**实现**:

1. **Zod Schema 作为单一真实来源**
   ```typescript
   // 定义 Schema
   export const ASRLanguageSchema = z.enum(["zh", "en", "ja", "ko", ...]);

   // 从 Schema 推断类型
   export type ASRLanguage = z.infer<typeof ASRLanguageSchema>;
   ```

2. **模型类型**: 每个 Provider 都有专属的 Zod Schema
   ```typescript
   export const ParaformerModelSchema = z.enum(["paraformer-v2", ...]);
   export const QwenASRModelSchema = z.enum(["qwen3-asr-flash", ...]);
   ```

3. **Provider 特定 Schema**: 使用 `discriminatedUnion` 和 `omit()`
   ```typescript
   export const ParaformerRequestSchema = BaseSchema.extend({
     provider: z.literal("paraformer"),
     enableDiarization: z.boolean().optional()
   });

   export const QwenRequestSchema = BaseSchema.extend({
     provider: z.literal("qwen")
   }).omit({ enableDiarization: true });  // Qwen 不支持

   export const RequestUnionSchema = z.discriminatedUnion("provider", [
     ParaformerRequestSchema,
     QwenRequestSchema
   ]);
   ```

4. **运行时验证**: 在关键入口点进行验证
   ```typescript
   async transcribe(request: ASRTranscriptionRequest) {
     // 运行时验证
     const result = ASRTranscriptionRequestUnionSchema.safeParse(request);
     if (!result.success) {
       throw new ASRError("Invalid request", "INVALID_REQUEST", undefined, undefined, result.error);
     }
     const validated = result.data;
     // 使用验证后的数据
   }
   ```

5. **配置验证**: 客户端初始化时验证配置
   ```typescript
   constructor(config: ClientConfig) {
     const result = ClientConfigSchema.safeParse(config);
     if (!result.success) {
       throw new Error(`Invalid config: ${result.error.message}`);
     }
     this.config = result.data;  // 使用验证和规范化后的配置
   }
   ```

**优势**:
- ✅ **编译时 + 运行时双重保障**: TypeScript 提供编译时检查，Zod 提供运行时验证
- ✅ **单一真实来源**: Schema 即类型，避免类型定义重复
- ✅ **自动类型推断**: `z.infer<>` 自动推断 TypeScript 类型
- ✅ **数据转换**: Zod 支持自动转换和默认值（如 `samplingRate: z.enum(["8000", "16000"]).transform(Number)`）
- ✅ **详细错误消息**: Zod 提供清晰的验证错误信息
- ✅ **IDE 智能提示**: TypeScript 类型提供完整的自动补全

**验证场景**:
- ✅ 用户配置（API Key、OSS 配置）
- ✅ API 请求参数
- ✅ 文件路径和 URL
- ❌ 内部函数调用（性能优化）
- ❌ 已验证的数据（避免重复验证）

**示例**:
```typescript
// ✅ 编译时错误 + 运行时验证
client.asr.transcribe({
  provider: "qwen",
  enableDiarization: true,     // TypeScript 编译错误
  language: "invalid-lang"     // Zod 运行时错误
});

// ✅ 配置验证和默认值
const client = new AliSpeechClient({
  apiKey: "xxx"
  // region 自动默认为 "cn-beijing"
  // pollingInterval 自动默认为 2000
});
```

### 5.3 为什么选择 Promise-based API？
- ✅ 符合现代 JavaScript/TypeScript 异步编程习惯
- ✅ 支持 async/await 语法，代码更简洁
- ✅ 统一异步和同步 API 的调用方式

### 5.4 为什么不使用 EventEmitter？
- ❌ 对于一次性任务，Promise 更简单
- ⏳ 但可以在 Phase 4 添加 EventEmitter 支持进度回调

### 5.5 如何处理异步任务轮询？
- 封装在 Provider 内部，对外暴露 Promise
- 提供可配置的轮询间隔和超时时间
- 支持取消轮询 (通过 AbortSignal)

### 5.6 Buffer/Base64 音频处理策略

**问题**: 三个 API 都不支持 Base64/Buffer 直传

**决策**: 提供可选的 OSS 自动上传功能
- ✅ **配置 OSS**: 用户提供 OSS 配置，SDK 自动上传 Buffer/Base64 并返回临时 URL
- ✅ **未配置 OSS + Qwen**: 写入临时本地文件，利用 Qwen 支持本地路径的特性
- ✅ **未配置 OSS + Paraformer/FunASR**: 抛出 `OSS_NOT_CONFIGURED` 运行时错误

**理由**:
1. **职责清晰**: SDK 不提供文件托管服务，由用户自行配置存储
2. **成本透明**: OSS 费用由用户承担，SDK 无运营成本
3. **灵活性**: 用户可选择不同的 OSS bucket 和地域
4. **安全性**: 临时 URL 自动过期，可配置自动清理

**权衡**:
- ❌ 增加了用户配置复杂度 (需要配置 OSS)
- ✅ 但避免了 SDK 维护文件服务的成本和责任
- ✅ 为需要 Buffer/Base64 的场景提供了可行方案

---

## 6. 附录: API 完整对照表

### 6.1 参数映射表

| 统一接口参数 | Paraformer | FunASR | Qwen-ASR |
|-------------|-----------|---------|----------|
| audio | file_urls | file_urls | messages[].content[].audio |
| model | model | model | model |
| language | language_hints | ❌ | parameters.language |
| customVocabulary | ❌ (需先创建) | ❌ (需先创建) | messages[].content[].text (context) |
| vocabularyId | vocabulary_id | vocabulary_id | ❌ |
| context | ❌ | ❌ | messages[0].content[].text |
| enableDiarization | diarization_enabled | diarization_enabled | ❌ |
| enableTimestamp | timestamp_alignment_enabled | ✅ (默认) | ❌ |
| enableITN | ❌ | ❌ | parameters.enable_inverse_text_normalization |
| enableEmotionDetection | ❌ | ❌ | ✅ (Qwen3-ASR默认) |
| removeDisfluency | disfluency_removal_enabled | ❌ | ❌ |
| channelId | ❌ | channel_id | ❌ |

### 6.2 响应映射表

| 统一接口字段 | Paraformer | FunASR | Qwen-ASR |
|------------|-----------|---------|----------|
| text | transcription_url → text | transcription_url → text | choices[0].message.content[0].text |
| sentences | transcription_url → sentences | transcription_url → sentences | ❌ |
| words | transcription_url → words | transcription_url → words | ❌ |
| speakers | transcription_url → speaker_id | transcription_url → speaker_id | ❌ |
| emotion | ❌ | ❌ | choices[0].message.content[0].annotations.emotion |
| metadata.duration | ❌ | ❌ | usage.audio_tokens / sampling_rate |
| metadata.language | ❌ | ❌ | choices[0].message.content[0].annotations.language |

---

## 7. 总结

本设计文档提出了一个基于统一客户端架构的语音 SDK，通过以下方式解决三个 ASR API 的差异：

### 架构特点：

1. **统一客户端入口**: `AliSpeechClient` 作为 SDK 唯一入口，ASR 作为子模块 `client.asr`
2. **服务分层设计**: `ASRService` 负责业务逻辑和音频处理，`ASRProvider` 负责 API 适配
3. **Provider 模式**: 使用抽象基类隐藏不同 API 的实现细节
4. **参数归一化**: 统一的请求/响应接口，自动映射到各 Provider 原生参数
5. **异步处理统一**: 将异步任务模式封装为 Promise，与同步 API 保持一致

### 核心优势：

1. **简单易用**: 一次初始化，所有服务可用
   ```typescript
   const client = new AliSpeechClient({ apiKey: "xxx" });
   await client.asr.transcribe({ ... });
   ```

2. **智能音频处理**: 自动检测多种音频输入格式，按需处理
   - HTTP/HTTPS URL → 直接传递给 Provider
   - `file://` 协议 + Qwen → 转换为本地路径
   - `file://` 协议 + Paraformer/FunASR → 读取并上传 OSS
   - 本地文件路径 + Qwen → 转换为绝对路径
   - 本地文件路径 + Paraformer/FunASR → 读取并上传 OSS
   - Buffer + Qwen → 写入临时本地文件
   - Buffer + Paraformer/FunASR → 自动上传 OSS
   - 未配置 OSS → 清晰的错误提示

3. **严格的类型安全（Zod v4 + TypeScript）**:
   - **编译时检查**: TypeScript 字面量类型和函数重载
   - **运行时验证**: Zod schemas 验证所有外部输入
   - **单一真实来源**: 从 Zod schema 推断 TypeScript 类型（`z.infer<>`）
   - **自动转换**: Zod 支持数据转换和默认值
   - **清晰错误**: Zod 提供详细的验证错误信息
   - **参数约束**: 使用 `discriminatedUnion` 为不同 Provider 提供专属验证

4. **可扩展性**: 易于添加 TTS、实时识别等新服务，配置共享

5. **灵活性**: 既保留各服务特色功能，又提供统一调用体验

这种设计为阿里云语音服务提供了一个现代化、易用、可扩展的 TypeScript/JavaScript SDK。
