---
name: "audio-framework"
description: "AudioFlinger 核心框架开发规范，适用于 PlaybackThread、RecordThread、Binder IPC、Effects 开发"
version: "1.0.0"
triggers: ["AudioFlinger", "AudioTrack", "AudioRecord", "PlaybackThread", "RecordThread", "MixThread", "Track", "Binder", "IInterface", "ashmem", "mmap", "status_t", "NO_ERROR", "MediaServer", "AudioEffect", "EffectHandle", "AudioPort", "AudioIoDescriptor"]
---

> 参考来源：Android AOSP AudioFlinger 源码

# 🎵 AudioFlinger 核心框架开发规范

---

## 1. AudioFlinger 架构

### 1.1 核心组件

| 组件 | 路径 | 职责 |
|------|------|------|
| `AudioTrack` | `frameworks/av/media/libaudiotrack/` | 音频回放客户端 |
| `AudioRecord` | `frameworks/av/media/libaudiorecord/` | 音频录制客户端 |
| `AudioFlinger` | `frameworks/av/services/audioflinger/` | 音频核心服务 |
| `AudioPolicyService` | `frameworks/av/services/audiopolicy/` | 音频策略管理 |
| `AudioEffects` | `frameworks/av/media/libeffects/` | 音效处理 |

### 1.2 层次结构

```
Java/App Framework
        ↓ Binder IPC
AudioTrack / AudioRecord (Native C++)
        ↓ Binder IPC
AudioFlinger (MediaServer 进程)
        ├── PlaybackThread / RecordThread
        ├── AudioEffects
        └── HAL 调用
        ↓
Audio HAL (hardware/libhardware)
        ↓
Linux Kernel (ALSA SoC)
```

---

## 2. Thread 模型

### 2.1 线程类型

```cpp
enum thread_type {
    PLAYBACK,        // 回放线程
    RECORD,          // 录制线程
    MIX,             // 混音线程
    DUPLICATING,    // 复制线程
    OFFLOAD,        // 硬解线程
    DIRECT_OUTPUT,  // 直输线程（低延迟）
};
```

### 2.2 PlaybackThread

```cpp
class PlaybackThread : public Thread {
public:
    virtual status_t initCheck() const;
    virtual status_t prepareTracks_l() = 0;
    virtual void onFirstRef();
    virtual bool threadLoop();
    
    // Track 管理
    sp<Track> createTrack_l(const audio_attributes_t& attr,
                            const audio_config_base_t* config,
                            size_t* frameCount);
    status_t destroyTrack_l(const sp<Track>& track);
    
protected:
    Mutex mLock;                          // 线程锁
    Vector<sp<Track>> mTracks;           // Track 列表
    sp<StreamOutHalInterface> mOutput;   // HAL 输出流
    audio_stream_out_t* mStream;         // HAL stream
};
```

### 2.3 MixerThread

```cpp
class MixerThread : public PlaybackThread {
public:
    MixerThread(const sp<AudioFlinger>& audioFlinger,
                AudioStreamOut* output,
                audio_io_handle_t id,
                audio_devices_t device);
    
    virtual status_t prepareTracks_l();
    virtual void threadLoop_mix();
    
private:
    AudioMixer* mAudioMixer;             // 软件混音器
    EffectChain* mEffectChain;           // 音效链
};
```

### 2.4 RecordThread

```cpp
class RecordThread : public Thread {
public:
    virtual bool threadLoop();
    
    // 从 HAL 读取数据
    ssize_t getInputFramesLost();
    
protected:
    sp<StreamInHalInterface> mInput;     // HAL 输入流
    Vector<sp<RecordTrack>> mTracks;     // 录音 Track 列表
};
```

---

## 3. Track 模型

### 3.1 Track 状态机

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│   IDLE ──→ RESTARTING ──→ STOPPED ──→ STOPPING     │
│     │          │              │           │        │
│     │          ↓              ↓           ↓        │
│     └────→ STARTED ←──── PAUSED ←──── PAUSING      │
│               │              │                     │
│               ↓              ↓                     │
│           FLUSHED ←─────────────────────────────── │
└─────────────────────────────────────────────────────┘
```

### 3.2 Track 关键类

```cpp
class Track : public RefBase {
public:
    // 状态控制
    status_t start();
    status_t stop();
    status_t pause();
    status_t flush();
    
