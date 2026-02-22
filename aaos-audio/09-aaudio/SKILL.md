---
name: "aaudio"
description: "Android AAudio 高性能音频 API，用于低延迟音频播放与录制，支持 mmap 模式、回调模式与阻塞模式"
version: "1.0.0"
triggers: ["AAudio", "AAudioStream", "AAudioStreamBuilder", "mmap", "low latency", "AAUDIO_PERFORMANCE_MODE_LOW_LATENCY", "AAUDIO_SHARING_MODE_EXCLUSIVE", "aaudio_data_callback", "AAudioStream_write", "AAudioStream_read", "Oboe"]
---

> 参考来源：Android AOSP frameworks/av/media/libaaudio, developer.android.com/ndk/guides/audio/aaudio

# 🎵 Android AAudio 高性能音频 API

---

## 1. AAudio 概述

### 1.1 简介

```
AAudio 是 Android 8.0 (Oreo) 引入的高性能音频 C API
- 目标: 低延迟、高性能音频应用
- 特点: 简洁的流式 API，支持 mmap 零拷贝模式
- 适用: 实时音频处理、游戏音效、音乐合成、VoIP
- 版本: Android 8.0+ (API 26+)
```

### 1.2 与其他 API 对比

```
┌─────────────────────────────────────────────────────────────────┐
│               Android Audio API Comparison                      │
├───────────────┬────────────┬──────────┬──────────┬─────────────┤
│  API          │ 延迟       │ 复杂度   │ 功能     │ 适用场景    │
├───────────────┼────────────┼──────────┼──────────┼─────────────┤
│ AAudio mmap   │ ~2ms       │ 中       │ 基础     │ 实时音频    │
│ AAudio shared │ ~10ms      │ 低       │ 基础     │ 普通应用    │
│ OpenSL ES     │ ~10ms      │ 高       │ 丰富     │ 多媒体      │
│ AudioTrack    │ ~20ms      │ 低       │ 基础     │ 普通播放    │
│ Fast Track    │ ~4ms       │ 中       │ 基础     │ 系统音效    │
└───────────────┴────────────┴──────────┴──────────┴─────────────┘
```

### 1.3 源码路径

```
# AAudio 库
frameworks/av/media/libaaudio/

# AAudio 服务
frameworks/av/services/oboeservice/

# 头文件
frameworks/av/media/libaaudio/include/aaudio/

# 示例代码
frameworks/av/media/libaaudio/examples/
```

### 1.4 库架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     AAudio 库架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  libaaudio.so (公共 API 库)                                 ││
│  │  - 提供给应用使用的公共 API                                  ││
│  │  - AAudioStreamBuilder, AAudioStream 等接口                 ││
│  │  - 轻量级，只做 API 封装                                     ││
│  │  - 路径: frameworks/av/media/libaaudio/src/core/            ││
│  └─────────────────────────────────────────────────────────────┘│
│                          ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  libaaudio_internal.so (内部实现库)                         ││
│  │  - 内部实现细节                                              ││
│  │  - AudioStreamInternal, AudioStreamTrack, AudioStreamRecord ││
│  │  - 与 AudioFlinger 的 Binder 通信                           ││
│  │  - mmap 模式实现                                            ││
│  │  - 路径: frameworks/av/media/libaaudio/src/client/          ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.5 调用路径

