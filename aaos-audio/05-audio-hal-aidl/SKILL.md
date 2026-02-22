---
name: "audio-hal-aidl"
description: "Android AIDL Audio HAL 开发规范，处理 IModule、IDevice、IStreamOut/In 接口实现与 FMQ 数据传输"
version: "2.0.0"
triggers: ["AIDL", "IModule", "IDevice", "IStreamOut", "IStreamIn", "MQDescriptor", "FMQ", "EventFlag", "audio core", "EX_SERVICE_SPECIFIC", "ScopedAStatus", "BnModule", "BnStreamOut", "aidl_interface", "vintf"]
---

> 参考来源：Android AOSP hardware/interfaces/audio/aidl

# 🎧 Android AIDL Audio HAL 开发规范

---

## 1. AIDL Audio HAL 架构

### 1.1 版本演进

```
HIDL HAL (Legacy)              AIDL HAL (New)
┌─────────────────┐            ┌─────────────────┐
│ audio@4.0/6.0   │            │ audio.core      │
│ (Android 8-14)  │   ───→     │ (Android 13+)   │
│ IDevicesFactory │            │ IModule/IDevice │
└─────────────────┘            └─────────────────┘
```

### 1.2 接口层级

```
AudioFlinger (Native)
        ↓ AIDL Binder IPC
IModule / IDevice
        ├── IStreamOut (播放流)
        ├── IStreamIn (录音流)
        └── ISoundDose (声音剂量)
        ↓
Audio HAL Implementation (厂商实现)
        ↓
TinyALSA / DSP Driver
```

### 1.3 源码路径

```
hardware/interfaces/audio/aidl/
├── android/hardware/audio/core/
│   ├── IModule.aidl           # 模块入口接口
│   ├── IDevice.aidl           # 设备接口
│   ├── IStreamOut.aidl        # 输出流接口
│   ├── IStreamIn.aidl         # 输入流接口
│   ├── ISoundDose.aidl        # 声音剂量接口
│   └── IModuleCallback.aidl   # 模块回调
├── android/hardware/audio/common/
│   └── types.aidl             # 公共类型定义
└── default/                   # 默认实现参考
```

---

## 2. 核心接口

### 2.1 IModule (模块入口)

```aidl
package android.hardware.audio.core;

interface IModule {
    // 获取模块 ID
    int getModuleId();

    // 打开设备
    IDevice openDevice(in String deviceName);

    // 打开输出流
    IStreamOut openOutputStream(
        in String deviceName,
        in AudioConfig config,
        in AudioSource source,
        in int flags
    );

    // 打开输入流
    IStreamIn openInputStream(
        in String deviceName,
        in AudioConfig config,
        in AudioSource source,
        in int flags
    );

    // AudioPatch 管理
    int createAudioPatch(in AudioPatch patch);
    void releaseAudioPatch(int patchId);

    // AudioPort 管理
    AudioPort[] getAudioPorts();
    void setAudioPortConfig(in AudioPortConfig config);

    // 注册回调
    void registerModuleCallback(in IModuleCallback callback);
}
```

### 2.2 IDevice (设备接口)

```aidl
package android.hardware.audio.core;

interface IDevice {
    // 初始化检查
    void initCheck();

    // 主音量控制
    void setMasterVolume(float volume);
    float getMasterVolume();
    void setMasterMute(boolean muted);
    boolean getMasterMute();

    // 麦克风控制
    void setMicMute(boolean muted);
    boolean getMicMute();

    // 音频模式
    void setMode(in AudioMode mode);
    AudioMode getMode();

    // 参数设置
    void setParameters(in String parameters);
    String getParameters(in String keys);

    // AudioPatch 支持
    boolean supportsAudioPatches();

    // 屏幕状态
    void setScreenState(boolean turnedOn);

    // 硬件同步
    int getHwAvSync();
}
```

### 2.3 IStreamOut (输出流)