    // 数据读写
    ssize_t write(const void* buffer, size_t bytes);
    ssize_t read(void* buffer, size_t bytes);
    
    // 属性查询
    bool isOffloaded() const;    // Offload 模式
    bool isFast() const;         // Fast Track (低延迟)
    bool isDirect() const;       // 直输模式
    
    // Buffer 信息
    size_t frameCount() const;
    size_t bufferSize() const;
    
private:
    sp<IMemory> mCblkMemory;     // 共享内存控制块
    audio_track_cblk_t* mCblk;   // 控制块指针
    void* mBuffer;               // 音频缓冲区
};
```

### 3.3 Fast Track (低延迟)

```cpp
// Fast Track 配置
audio_output_flags_t flags = AUDIO_OUTPUT_FLAG_FAST;

// Fast Track 特点
// 1. 独立的 Mixer 线程
// 2. 更小的缓冲区
// 3. 优先级更高的调度
// 4. 不参与普通混音
```

---

## 4. Audio Effects 框架

### 4.1 Effect 模块结构

```
frameworks/av/media/libeffects/
├── downmix/         // 下混
├── dynamicsprocessing/  // 动态处理
├── equalizer/       // 均衡器
├── loudnessenhancer/  // 响度增强
├── reverb/          // 混响
├── visualizer/      // 可视化
└── bundle/          // 效果包
```

### 4.2 Effect 接口

```cpp
class EffectHandle : public android::RefBase {
public:
    // 生命周期
    status_t init();
    status_t enable();
    status_t disable();
    status_t release();
    
    // 参数控制
    status_t setParameter(effect_param_t* param);
    status_t getParameter(effect_param_t* param);
    
    // 命令
    status_t command(int cmdCode, void* cmdData,
                     void* reply, size_t* replySize);
};

// Effect 描述
typedef struct effect_descriptor_s {
    effect_uuid_t type;       // 效果类型 UUID
    effect_uuid_t uuid;       // 实例 UUID
    uint32_t apiVersion;      // API 版本
    uint32_t flags;           // 标志
    char name[EFFECT_STRING_LEN_MAX];   // 名称
    char implementor[EFFECT_STRING_LEN_MAX];
} effect_descriptor_t;
```

### 4.3 EffectChain

```cpp
class EffectChain : public RefBase {
public:
    // 添加/移除 Effect
    status_t addEffect(const sp<EffectHandle>& handle);
    status_t removeEffect(const sp<EffectHandle>& handle);
    
    // 处理音频数据
    void process(float* in, float* out, size_t frames);
    
private:
    Vector<sp<EffectHandle>> mEffects;  // Effect 列表
    Mutex mLock;
};
```

### 4.4 创建 Effect

```cpp
// 创建音效实例
sp<AudioEffect> effect = new AudioEffect(
    EFFECT_TYPE_EQUALIZER,    // 效果类型
    String16("com.example"),  // 包名
    0,                        // priority
    new EffectCallback());    // 回调

status_t status = effect->initCheck();
if (status != NO_ERROR) {
    ALOGE("Failed to create effect: %d", status);
    return;
}

// 设置参数
effect->setParameter(PARAM_EQ_BAND_0, -3.0f);