```
┌─────────────────────────────────────────────────────────────────┐
│                     AAudio 调用路径                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  应用调用 AAudio API                                            │
│      │                                                          │
│      ↓                                                          │
│  ┌─────────────────┐                                            │
│  │ libaaudio.so    │  公共 API                                  │
│  │ AAudioStream_   │  - AAudioStream_write()                    │
│  │ write()         │  - AAudioStream_read()                     │
│  └────────┬────────┘                                            │
│           ↓                                                      │
│  ┌─────────────────┐                                            │
│  │libaaudio_       │  内部实现                                  │
│  │internal.so      │  - AudioStreamInternal (mmap 模式)        │
│  │                 │  - AudioStreamTrack (共享模式播放)         │
│  │                 │  - AudioStreamRecord (共享模式录音)        │
│  └────────┬────────┘                                            │
│           │                                                      │
│     ┌─────┴─────┐                                               │
│     ↓           ↓                                               │
│  ┌──────────┐ ┌──────────────┐                                  │
│  │ mmap 模式│ │ 共享模式     │                                  │
│  │ (独占)   │ │ (通过Binder) │                                  │
│  └────┬─────┘ └──────┬───────┘                                  │
│       │              │                                          │
│       ↓              ↓                                          │
│  ┌──────────┐ ┌──────────────┐                                  │
│  │ AAudio   │ │ AudioFlinger │                                  │
│  │ Service  │ │(IAudioFlinger)│                                 │
│  │ (mmap)   │ │              │                                  │
│  └────┬─────┘ └──────┬───────┘                                  │
│       │              │                                          │
│       └──────┬───────┘                                          │
│              ↓                                                  │
│  ┌──────────────────┐                                           │
│  │    Audio HAL     │                                           │
│  └──────────────────┘                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.6 关键类对应关系

```
libaaudio.so (公共 API):
├── AAudioStreamBuilder          # 流构建器
├── AAudioStream                 # 流对象 (不透明句柄)
└── AAudio API 函数              # C API 入口

libaaudio_internal.so (内部实现):
├── AudioStream                  # 流基类
├── AudioStreamBuilder           # 构建器实现
├── AudioStreamInternal          # mmap 模式实现
│   ├── AudioStreamInternalPlay  # mmap 播放
│   └── AudioStreamInternalCapture # mmap 录音
├── AudioStreamTrack             # 共享模式播放 (封装 AudioTrack)
├── AudioStreamRecord            # 共享模式录音 (封装 AudioRecord)
└── AudioEndpoint                # mmap 端点
```

---

## 2. 核心概念

### 2.1 音频流 (AAudioStream)

```
┌─────────────────────────────────────────────────────────────────┐
│                     AAudioStream                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  AAudioStream 是 AAudio 的核心概念                              │
│  - 代表一个单向的音频数据流 (输入或输出)                        │
│  - 连接到一个音频设备                                           │
│  - 通过读写操作传输音频数据                                     │
│                                                                 │
│  流方向:                                                        │
│  - AAUDIO_DIRECTION_OUTPUT: 播放                                │
│  - AAUDIO_DIRECTION_INPUT: 录音                                 │
│                                                                 │
│  数据传输方式:                                                  │
│  - 阻塞读写: AAudioStream_write / AAudioStream_read            │
│  - 回调模式: AAudioStreamBuilder_setDataCallback               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 共享模式

```c
// 共享模式
typedef enum {
    // 共享模式: 多个应用可以同时使用设备
    AAUDIO_SHARING_MODE_SHARED,
    
    // 独占模式: 独占设备，延迟最低
    AAUDIO_SHARING_MODE_EXCLUSIVE,
} aaudio_sharing_mode_t;

// 独占模式特点:
// - 延迟最低 (可达 2ms 以内)
// - 可能获取失败 (设备被占用)
// - 设备断开时流会被关闭
// - 使用完毕应尽快释放
```

### 2.3 性能模式

```c
// 性能模式
typedef enum {
    // 无特殊性能要求
    AAUDIO_PERFORMANCE_MODE_NONE,
    
    // 低延迟模式: 优先降低延迟
    AAUDIO_PERFORMANCE_MODE_LOW_LATENCY,
    
    // 省电模式: 优先降低功耗
    AAUDIO_PERFORMANCE_MODE_POWER_SAVING,
} aaudio_performance_mode_t;

// 低延迟模式特点:
// - 使用更小的缓冲区
// - 可能使用 mmap 模式
// - CPU 占用较高
// - 适合实时音频处理
```

### 2.4 流状态