```aidl
package android.hardware.audio.core;

interface IStreamOut {
    // 基本属性
    int getFrameSize();
    long getFrameCount();
    int getBufferSize();
    int getSampleRate();
    AudioChannelMask getChannelMask();
    AudioFormat getFormat();

    // FMQ 数据传输
    MQDescriptor<byte, SynchronizedReadWrite> prepareForWriting(
        in int frameSize,
        in int framesCount
    );

    // 延迟与位置
    int getLatencyMs();
    void getRenderPosition(out long frames, out long timestamp);
    void getPresentationPosition(out long frames, out long timestamp);

    // 音量控制
    void setVolume(float left, float right);

    // 流控制
    void standby();
    void pause();
    void resume();
    void flush();
    void drain(in DrainType type);

    // 设备
    void setDevice(in AudioDevice device);
    AudioDevice getDevice();

    // 参数
    void setParameters(in String parameters);
    String getParameters(in String keys);

    // 关闭
    void close();
}
```

### 2.4 IStreamIn (输入流)

```aidl
package android.hardware.audio.core;

interface IStreamIn {
    // 基本属性 (同 IStreamOut)
    int getFrameSize();
    long getFrameCount();
    int getBufferSize();
    int getSampleRate();
    AudioChannelMask getChannelMask();
    AudioFormat getFormat();

    // FMQ 数据传输
    MQDescriptor<byte, SynchronizedReadWrite> prepareForReading(
        in int frameSize,
        in int framesCount
    );

    // 增益控制
    void setGain(float gain);

    // 位置信息
    void getCapturePosition(out long frames, out long timestamp);
    int getInputFramesLost();

    // 音频源
    AudioSource getAudioSource();
    void setAudioSource(in AudioSource source);

    // 流控制
    void standby();

    // 设备
    void setDevice(in AudioDevice device);
    AudioDevice getDevice();

    // 参数
    void setParameters(in String parameters);
    String getParameters(in String keys);

    // 关闭
    void close();
}
```

---

## 3. FMQ 数据传输

### 3.1 MQDescriptor 结构

```cpp
// 路径: hardware/interfaces/common/fmq/aidl/android/hardware/common/fmq/MQDescriptor.aidl
// FMQ 本质是共享内存，通过 Binder 传输描述符

parcelable MQDescriptor<T, flag> {
    // grantors: 描述共享内存分段结构
    GrantorDescriptor[] grantors;
    // handle: 共享内存句柄
    ParcelFileDescriptor handle;
    // quantum: 元素大小
    int quantum;
    // flags: FMQ 类型
    int flags;
}
```

### 3.2 FMQ 类型

```cpp
// 两种 FMQ 类型:
// 1. SynchronizedReadWrite - 同步队列，只有一个读取端
// 2. UnsynchronizedWrite - 非同步队列，可有多个读取端

// SynchronizedReadWrite 特点:
// - 一个写入端，一个读取端
// - 写入位置和读取位置都有记录
// - 支持阻塞和非阻塞操作
```

### 3.3 AidlMessageQueue 使用

```cpp
#include <fmq/AidlMessageQueue.h>

using AidlMessageQueue = android::AidlMessageQueue<uint8_t, 
                          android::SynchronizedReadWrite>;

// 创建 FMQ (服务端)
std::unique_ptr<AidlMessageQueue> createFmq(size_t bufferSize) {
    // 创建同步 FMQ
    auto fmq = std::make_unique<AidlMessageQueue>(
        bufferSize, 
        true /* configureEventFlagWord */
    );
    
    if (!fmq->isValid()) {
        ALOGE("Failed to create FMQ");
        return nullptr;
    }
    
    return fmq;
}

// 获取 MQDescriptor 传给客户端
MQDescriptor<uint8_t, SynchronizedReadWrite> getDescriptor(
        const AidlMessageQueue& fmq) {
    return fmq.dupeDescriptor();
}
```

### 3.4 数据写入 (HAL 端)

```cpp
// 写入音频数据到 FMQ
size_t StreamOut::write(const void* buffer, size_t bytes) {
    if (!mFmq || !mFmq->isValid()) {
        return 0;
    }
    
    const uint8_t* data = static_cast<const uint8_t*>(buffer);
    size_t available = mFmq->availableToWrite();
    size_t toWrite = std::min(bytes, available);
    
    if (toWrite == 0) {
        return 0;
    }
    
    // 写入数据
    bool success = mFmq->write(data, toWrite);
    if (!success) {
        ALOGE("FMQ write failed");
        return 0;
    }
    
    // 通知客户端数据可用
    if (mEventFlag) {
        mEventFlag->wake(kDataAvailable);
    }
    
    return toWrite;
}
```