// 启用效果
effect->enable();
```

---

## 5. AudioPort 与 AudioPatch

### 5.1 AudioPort

```cpp
struct audio_port {
    audio_port_role_t role;          // source/sink
    audio_port_type_t type;          // device/mix/session
    char name[AUDIO_PORT_NAME_LEN];
    unsigned int num_sample_rates;
    unsigned int sample_rates[AUDIO_PORT_MAX_SAMPLING_RATES];
    unsigned int num_channel_masks;
    audio_channel_mask_t channel_masks[AUDIO_PORT_MAX_CHANNEL_MASKS];
    unsigned int num_formats;
    audio_format_t formats[AUDIO_PORT_MAX_FORMATS];
    struct audio_gain gains[AUDIO_PORT_MAX_GAINS];
};
```

### 5.2 AudioPortConfig

```cpp
struct audio_port_config {
    audio_port_role_t role;
    audio_port_type_t type;
    audio_port_handle_t id;
    unsigned int config_mask;        // 配置掩码
    unsigned int sample_rate;        // 采样率
    audio_channel_mask_t channel_mask; // 通道掩码
    audio_format_t format;           // 格式
    struct audio_gain_config gain;   // 增益配置
    audio_devices_t device;          // 设备类型
    char address[AUDIO_DEVICE_MAX_ADDRESS_LEN]; // 设备地址
};
```

### 5.3 AudioPatch

```cpp
struct audio_patch {
    audio_patch_handle_t id;
    unsigned int num_sources;
    struct audio_port_config sources[AUDIO_PATCH_PORTS_MAX];
    unsigned int num_sinks;
    struct audio_port_config sinks[AUDIO_PATCH_PORTS_MAX];
};
```

---

## 6. Binder IPC (Native 层)

### 6.1 IAudioFlinger 接口

```cpp
class IAudioFlinger : public IInterface {
public:
    DECLARE_META_INTERFACE(AudioFlinger);
    
    // Track 创建
    virtual sp<IAudioTrack> createTrack(
            audio_stream_type_t streamType,
            uint32_t sampleRate,
            audio_format_t format,
            audio_channel_mask_t channelMask,
            size_t frameCount,
            audio_output_flags_t flags,
            const sp<IMemory>& sharedBuffer,
            audio_io_handle_t output,
            pid_t tid,
            int* sessionId,
            size_t* notificationFrames) = 0;
    
    // 输出管理
    virtual audio_io_handle_t openOutput(
            audio_module_handle_t module,
            audio_devices_t* devices,
            uint32_t* samplingRate,
            audio_format_t* format,
            audio_channel_mask_t* channelMask,
            uint32_t* latency,
            audio_output_flags_t flags) = 0;
    
    // 设备管理
    virtual status_t setDevicePortConfig(
            const struct audio_port_config* config) = 0;
    
    // Patch 管理
    virtual status_t createAudioPatch(
            const struct audio_patch* patch,
            audio_patch_handle_t* handle) = 0;
};
```

### 6.2 Parcel 序列化

```cpp
// 写入 audio_config
void writeAudioConfig(Parcel* parcel, const audio_config_t* config) {
    parcel->writeUint32(config->sample_rate);
    parcel->writeInt32(config->channel_mask);
    parcel->writeInt32(config->format);
}

// 读取 audio_config
void readAudioConfig(const Parcel& parcel, audio_config_t* config) {
    config->sample_rate = parcel->readUint32();
    config->channel_mask = parcel->readInt32();
    config->format = parcel->readInt32();
}
```

---

## 7. 内存管理

### 7.1 ashmem 共享内存

```cpp
#include <sys/mman.h>
#include <cutils/ashmem.h>

// 创建共享内存
int fd = ashmem_create_region("audio_track", size);
ashmem_set_prot_region(fd, PROT_READ | PROT_WRITE);

// 映射
void* addr = mmap(nullptr, size, PROT_READ | PROT_WRITE,
                   MAP_SHARED, fd, 0);

// 释放
munmap(addr, size);
close(fd);
```

### 7.2 audio_track_cblk_t 控制块

```cpp
struct audio_track_cblk_t {
    volatile uint32_t user;       // 用户写入位置
    volatile uint32_t server;     // 服务端读取位置
    volatile uint32_t userBase;   // 用户基准
    volatile uint32_t serverBase; // 服务端基准
    uint32_t frameCount;          // 帧数
    uint32_t bufferSize;          // 缓冲区大小
    uint8_t* buffer;              // 缓冲区指针
    Mutex lock;                   // 锁
    Condition cv;                 // 条件变量
};
```

---

## 8. 线程安全

### 8.1 锁策略

```cpp
// AudioFlinger 锁层次
// 1. mLock (AudioFlinger 全局锁)
// 2. mLock (Thread 锁)
// 3. mLock (Track 锁)