```c
typedef enum {
    AAUDIO_STREAM_STATE_UNINITIALIZED = 0,  // 未初始化
    AAUDIO_STREAM_STATE_UNKNOWN,             // 未知
    AAUDIO_STREAM_STATE_OPEN,                // 已打开
    AAUDIO_STREAM_STATE_STARTING,            // 正在启动
    AAUDIO_STREAM_STATE_STARTED,             // 已启动
    AAUDIO_STREAM_STATE_PAUSING,             // 正在暂停
    AAUDIO_STREAM_STATE_PAUSED,              // 已暂停
    AAUDIO_STREAM_STATE_FLUSHING,            // 正在刷新
    AAUDIO_STREAM_STATE_FLUSHED,             // 已刷新
    AAUDIO_STREAM_STATE_STOPPING,            // 正在停止
    AAUDIO_STREAM_STATE_STOPPED,             // 已停止
    AAUDIO_STREAM_STATE_CLOSING,             // 正在关闭
    AAUDIO_STREAM_STATE_CLOSED,              // 已关闭
    AAUDIO_STREAM_STATE_DISCONNECTED,        // 已断开
} aaudio_stream_state_t;

// 状态转换:
// OPEN -> STARTING -> STARTED -> STOPPING -> STOPPED -> CLOSING -> CLOSED
//                    ↓
//                  PAUSING -> PAUSED
//                    ↓
//                  FLUSHING -> FLUSHED
```

---

## 3. API 详解

### 3.1 创建流构建器

```c
#include <aaudio/AAudio.h>

// 创建流构建器
AAudioStreamBuilder* builder;
aaudio_result_t result = AAudio_createStreamBuilder(&builder);

if (result != AAUDIO_OK) {
    ALOGE("Failed to create stream builder: %d", result);
    return;
}
```

### 3.2 配置流参数

```c
// 设置流方向
AAudioStreamBuilder_setDirection(builder, AAUDIO_DIRECTION_OUTPUT);

// 设置设备 ID (可选，默认自动选择)
AAudioStreamBuilder_setDeviceId(builder, AAUDIO_UNSPECIFIED);

// 设置采样率
AAudioStreamBuilder_setSampleRate(builder, 48000);

// 设置声道数
AAudioStreamBuilder_setChannelCount(builder, 2);

// 设置格式
AAudioStreamBuilder_setFormat(builder, AAUDIO_FORMAT_PCM_I16);
// 可选格式:
// AAUDIO_FORMAT_INVALID
// AAUDIO_FORMAT_PCM_I16     - 16位整数
// AAUDIO_FORMAT_PCM_FLOAT   - 32位浮点
// AAUDIO_FORMAT_PCM_I24     - 24位整数 (Android 11+)
// AAUDIO_FORMAT_PCM_I32     - 32位整数 (Android 12+)

// 设置共享模式
AAudioStreamBuilder_setSharingMode(builder, AAUDIO_SHARING_MODE_EXCLUSIVE);

// 设置性能模式
AAudioStreamBuilder_setPerformanceMode(builder, AAUDIO_PERFORMANCE_MODE_LOW_LATENCY);

// 设置缓冲区容量
AAudioStreamBuilder_setBufferCapacityInFrames(builder, 0);  // 0 = 自动

// 设置用途 (Android 11+)
AAudioStreamBuilder_setUsage(builder, AAUDIO_USAGE_MEDIA);
// 可选用途:
// AAUDIO_USAGE_MEDIA           - 媒体播放
// AAUDIO_USAGE_VOICE_COMMUNICATION - 通话
// AAUDIO_USAGE_VOICE_COMMUNICATION_SIGNALLING - 通话信令
// AAUDIO_USAGE_ALARM           - 闹钟
// AAUDIO_USAGE_NOTIFICATION    - 通知
// AAUDIO_USAGE_NOTIFICATION_RINGTONE - 铃声
// AAUDIO_USAGE_NOTIFICATION_EVENT - 事件
// AAUDIO_USAGE_ASSISTANCE_ACCESSIBILITY - 辅助功能
// AAUDIO_USAGE_ASSISTANCE_NAVIGATION_GUIDANCE - 导航
// AAUDIO_USAGE_ASSISTANCE_SONIFICATION - 音效
// AAUDIO_USAGE_GAME            - 游戏
// AAUDIO_USAGE_ASSISTANT       - 助手

// 设置内容类型 (Android 11+)
AAudioStreamBuilder_setContentType(builder, AAUDIO_CONTENT_TYPE_MUSIC);
// 可选类型:
// AAUDIO_CONTENT_TYPE_MUSIC      - 音乐
// AAUDIO_CONTENT_TYPE_SPEECH     - 语音
// AAUDIO_CONTENT_TYPE_MOVIE      - 电影
// AAUDIO_CONTENT_TYPE_SONIFICATION - 音效

// 设置输入预设 (录音用，Android 11+)
AAudioStreamBuilder_setInputPreset(builder, AAUDIO_INPUT_PRESET_VOICE_RECOGNITION);
// 可选预设:
// AAUDIO_INPUT_PRESET_GENERIC         - 通用
// AAUDIO_INPUT_PRESET_CAMCORDER       - 摄像
// AAUDIO_INPUT_PRESET_VOICE_RECOGNITION - 语音识别
// AAUDIO_INPUT_PRESET_VOICE_COMMUNICATION - 通话
// AAUDIO_INPUT_PRESET_UNPROCESSED     - 原始
// AAUDIO_INPUT_PRESET_VOICE_PERFORMANCE - 表演 (Android 12+)
```