### 3.5 EventFlag 同步

```cpp
#include <fmq/EventFlag.h>

// EventFlag 标志定义
static constexpr uint32_t kDataAvailable = 1 << 0;
static constexpr uint32_t kDataConsumed = 1 << 1;

// 创建 EventFlag
android::status_t createEventFlag(AidlMessageQueue& fmq) {
    android::EventFlag* ef = nullptr;
    android::status_t status = android::EventFlag::createEventFlag(
        fmq.getEventFlagWord(), 
        &ef
    );
    
    if (status != android::OK || ef == nullptr) {
        ALOGE("Failed to create EventFlag");
        return status;
    }
    
    mEventFlag.reset(ef);
    return android::OK;
}

// 等待数据
void waitForData() {
    uint32_t state = 0;
    android::status_t status = mEventFlag->wait(
        kDataAvailable,    // 等待的标志
        &state,            // 返回的状态
        5000000000ULL      // 超时 (纳秒)
    );
    
    if (status != android::OK) {
        ALOGW("EventFlag wait failed: %d", status);
    }
}

// 通知数据已消费
void notifyDataConsumed() {
    if (mEventFlag) {
        mEventFlag->wake(kDataConsumed);
    }
}
```

---

## 4. 服务实现

### 4.1 继承 Bn 接口

```cpp
#include <aidl/android/hardware/audio/core/BnModule.h>
#include <aidl/android/hardware/audio/core/BnStreamOut.h>

namespace aidl::android::hardware::audio::core {

class Module : public BnModule {
public:
    ndk::ScopedAStatus openDevice(const std::string& deviceName,
                                   std::shared_ptr<IDevice>* device) override;
    
    ndk::ScopedAStatus openOutputStream(
            const std::string& deviceName,
            const AudioConfig& config,
            AudioSource source,
            int32_t flags,
            std::shared_ptr<IStreamOut>* stream) override;
    
    ndk::ScopedAStatus openInputStream(
            const std::string& deviceName,
            const AudioConfig& config,
            AudioSource source,
            int32_t flags,
            std::shared_ptr<IStreamIn>* stream) override;
    
    ndk::ScopedAStatus createAudioPatch(const AudioPatch& patch,
                                        int32_t* patchId) override;
    
    ndk::ScopedAStatus releaseAudioPatch(int32_t patchId) override;

private:
    std::map<int32_t, std::shared_ptr<IStreamOut>> mOutputStreams;
    std::map<int32_t, std::shared_ptr<IStreamIn>> mInputStreams;
};

class StreamOut : public BnStreamOut {
public:
    StreamOut(const AudioConfig& config);
    
    ndk::ScopedAStatus prepareForWriting(
            int32_t frameSize,
            int32_t framesCount,
            MQDescriptor<uint8_t, SynchronizedReadWrite>* desc) override;
    
    ndk::ScopedAStatus getLatencyMs(int32_t* latencyMs) override;
    ndk::ScopedAStatus setVolume(float left, float right) override;
    ndk::ScopedAStatus standby() override;
    ndk::ScopedAStatus close() override;

private:
    std::unique_ptr<AidlMessageQueue> mFmq;
    std::unique_ptr<android::EventFlag> mEventFlag;
    AudioConfig mConfig;
};

}  // namespace aidl::android::hardware::audio::core
```

### 4.2 服务注册

```cpp
// main.cpp
#include <aidl/android/hardware/audio/core/IModule.h>
#include <android/binder_manager.h>
#include <android/binder_process.h>

using aidl::android::hardware::audio::core::IModule;
using aidl::android::hardware::audio::core::Module;

int main() {
    // 设置 Binder 线程池
    ABinderProcess_setThreadPoolMaxThreadCount(4);
    
    // 创建服务实例
    std::shared_ptr<Module> module = ndk::SharedRefBase::make<Module>();
    
    // 注册服务
    const std::string instance = std::string(IModule::descriptor) + "/default";
    binder_status_t status = AServiceManager_addService(
        module->asBinder().get(),
        instance.c_str()
    );
    
    if (status != STATUS_OK) {
        ALOGE("Failed to register service: %d", status);
        return 1;
    }
    
    ALOGI("Audio AIDL HAL service registered: %s", instance.c_str());
    
    // 进入 Binder 循环
    ABinderProcess_joinThreadPool();
    
    return 0;
}
```