// 正确的锁顺序
void AudioFlinger::someMethod() {
    AutoMutex lock(mLock);           // 先获取全局锁
    {
        AutoMutex threadLock(thread->mLock);  // 再获取线程锁
        // 操作...
    }
}
```

### 8.2 Lock-free 热路径

```cpp
// 音频处理路径使用原子操作
class FastMixer {
    std::atomic<uint32_t> mWriteIndex{0};
    std::atomic<uint32_t> mReadIndex{0};
    
    void write(const void* data, size_t frames) {
        uint32_t index = mWriteIndex.load(std::memory_order_relaxed);
        memcpy(mBuffer + index * mFrameSize, data, frames * mFrameSize);
        mWriteIndex.store((index + frames) % mBufferFrames,
                          std::memory_order_release);
    }
};
```

---

## 9. 调试技巧

### 9.1 dumpsys

```bash
# 完整音频状态
dumpsys audio

# 查看输出线程
dumpsys audio | grep -A 50 "Output thread"

# 查看活动 Track
dumpsys audio | grep -A 20 "Active Track"

# 查看音效链
dumpsys audio | grep -A 30 "EffectChain"
```

### 9.2 日志过滤

```bash
# AudioFlinger 核心日志
adb logcat -s AudioFlinger

# 线程相关
adb logcat | grep -E "PlaybackThread|RecordThread|MixerThread"

# Track 状态变化
adb logcat | grep -E "Track::(start|stop|pause|flush)"

# HAL 调用
adb logcat | grep -E "openOutputStream|closeOutputStream|write"
```

### 9.3 常见问题定位

```bash
# 1. 无声问题
dumpsys audio | grep -i "state\|status"
adb logcat | grep -i "underrun\|error"

# 2. 延迟问题
dumpsys audio | grep -i "latency"
adb logcat | grep -i "buffer"

# 3. 音效问题
dumpsys audio | grep -A 20 "Effect"
adb logcat | grep -E "AudioEffect|EffectHandle"
```

### 9.4 ATRACE

```bash
# 启用音频追踪
atrace --app audio

# 或使用 systrace
python $ANDROID/sdk/platform-tools/systrace/systrace.py \
    --app=audioflinger -t 5 -o trace.html
```

---

## 10. 错误处理

### 10.1 status_t 错误码

```cpp
#include <utils/Errors.h>

// 常用错误码
NO_ERROR              = 0       // 成功
BAD_VALUE             = -EINVAL // 参数错误
INVALID_OPERATION     = -ENOSYS // 操作无效
NOT_ENOUGH_DATA       = -ENODATA
WOULD_BLOCK           = -EWOULDBLOCK
TIMED_OUT             = -ETIMEDOUT
NO_MEMORY             = -ENOMEM
BUSY                  = -EBUSY
```

### 10.2 错误处理模式

```cpp
status_t AudioFlinger::openOutput(...) {
    // 1. 参数检查
    if (devices == nullptr || samplingRate == nullptr) {
        return BAD_VALUE;
    }
    
    // 2. 资源分配
    sp<PlaybackThread> thread = new MixerThread(...);
    status_t status = thread->initCheck();
    if (status != NO_ERROR) {
        ALOGE("Failed to init playback thread: %d", status);
        return status;
    }
    
    // 3. 注册线程
    {
        AutoMutex lock(mLock);
        mPlaybackThreads.add(id, thread);
    }
    
    return NO_ERROR;
}
```

---

## 📌 总结

| 类别 | 规则 |
|------|------|
| **Thread** | PlaybackThread / RecordThread / MixerThread |
| **Track** | 状态机管理，Fast Track 低延迟 |
| **Effects** | EffectHandle / EffectChain 音效链 |
| **AudioPort** | 设备端口配置与 Patch 管理 |
| **Binder** | IInterface / Parcel 序列化 |
| **内存** | ashmem / audio_track_cblk_t |
| **并发** | 锁层次正确，热路径 Lock-free |
| **调试** | dumpsys / logcat / ATRACE |