### 3.3 打开流

```c
AAudioStream* stream;
aaudio_result_t result = AAudioStreamBuilder_openStream(builder, &stream);

if (result != AAUDIO_OK) {
    ALOGE("Failed to open stream: %d", result);
    AAudioStreamBuilder_delete(builder);
    return;
}

// 检查实际配置的参数
int32_t sampleRate = AAudioStream_getSampleRate(stream);
int32_t channelCount = AAudioStream_getChannelCount(stream);
aaudio_format_t format = AAudioStream_getFormat(stream);
int32_t framesPerBurst = AAudioStream_getFramesPerBurst(stream);
int32_t bufferSize = AAudioStream_getBufferSizeInFrames(stream);
int32_t bufferCapacity = AAudioStream_getBufferCapacityInFrames(stream);

ALOGD("Stream opened: rate=%d, channels=%d, format=%d, burst=%d",
      sampleRate, channelCount, format, framesPerBurst);

// 检查是否使用 mmap 模式
bool isMMap = AAudioStream_getSharingMode(stream) == AAUDIO_SHARING_MODE_EXCLUSIVE;
ALOGD("mmap mode: %s", isMMap ? "enabled" : "disabled");

// 释放构建器
AAudioStreamBuilder_delete(builder);
```

### 3.4 阻塞模式读写

```c
// 启动流
aaudio_result_t result = AAudioStream_requestStart(stream);
if (result != AAUDIO_OK) {
    ALOGE("Failed to start stream: %d", result);
    return;
}

// 等待流启动
aaudio_stream_state_t state = AAUDIO_STREAM_STATE_STARTING;
result = AAudioStream_waitForStateChange(stream, AAUDIO_STREAM_STATE_STARTING, &state, 1000000000);

// 写入数据 (播放)
int16_t buffer[1024 * 2];  // 1024 帧，立体声
int64_t timeoutNanos = 1000000000;  // 1 秒超时

while (playing) {
    // 生成音频数据
    generateAudioData(buffer, 1024);
    
    // 写入流
    int64_t framesWritten = AAudioStream_write(stream, buffer, 1024, timeoutNanos);
    
    if (framesWritten < 0) {
        ALOGE("Write error: %d", framesWritten);
        break;
    } else if (framesWritten < 1024) {
        ALOGW("Partial write: %lld / %d", framesWritten, 1024);
    }
}

// 读取数据 (录音)
int64_t framesRead = AAudioStream_read(stream, buffer, 1024, timeoutNanos);
if (framesRead < 0) {
    ALOGE("Read error: %d", framesRead);
}
```

### 3.5 回调模式