### 4.3 Android.bp 配置

```bp
// aidl_interface 定义
aidl_interface {
    name: "android.hardware.audio.core",
    vendor_available: true,
    srcs: ["android/hardware/audio/core/*.aidl"],
    imports: [
        "android.hardware.audio.common-V1",
    ],
    stability: "vintf",
    backend: {
        cpp: {
            enabled: true,
        },
        java: {
            sdk_version: "module_current",
        },
    },
}

// 服务模块
cc_binary {
    name: "android.hardware.audio.service",
    vendor: true,
    srcs: ["main.cpp", "Module.cpp", "StreamOut.cpp", "StreamIn.cpp"],
    shared_libs: [
        "libbase",
        "libbinder_ndk",
        "libfmq",
        "liblog",
        "android.hardware.audio.core-V1-ndk",
    ],
    init_rc: ["android.hardware.audio.service.rc"],
    vintf_fragments: ["android.hardware.audio.service.xml"],
}
```

---

## 5. VINTF 配置

### 5.1 服务 RC 文件

```rc
# android.hardware.audio.service.rc
service vendor.audio-hal /vendor/bin/hw/android.hardware.audio.service
    class hal
    user audioserver
    group audio system
    capabilities BLOCK_SUSPEND
    onrestart restart audioserver
```

### 5.2 VINTF Manifest

```xml
<!-- android.hardware.audio.service.xml -->
<manifest version="1.0" type="device">
    <hal format="aidl">
        <name>android.hardware.audio.core</name>
        <version>1</version>
        <fqname>IModule/default</fqname>
    </hal>
</manifest>
```

### 5.3 兼容性矩阵

```xml
<!-- compatibility_matrix.xml -->
<hal format="aidl" optional="true">
    <name>android.hardware.audio.core</name>
    <version>1</version>
    <interface>
        <name>IModule</name>
        <instance>default</instance>
    </interface>
</hal>
```

---

## 6. 错误处理

### 6.1 ScopedAStatus

```cpp
// 成功返回
ndk::ScopedAStatus StreamOut::getLatencyMs(int32_t* latencyMs) {
    *latencyMs = mLatencyMs;
    return ndk::ScopedAStatus::ok();
}

// 服务特定错误
ndk::ScopedAStatus StreamOut::setVolume(float left, float right) {
    if (left < 0.0f || left > 1.0f || right < 0.0f || right > 1.0f) {
        return ndk::ScopedAStatus::fromServiceSpecificError(
            STATUS_BAD_VALUE
        );
    }
    
    mLeftVolume = left;
    mRightVolume = right;
    return ndk::ScopedAStatus::ok();
}

// 异常错误
ndk::ScopedAStatus Module::openOutputStream(...) {
    if (!isDeviceAvailable(deviceName)) {
        return ndk::ScopedAStatus::fromExceptionCode(
            EX_ILLEGAL_ARGUMENT
        );
    }
    // ...
}
```

### 6.2 错误码定义

```cpp
// 常用错误码
enum {
    STATUS_OK = 0,
    STATUS_BAD_VALUE = -EINVAL,      // -22
    STATUS_INVALID_OPERATION = -ENOSYS, // -38
    STATUS_NO_MEMORY = -ENOMEM,      // -12
    STATUS_BUSY = -EBUSY,            // -16
    STATUS_DEAD_OBJECT = -EPIPE,     // -32
};

// Binder 异常码
enum {
    EX_NONE = 0,
    EX_SECURITY = -1,
    EX_BAD_PARCELABLE = -2,
    EX_ILLEGAL_ARGUMENT = -3,
    EX_NULL_POINTER = -4,
    EX_ILLEGAL_STATE = -5,
    EX_NETWORK_MAIN_THREAD = -6,
    EX_UNSUPPORTED_OPERATION = -7,
    EX_SERVICE_SPECIFIC = -8,
    EX_PARCELABLE = -9,
};

// 错误映射函数
ndk::ScopedAStatus mapStatus(int err) {
    if (err == 0) {
        return ndk::ScopedAStatus::ok();
    }
    return ndk::ScopedAStatus::fromServiceSpecificError(err);
}
```