```c
// 数据回调函数
aaudio_data_callback_result_t dataCallback(
        AAudioStream* stream,
        void* userData,
        void* audioData,
        int32_t numFrames) {
    
    // 生成音频数据 (播放)
    int16_t* outputData = (int16_t*)audioData;
    generateAudioData(outputData, numFrames);
    
    // 或处理音频数据 (录音)
    // int16_t* inputData = (int16_t*)audioData;
    // processAudioData(inputData, numFrames);
    
    return AAUDIO_CALLBACK_RESULT_CONTINUE;
}

// 错误回调函数
void errorCallback(
        AAudioStream* stream,
        void* userData,
        aaudio_result_t error) {
    
    ALOGE("Stream error: %d", error);
    
    if (error == AAUDIO_ERROR_DISCONNECTED) {
        // 设备断开，需要重新创建流
        // 通常在另一个线程中处理
    }
}

// 设置回调
AAudioStreamBuilder_setDataCallback(builder, dataCallback, userData);
AAudioStreamBuilder_setFramesPerDataCallback(builder, 0);  // 0 = 自动
AAudioStreamBuilder_setErrorCallback(builder, errorCallback, userData);

// 回调返回值:
// AAUDIO_CALLBACK_RESULT_CONTINUE - 继续流
// AAUDIO_CALLBACK_RESULT_STOP     - 停止流
```

### 3.6 流控制

```c
// 启动流
aaudio_result_t result = AAudioStream_requestStart(stream);

// 暂停流
result = AAudioStream_requestPause(stream);

// 刷新流 (丢弃缓冲区数据)
result = AAudioStream_requestFlush(stream);

// 停止流
result = AAudioStream_requestStop(stream);

// 等待状态变化
aaudio_stream_state_t nextState;
result = AAudioStream_waitForStateChange(
    stream, 
    AAUDIO_STREAM_STATE_STOPPING,  // 当前预期状态
    &nextState,                     // 实际状态
    1000000000                      // 超时 (纳秒)
);

// 关闭流
result = AAudioStream_close(stream);
```

### 3.7 缓冲区管理

```c
// 获取缓冲区信息
int32_t bufferSize = AAudioStream_getBufferSizeInFrames(stream);
int32_t bufferCapacity = AAudioStream_getBufferCapacityInFrames(stream);
int32_t framesPerBurst = AAudioStream_getFramesPerBurst(stream);
int32_t framesAvailable = AAudioStream_getFramesAvailable(stream, NULL);
int32_t xRunCount = AAudioStream_getXRunCount(stream);

// 设置缓冲区大小 (建议为 framesPerBurst 的整数倍)
int32_t newBufferSize = framesPerBurst * 2;
result = AAudioStream_setBufferSizeInFrames(stream, newBufferSize);

// 获取延迟
int64_t presentationTime;
int64_t framesWritten = AAudioStream_getFramesWritten(stream);
int64_t framesRead = AAudioStream_getFramesRead(stream);
// 延迟 ≈ framesWritten - framesRead (帧数)
```

---

## 4. mmap 模式

### 4.1 mmap 原理

```
┌─────────────────────────────────────────────────────────────────┐
│                     AAudio mmap Mode                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  传统模式 (IRQ-based)                                           │
│  ┌─────────┐    ┌─────────────┐    ┌─────────┐    ┌─────────┐  │
│  │   App   │ →  │AudioFlinger │ → │   HAL   │ → │  Driver │  │
│  │         │    │  (混音)     │    │         │    │  (IRQ)  │  │
│  └─────────┘    └─────────────┘    └─────────┘    └─────────┘  │
│       ↑              ↑                ↑              ↑          │
│       └──────────────┴────────────────┴──────────────┘          │
│                    多次数据拷贝                                 │
│                                                                 │
│  mmap 模式 (NOIRQ)                                              │
│  ┌─────────┐                                                    │
│  │   App   │ ─────────────────────────────→ ┌─────────┐        │
│  │         │         共享内存                │  Driver │        │
│  └─────────┘                                  └─────────┘        │
│       ↑                                            ↑             │
│       └────────────────────────────────────────────┘             │
│                    零拷贝，直接访问                              │
│                                                                 │
│  优势:                                                          │
│  - 延迟极低 (2ms 以内)                                          │
│  - 无数据拷贝                                                   │
│  - 无中断开销                                                   │
│                                                                 │
│  条件:                                                          │
│  - 独占模式 (AAUDIO_SHARING_MODE_EXCLUSIVE)                     │
│  - 硬件支持 mmap                                                │
│  - HAL 支持 mmap                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 启用 mmap

```c
// 设置独占模式以启用 mmap
AAudioStreamBuilder_setSharingMode(builder, AAUDIO_SHARING_MODE_EXCLUSIVE);
AAudioStreamBuilder_setPerformanceMode(builder, AAUDIO_PERFORMANCE_MODE_LOW_LATENCY);