---

## 7. 音频配置结构

### 7.1 AudioConfig

```cpp
// AudioConfig.aidl
parcelable AudioConfig {
    int sampleRate;              // 采样率
    AudioChannelMask channelMask; // 通道掩码
    AudioFormat format;          // 音频格式
    int frameCount;              // 帧数
}

// 使用示例
AudioConfig config = {
    .sampleRate = 48000,
    .channelMask = AudioChannelMask::OUT_STEREO,
    .format = AudioFormat::PCM_16_BIT,
    .frameCount = 960,  // 20ms @ 48kHz
};
```

### 7.2 AudioDevice

```cpp
// AudioDevice.aidl
parcelable AudioDevice {
    AudioDeviceType type;        // 设备类型
    String address;              // 设备地址 (如 bus0_media_out)
}

// 设备类型
enum AudioDeviceType {
    NONE = 0,
    // 输出设备
    OUT_SPEAKER = 1,
    OUT_EARPIECE = 2,
    OUT_HEADPHONE = 3,
    OUT_WIRED_HEADSET = 4,
    OUT_BLUETOOTH_SCO = 5,
    OUT_BLUETOOTH_A2DP = 6,
    OUT_HDMI = 7,
    OUT_USB_DEVICE = 8,
    OUT_BUS = 9,  // 车载 Bus
    // 输入设备
    IN_BUILTIN_MIC = 100,
    IN_BLUETOOTH_SCO_HEADSET = 101,
    IN_WIRED_HEADSET = 102,
    IN_USB_DEVICE = 103,
    IN_BUS = 104,
};
```

### 7.3 AudioPatch

```cpp
// AudioPatch.aidl
parcelable AudioPatch {
    int id;
    AudioPortConfig[] sources;  // 源端口
    AudioPortConfig[] sinks;    // 目标端口
}

// AudioPortConfig.aidl
parcelable AudioPortConfig {
    int id;
    AudioDevice device;
    int sampleRate;
    AudioChannelMask channelMask;
    AudioFormat format;
    AudioGainConfig gain;
}
```

---

## 8. 调试

### 8.1 日志

```cpp
#include <android-base/logging.h>

// 使用 Android 日志
ALOGV("Verbose log");
ALOGD("Debug log");
ALOGI("Info log");
ALOGW("Warning log");
ALOGE("Error log: %s", strerror(errno));

// 使用 LOG 宏
LOG(INFO) << "Stream opened with sample rate: " << config.sampleRate;
LOG(ERROR) << "Failed to write: " << status;
```

### 8.2 dumpsys

```bash
# 查看音频 HAL 状态
dumpsys audio

# 查看详细 HAL 信息
dumpsys audio | grep -A 50 "HAL"

# 查看 AIDL 服务
service list | grep audio

# 查看 VINTF 清单
dumpsys vintf
```

### 8.3 常见问题定位

```bash
# 1. 服务未启动
adb shell service list | grep audio
adb logcat | grep "audio.*service"

# 2. FMQ 问题
adb logcat | grep -E "FMQ|MessageQueue"

# 3. Binder 错误
adb logcat | grep -E "Binder|AIDL"

# 4. VINTF 兼容性问题
adb shell dumpsys vintf | grep audio
```

---

## 📌 总结

| 类别 | 规则 |
|------|------|
| **接口** | IModule / IDevice / IStreamOut / IStreamIn |
| **基类** | BnModule / BnDevice / BnStreamOut / BnStreamIn |
| **数据传输** | MQDescriptor + AidlMessageQueue (FMQ) |
| **同步** | EventFlag (kDataAvailable / kDataConsumed) |
| **错误** | ScopedAStatus + EX_* 异常码 |
| **服务注册** | AServiceManager_addService |
| **配置** | aidl_interface + vintf_fragments |
| **VINTF** | manifest.xml + compatibility_matrix |
| **调试** | dumpsys audio / logcat |