// 打开流后检查是否成功
AAudioStream* stream;
result = AAudioStreamBuilder_openStream(builder, &stream);

if (result == AAUDIO_OK) {
    bool isExclusive = AAudioStream_getSharingMode(stream) == AAUDIO_SHARING_MODE_EXCLUSIVE;
    ALOGD("Exclusive mode: %s", isExclusive ? "yes" : "no");
    
    if (isExclusive) {
        // mmap 模式已启用
        int32_t framesPerBurst = AAudioStream_getFramesPerBurst(stream);
        ALOGD("mmap burst size: %d frames", framesPerBurst);
    }
} else if (result == AAUDIO_ERROR_INVALID_FORMAT) {
    // 设备不支持请求的格式，尝试共享模式
    AAudioStreamBuilder_setSharingMode(builder, AAUDIO_SHARING_MODE_SHARED);
    result = AAudioStreamBuilder_openStream(builder, &stream);
}
```

---

## 5. 完整示例

### 5.1 播放示例 (回调模式)

```c
#include <aaudio/AAudio.h>
#include <math.h>

#define SAMPLE_RATE 48000
#define CHANNEL_COUNT 2
#define FREQUENCY 440.0  // A4 音符

typedef struct {
    double phase;
    double phaseIncrement;
} SineWaveData;

aaudio_data_callback_result_t sineWaveCallback(
        AAudioStream* stream,
        void* userData,
        void* audioData,
        int32_t numFrames) {
    
    SineWaveData* data = (SineWaveData*)userData;
    float* output = (float*)audioData;
    
    for (int i = 0; i < numFrames; i++) {
        float sample = (float)sin(data->phase) * 0.5f;
        
        // 立体声输出
        *output++ = sample;  // 左声道
        *output++ = sample;  // 右声道
        
        data->phase += data->phaseIncrement;
        if (data->phase >= 2.0 * M_PI) {
            data->phase -= 2.0 * M_PI;
        }
    }
    
    return AAUDIO_CALLBACK_RESULT_CONTINUE;
}

void errorCallback(
        AAudioStream* stream,
        void* userData,
        aaudio_result_t error) {
    ALOGE("Stream error: %d", error);
}

int main() {
    AAudioStreamBuilder* builder;
    AAudioStream* stream;
    aaudio_result_t result;
    
    // 创建构建器
    result = AAudio_createStreamBuilder(&builder);
    if (result != AAUDIO_OK) return -1;
    
    // 配置流
    AAudioStreamBuilder_setDirection(builder, AAUDIO_DIRECTION_OUTPUT);
    AAudioStreamBuilder_setSampleRate(builder, SAMPLE_RATE);
    AAudioStreamBuilder_setChannelCount(builder, CHANNEL_COUNT);
    AAudioStreamBuilder_setFormat(builder, AAUDIO_FORMAT_PCM_FLOAT);
    AAudioStreamBuilder_setSharingMode(builder, AAUDIO_SHARING_MODE_EXCLUSIVE);
    AAudioStreamBuilder_setPerformanceMode(builder, AAUDIO_PERFORMANCE_MODE_LOW_LATENCY);
    AAudioStreamBuilder_setUsage(builder, AAUDIO_USAGE_MEDIA);
    AAudioStreamBuilder_setContentType(builder, AAUDIO_CONTENT_TYPE_MUSIC);
    
    // 设置回调
    SineWaveData waveData = {
        .phase = 0.0,
        .phaseIncrement = 2.0 * M_PI * FREQUENCY / SAMPLE_RATE
    };
    AAudioStreamBuilder_setDataCallback(builder, sineWaveCallback, &waveData);
    AAudioStreamBuilder_setErrorCallback(builder, errorCallback, NULL);
    
    // 打开流
    result = AAudioStreamBuilder_openStream(builder, &stream);
    AAudioStreamBuilder_delete(builder);
    
    if (result != AAUDIO_OK) {
        ALOGE("Failed to open stream: %d", result);
        return -1;
    }
    
    // 启动流
    result = AAudioStream_requestStart(stream);
    if (result != AAUDIO_OK) {
        ALOGE("Failed to start stream: %d", result);
        AAudioStream_close(stream);
        return -1;
    }
    
    // 播放 5 秒
    sleep(5);
    
    // 停止并关闭
    AAudioStream_requestStop(stream);
    AAudioStream_close(stream);
    
    return 0;
}
```

### 5.2 录音示例 (阻塞模式)

```c
#include <aaudio/AAudio.h>
#include <stdio.h>

#define SAMPLE_RATE 48000
#define CHANNEL_COUNT 1
#define DURATION_SECONDS 5

int main() {
    AAudioStreamBuilder* builder;
    AAudioStream* stream;
    aaudio_result_t result;
    
    // 创建构建器
    result = AAudio_createStreamBuilder(&builder);
    if (result != AAUDIO_OK) return -1;
    
    // 配置录音流
    AAudioStreamBuilder_setDirection(builder, AAUDIO_DIRECTION_INPUT);
    AAudioStreamBuilder_setSampleRate(builder, SAMPLE_RATE);
    AAudioStreamBuilder_setChannelCount(builder, CHANNEL_COUNT);
    AAudioStreamBuilder_setFormat(builder, AAUDIO_FORMAT_PCM_I16);
    AAudioStreamBuilder_setInputPreset(builder, AAUDIO_INPUT_PRESET_VOICE_RECOGNITION);
    
    // 打开流
    result = AAudioStreamBuilder_openStream(builder, &stream);
    AAudioStreamBuilder_delete(builder);
    
    if (result != AAUDIO_OK) {
        ALOGE("Failed to open stream: %d", result);
        return -1;
    }
    
    // 获取缓冲区信息
    int32_t framesPerBurst = AAudioStream_getFramesPerBurst(stream);
    int16_t* buffer = (int16_t*)malloc(framesPerBurst * CHANNEL_COUNT * sizeof(int16_t));
    
    // 打开输出文件
    FILE* file = fopen("recording.pcm", "wb");
    
    // 启动流
    result = AAudioStream_requestStart(stream);
    
    // 录音
    int64_t totalFrames = 0;
    int64_t targetFrames = SAMPLE_RATE * DURATION_SECONDS;
    
    while (totalFrames < targetFrames) {
        int64_t framesRead = AAudioStream_read(stream, buffer, framesPerBurst, 1000000000);
        
        if (framesRead < 0) {
            ALOGE("Read error: %lld", framesRead);
            break;
        }
        
        fwrite(buffer, sizeof(int16_t), framesRead * CHANNEL_COUNT, file);
        totalFrames += framesRead;
    }
    
    // 清理
    fclose(file);
    free(buffer);
    AAudioStream_requestStop(stream);
    AAudioStream_close(stream);
    
    ALOGD("Recorded %lld frames", totalFrames);
    return 0;
}
```

---

## 6. 错误处理

### 6.1 错误码

```c
// 常见错误码
typedef enum {
    AAUDIO_OK = 0,
    AAUDIO_ERROR_BASE = -900,              // 基础错误
    AAUDIO_ERROR_DISCONNECTED = -899,      // 设备断开
    AAUDIO_ERROR_ILLEGAL_ARGUMENT = -898,  // 非法参数
    AAUDIO_ERROR_INTERNAL = -896,          // 内部错误
    AAUDIO_ERROR_INVALID_STATE = -895,     // 无效状态
    AAUDIO_ERROR_INVALID_HANDLE = -892,    // 无效句柄
    AAUDIO_ERROR_UNIMPLEMENTED = -890,     // 未实现
    AAUDIO_ERROR_UNAVAILABLE = -889,       // 不可用
    AAUDIO_ERROR_NO_FREE_HANDLES = -888,   // 无空闲句柄
    AAUDIO_ERROR_NO_MEMORY = -887,         // 内存不足
    AAUDIO_ERROR_NULL = -886,              // 空指针
    AAUDIO_ERROR_TIMEOUT = -885,           // 超时
    AAUDIO_ERROR_WOULD_BLOCK = -884,       // 会阻塞
    AAUDIO_ERROR_INVALID_FORMAT = -883,    // 无效格式
    AAUDIO_ERROR_OUT_OF_RANGE = -882,      // 超出范围
    AAUDIO_ERROR_NO_SERVICE = -881,        // 服务不可用
    AAUDIO_ERROR_INVALID_RATE = -880,      // 无效采样率
} aaudio_result_t;

// 错误处理
const char* AAudio_convertResultToText(aaudio_result_t result);
```

### 6.2 设备断开处理

```c
void errorCallback(
        AAudioStream* stream,
        void* userData,
        aaudio_result_t error) {
    
    if (error == AAUDIO_ERROR_DISCONNECTED) {
        // 设备断开，需要重新创建流
        // 在工作线程中处理
        requestStreamRestart();
    }
}

// 重新创建流
void restartStream(AAudioStream** ppStream, AAudioStreamBuilder* builder) {
    AAudioStream* oldStream = *ppStream;
    
    // 关闭旧流
    if (oldStream) {
        AAudioStream_requestStop(oldStream);
        AAudioStream_close(oldStream);
    }
    
    // 打开新流
    aaudio_result_t result = AAudioStreamBuilder_openStream(builder, ppStream);
    if (result == AAUDIO_OK) {
        AAudioStream_requestStart(*ppStream);
    }
}
```

---

## 7. 性能优化

### 7.1 缓冲区调优

```c
// 获取最优缓冲区大小
int32_t framesPerBurst = AAudioStream_getFramesPerBurst(stream);

// 设置缓冲区大小 (建议 2-3 个 burst)
int32_t optimalBufferSize = framesPerBurst * 2;
AAudioStream_setBufferSizeInFrames(stream, optimalBufferSize);

// 平衡延迟和稳定性:
// - 小缓冲区: 低延迟，易 XRUN
// - 大缓冲区: 高延迟，稳定
```

### 7.2 线程优先级

```c
// AAudio 回调运行在高优先级线程
// 回调中应避免:
// - 阻塞操作 (锁、I/O)
// - 内存分配
// - 长时间计算

// 如需处理，使用无锁队列传递到工作线程
```

### 7.3 CPU 亲和性

```c
// 在回调中设置 CPU 亲和性 (Android 10+)
// AAudio 会自动设置，通常无需手动处理
```

---

## 8. 调试

### 8.1 日志

```bash
# AAudio 日志
adb logcat -s AAudio

# AAudioService 日志
adb logcat -s AAudioService

# 查看 mmap 状态
adb logcat | grep -i mmap
```

### 8.2 dumpsys

```bash
# 查看音频流状态
dumpsys audio | grep -A 20 "AAudio"

# 查看 mmap 流
dumpsys audio | grep -i mmap
```

### 8.3 性能分析

```bash
# 查看 AAudio 延迟
dumpsys audio | grep -i latency

# 查看 XRUN 统计
dumpsys audio | grep -i xrun
```

---

## 📌 总结

| 类别 | API/概念 |
|------|----------|
| **公共 API 库** | libaaudio.so (AAudioStream, AAudioStreamBuilder) |
| **内部实现库** | libaaudio_internal.so (AudioStreamInternal, AudioStreamTrack, AudioStreamRecord) |
| **核心结构** | AAudioStream, AAudioStreamBuilder |
| **流方向** | AAUDIO_DIRECTION_OUTPUT, AAUDIO_DIRECTION_INPUT |
| **共享模式** | AAUDIO_SHARING_MODE_SHARED, AAUDIO_SHARING_MODE_EXCLUSIVE |
| **性能模式** | AAUDIO_PERFORMANCE_MODE_LOW_LATENCY, POWER_SAVING |
| **数据传输** | AAudioStream_write/read, setDataCallback |
| **流控制** | requestStart/Stop/Pause/Flush |
| **缓冲区** | getFramesPerBurst, setBufferSizeInFrames |
| **错误处理** | AAUDIO_ERROR_DISCONNECTED, errorCallback |
| **mmap** | 独占模式 + 硬件支持 |
| **调试** | logcat AAudio, dumpsys audio |
