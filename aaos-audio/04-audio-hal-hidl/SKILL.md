---
name: "audio-hal-hidl"
description: "Android HIDL Audio HAL 开发规范，适用于 IDevicesFactory、IDevice、IStreamOut/In 接口实现 (Android 8-14)"
version: "2.0.0"
triggers: ["HIDL", "IDevicesFactory", "IDevice", "IPrimaryDevice", "IStreamOut", "IStreamIn", "HIDL_FETCH", "hidl_interface", "passthrough", "binderized", "audio_hw_device", "Result"]
---

> 参考来源：Android AOSP hardware/interfaces/audio

# 🔌 HIDL Audio HAL 开发规范

---

## 1. HIDL 架构概述

### 1.1 HIDL 简介

```
HIDL (Hardware Interface Definition Language)
- 全称: HAL Interface Definition Language
- 目的: 使 Android 可以在不重新编译 HAL 的情况下对 Framework 进行 OTA 升级
- 通信: Binder IPC
- 文件: .hal 格式
```

### 1.2 版本演进

```
┌─────────────────────────────────────────────────────────────────┐
│  Legacy HAL        HIDL HAL              AIDL HAL              │
│  (Android 7-)      (Android 8-14)        (Android 13+)         │
├─────────────────────────────────────────────────────────────────┤
│  audio.primary.so  audio@2.0-6.0         audio.core            │
│  (C 结构体)         IDevicesFactory       IModule               │
│                     IDevice               IDevice               │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 源码路径

```
hardware/interfaces/audio/
├── 2.0/                        # HIDL v2.0
│   ├── IDevice.hal
│   ├── IDevicesFactory.hal
│   ├── IPrimaryDevice.hal
│   ├── IStream.hal
│   ├── IStreamIn.hal
│   ├── IStreamOut.hal
│   └── types.hal
├── 4.0/                        # HIDL v4.0
├── 5.0/                        # HIDL v5.0
├── 6.0/                        # HIDL v6.0 (最新 HIDL 版本)
├── common/                     # 公共实现
├── core/                       # AIDL 接口
└── effect/                     # 音效 HAL
```

---

## 2. 核心接口

### 2.1 IDevicesFactory (设备工厂)

```hal
// hardware/interfaces/audio/6.0/IDevicesFactory.hal
package android.hardware.audio@6.0;

interface IDevicesFactory {
    // 打开设备 (v4.0+)
    openDevice(string moduleName)
        generates (Result retval, IDevice device);
    
    // 打开主设备 (v4.0+)
    openPrimaryDevice()
        generates (Result retval, IPrimaryDevice device);
};

// v2.0 使用枚举类型
interface IDevicesFactory {
    enum Device : int32_t {
        PRIMARY,
        A2DP,
        USB,
        R_SUBMIX,
        STUB
    };
    
    openDevice(Device device)
        generates (Result retval, IDevice device);
};
```

### 2.2 IDevice (设备接口)

```hal
// hardware/interfaces/audio/6.0/IDevice.hal
package android.hardware.audio@6.0;

interface IDevice {
    // 初始化检查
    initCheck() generates (Result retval);
    
    // 主音量控制
    setMasterVolume(float volume) generates (Result retval);
    getMasterVolume() generates (Result retval, float volume);
    setMasterMute(bool muted) generates (Result retval);
    getMasterMute() generates (Result retval, bool muted);
    
    // 麦克风控制
    setMicMute(bool muted) generates (Result retval);
    getMicMute() generates (Result retval, bool muted);
    
    // 输入缓冲区大小
    getInputBufferSize(AudioConfig config)
        generates (Result retval, uint64_t bufferSize);
    
    // 打开流
    openOutputStream(
        int32_t ioHandle,
        DeviceAddress device,
        AudioConfig config,
        bitfield<AudioOutputFlag> flags,
        AudioSource source
    ) generates (
        Result retval,
        IStreamOut stream,
        AudioConfig suggestedConfig
    );
    
    openInputStream(
        int32_t ioHandle,
        DeviceAddress device,
        AudioConfig config,
        bitfield<AudioInputFlag> flags,
        AudioSource source,
        string devices
    ) generates (
        Result retval,
        IStreamIn stream,
        AudioConfig suggestedConfig
    );
    
    // AudioPatch 支持
    supportsAudioPatches() generates (bool supports);
    createAudioPatch(vec<AudioPortConfig> sources,
                     vec<AudioPortConfig> sinks)
        generates (Result retval, AudioPatchHandle patch);
    releaseAudioPatch(AudioPatchHandle patch) generates (Result retval);
    
    // AudioPort 管理
    getAudioPort(AudioPort port) generates (Result retval, AudioPort port);
    setAudioPortConfig(AudioPortConfig config) generates (Result retval);
    
    // 硬件同步
    getHwAvSync() generates (Result retval, AudioHwSync hwAvSync);
    
    // 屏幕状态
    setScreenState(bool turnedOn) generates (Result retval);
    
    // 参数
    getParameters(vec<string> keys)
        generates (Result retval, vec<ParameterValue> parameters);
    setParameters(vec<ParameterValue> parameters) generates (Result retval);
    
    // 关闭
    close() generates (Result retval);
};
```

### 2.3 IPrimaryDevice (主设备)

```hal
// hardware/interfaces/audio/6.0/IPrimaryDevice.hal
package android.hardware.audio@6.0;

interface IPrimaryDevice extends IDevice {
    // 音频模式
    setMode(AudioMode mode) generates (Result retval);
    
    // 蓝牙 SCO
    setBtScoNrecEnabled(bool enabled) generates (Result retval);
    setBtScoWidebandEnabled(bool enabled) generates (Result retval);
    
    // TTY 模式
    setTtyMode(TtyMode mode) generates (Result retval);
    getTtyMode() generates (Result retval, TtyMode mode);
    
    // Hearing Aid
    setHacEnabled(bool enabled) generates (Result retval);
    getHacEnabled() generates (Result retval, bool enabled);
    
    // 语音音量
    setVoiceVolume(float volume) generates (Result retval);
};
```

### 2.4 IStreamOut (输出流)

```hal
// hardware/interfaces/audio/6.0/IStreamOut.hal
package android.hardware.audio@6.0;

interface IStreamOut {
    // 基本属性
    getFrameSize() generates (uint64_t size);
    getFrameCount() generates (uint64_t count);
    getBufferSize() generates (uint64_t size);
    getSampleRate() generates (uint32_t rate);
    getSupportedSampleRates()
        generates (Result retval, vec<uint32_t> rates);
    setSampleRate(uint32_t rate) generates (Result retval);
    getChannelMask() generates (AudioChannelMask mask);
    getSupportedChannelMasks()
        generates (Result retval, vec<AudioChannelMask> masks);
    setChannelMask(AudioChannelMask mask) generates (Result retval);
    getFormat() generates (AudioFormat format);
    getSupportedFormats()
        generates (Result retval, vec<AudioFormat> formats);
    setFormat(AudioFormat format) generates (Result retval);
    getAudioProperties()
        generates (uint32_t sampleRate, AudioChannelMask mask, AudioFormat format);
    
    // 音效
    addEffect(uint64_t effectId) generates (Result retval);
    removeEffect(uint64_t effectId) generates (Result retval);
    
    // 设备
    getDevice() generates (DeviceAddress device);
    setDevice(DeviceAddress device) generates (Result retval);
    setConnectedState(DeviceAddress device, bool connected)
        generates (Result retval);
    
    // 数据传输 (非阻塞模式)
    prepareForWriting(uint32_t frameSize, uint32_t framesCount)
        generates (
            Result retval,
            MQDescriptor<uint8_t, SynchronizedReadWrite> dataMQ,
            MQDescriptor<uint32_t, SynchronizedReadWrite> statusMQ
        );
    
    // 回调
    setCallback(IStreamOutCallback callback) generates (Result retval);
    clearCallback() generates (Result retval);
    
    // 流控制
    standby() generates (Result retval);
    pause() generates (Result retval);
    resume() generates (Result retval);
    flush() generates (Result retval);
    drain(AudioDrain type) generates (Result retval);
    
    // 延迟与位置
    getLatency() generates (Result retval, uint32_t latencyMs);
    getRenderPosition() generates (Result retval, uint32_t frames);
    getPresentationPosition()
        generates (Result retval, uint64_t frames, TimeSpec timestamp);
    
    // 音量
    setVolume(float left, float right) generates (Result retval);
    
    // 参数
    getParameters(vec<string> keys)
        generates (Result retval, vec<ParameterValue> parameters);
    setParameters(vec<ParameterValue> parameters) generates (Result retval);
    
    // mmap
    createMmapBuffer(int32_t minSizeFrames)
        generates (Result retval, MmapBufferInfo info);
    getMmapPosition()
        generates (Result retval, MmapPosition position);
    
    // 关闭
    close() generates (Result retval);
};
```

### 2.5 IStreamOutCallback (回调接口)

```hal
// hardware/interfaces/audio/6.0/IStreamOutCallback.hal
package android.hardware.audio@6.0;

interface IStreamOutCallback {
    oneway onWriteReady();
    oneway onDrainReady();
    oneway onError();
};
```

### 2.6 IStreamIn (输入流)

```hal
// hardware/interfaces/audio/6.0/IStreamIn.hal
package android.hardware.audio@6.0;

interface IStreamIn {
    // 基本属性 (同 IStreamOut)
    getFrameSize() generates (uint64_t size);
    getFrameCount() generates (uint64_t count);
    // ... 其他属性方法
    
    // 数据传输 (非阻塞模式)
    prepareForReading(uint32_t frameSize, uint32_t framesCount)
        generates (
            Result retval,
            MQDescriptor<uint8_t, SynchronizedReadWrite> dataMQ,
            MQDescriptor<uint32_t, SynchronizedReadWrite> statusMQ
        );
    
    // 增益
    setGain(float gain) generates (Result retval);
    
    // 音频源
    getAudioSource() generates (AudioSource source);
    setAudioSource(AudioSource source) generates (Result retval);
    
    // 位置
    getCapturePosition()
        generates (Result retval, uint64_t frames, TimeSpec timestamp);
    getInputFramesLost() generates (uint32_t framesLost);
    
    // 流控制
    standby() generates (Result retval);
    
    // 关闭
    close() generates (Result retval);
};
```

---

## 3. HIDL 模式

### 3.1 Passthrough 模式

```
┌─────────────────────────────────────────────────────────────────┐
│  AudioFlinger 进程                                              │
│  ┌─────────────────┐                                            │
│  │ IDevicesFactory │ ← dlopen 直接加载 .so                       │
│  │ (HIDL Proxy)    │                                            │
│  └────────┬────────┘                                            │
│           │ 函数调用                                             │
│  ┌────────▼────────┐                                            │
│  │ HAL Implementation │ ← audio.primary.default.so              │
│  │ (HIDL_FETCH)       │                                         │
│  └───────────────────┘                                          │
└─────────────────────────────────────────────────────────────────┘

特点:
- HAL 在 AudioFlinger 进程中加载
- 无 Binder IPC 开销
- 适合性能敏感场景
```

### 3.2 Binderized 模式

```
┌──────────────────────┐      Binder IPC     ┌──────────────────────┐
│  AudioFlinger 进程   │  ←───────────────→  │  HAL 服务进程        │
│  ┌────────────────┐  │                     │  ┌────────────────┐  │
│  │ IDevicesFactory│  │                     │  │ IDevicesFactory│  │
│  │ (BpHwDevice)   │  │                     │  │ (BnHwDevice)   │  │
│  └────────────────┘  │                     │  └────────────────┘  │
└──────────────────────┘                     └──────────────────────┘

特点:
- HAL 在独立进程中运行
- 通过 Binder IPC 通信
- 适合模块化部署
```

### 3.3 模式选择配置

```mk
# Passthrough 模式 (推荐)
PRODUCT_PACKAGES += \
    android.hardware.audio@6.0-impl \
    android.hardware.audio@6.0-service

# Binderized 模式
PRODUCT_PACKAGES += \
    android.hardware.audio@6.0-service
```

---

## 4. HAL 实现

### 4.1 DevicesFactory 实现

```cpp
// DevicesFactory.cpp
#include <android/hardware/audio/6.0/IDevicesFactory.h>
#include <hidl/HidlTransportSupport.h>

using android::hardware::audio::V6_0::IDevicesFactory;
using android::hardware::audio::V6_0::IPrimaryDevice;
using android::hardware::audio::V6_0::Result;
using android::hardware::Return;
using android::hardware::Void;

class DevicesFactory : public IDevicesFactory {
public:
    // v6.0 接口
    Return<void> openDevice(const hidl_string& moduleName,
                            openDevice_cb _hidl_cb) override {
        if (moduleName == AUDIO_HARDWARE_MODULE_ID_PRIMARY) {
            return openPrimaryDevice(_hidl_cb);
        }
        return openDevice<Device>(moduleName.c_str(), _hidl_cb);
    }
    
    Return<void> openPrimaryDevice(openPrimaryDevice_cb _hidl_cb) override {
        return openDevice<PrimaryDevice>(AUDIO_HARDWARE_MODULE_ID_PRIMARY, _hidl_cb);
    }

private:
    template <class DeviceShim>
    Return<void> openDevice(const char* moduleName, 
                            openDevice_cb _hidl_cb) {
        audio_hw_device_t* halDevice = nullptr;
        Result result = Result::NOT_INITIALIZED;
        sp<DeviceShim> device;
        
        // 加载 Legacy HAL
        int status = loadAudioInterface(moduleName, &halDevice);
        if (status == OK) {
            device = new DeviceShim(halDevice);
            result = Result::OK;
        }
        
        _hidl_cb(result, device);
        return Void();
    }
    
    int loadAudioInterface(const char* moduleName, 
                           audio_hw_device_t** dev) {
        const hw_module_t* module = nullptr;
        int rc = hw_get_module_by_class(AUDIO_HARDWARE_MODULE_ID, 
                                        moduleName, &module);
        if (rc) {
            ALOGE("Couldn't load audio hw module %s.%s (%s)",
                  AUDIO_HARDWARE_MODULE_ID, moduleName, strerror(-rc));
            return rc;
        }
        
        rc = audio_hw_device_open(module, dev);
        if (rc) {
            ALOGE("Couldn't open audio hw device in %s.%s (%s)",
                  AUDIO_HARDWARE_MODULE_ID, moduleName, strerror(-rc));
            return rc;
        }
        
        return OK;
    }
};

// Passthrough 入口函数
extern "C" IDevicesFactory* HIDL_FETCH_IDevicesFactory(const char* name) {
    return (strcmp(name, "default") == 0) ? new DevicesFactory() : nullptr;
}
```

### 4.2 Device 实现

```cpp
// Device.cpp
#include <android/hardware/audio/6.0/IDevice.h>

class Device : public IDevice {
public:
    Device(audio_hw_device_t* device) : mDevice(device) {}
    ~Device() { close(); }
    
    Return<Result> initCheck() override {
        return mDevice->init_check(mDevice);
    }
    
    Return<Result> setMasterVolume(float volume) override {
        return mDevice->set_master_volume(mDevice, volume);
    }
    
    Return<void> getMasterVolume(getMasterVolume_cb _hidl_cb) override {
        float volume = 0.0f;
        int result = mDevice->get_master_volume(mDevice, &volume);
        _hidl_cb(mapResult(result), volume);
        return Void();
    }
    
    Return<Result> setMicMute(bool muted) override {
        return mDevice->set_mic_mute(mDevice, muted);
    }
    
    Return<void> openOutputStream(
            int32_t ioHandle,
            const DeviceAddress& device,
            const AudioConfig& config,
            hidl_bitfield<AudioOutputFlag> flags,
            AudioSource source,
            openOutputStream_cb _hidl_cb) override {
        
        audio_stream_out_t* outStream = nullptr;
        audio_config_t halConfig = {};
        convertToHalConfig(config, &halConfig);
        
        int result = mDevice->open_output_stream(
            mDevice, ioHandle, 
            convertDeviceAddress(device),
            &halConfig, 
            static_cast<audio_output_flags_t>(flags),
            &outStream
        );
        
        if (result == OK && outStream != nullptr) {
            sp<IStreamOut> stream = new StreamOut(outStream);
            AudioConfig suggestedConfig;
            convertFromHalConfig(&halConfig, &suggestedConfig);
            _hidl_cb(Result::OK, stream, suggestedConfig);
        } else {
            _hidl_cb(mapResult(result), nullptr, config);
        }
        
        return Void();
    }
    
    Return<void> openInputStream(
            int32_t ioHandle,
            const DeviceAddress& device,
            const AudioConfig& config,
            hidl_bitfield<AudioInputFlag> flags,
            AudioSource source,
            const hidl_string& devices,
            openInputStream_cb _hidl_cb) override {
        // 类似实现...
    }
    
    Return<Result> setParameters(const hidl_vec<ParameterValue>& parameters) override {
        String8 pairs = parameterPairsToKeyValueString(parameters);
        return mDevice->set_parameters(mDevice, pairs.c_str());
    }
    
    Return<void> close() override {
        if (mDevice) {
            audio_hw_device_close(mDevice);
            mDevice = nullptr;
        }
        return Result::OK;
    }

private:
    audio_hw_device_t* mDevice;
    
    Result mapResult(int status) {
        switch (status) {
            case 0: return Result::OK;
            case -EINVAL: return Result::INVALID_ARGUMENTS;
            case -ENODEV: return Result::NOT_INITIALIZED;
            case -ENOSYS: return Result::NOT_SUPPORTED;
            default: return Result::INVALID_STATE;
        }
    }
};
```

### 4.3 StreamOut 实现

```cpp
// StreamOut.cpp
#include <android/hardware/audio/6.0/IStreamOut.h>

class StreamOut : public IStreamOut {
public:
    StreamOut(audio_stream_out_t* stream) : mStream(stream) {}
    ~StreamOut() { close(); }
    
    Return<uint64_t> getFrameSize() override {
        return audio_stream_out_frame_size(mStream);
    }
    
    Return<uint64_t> getFrameCount() override {
        return mStream->common.get_buffer_size(&mStream->common) / 
               audio_stream_out_frame_size(mStream);
    }
    
    Return<uint32_t> getSampleRate() override {
        return mStream->common.get_sample_rate(&mStream->common);
    }
    
    Return<Result> setSampleRate(uint32_t rate) override {
        return mStream->common.set_sample_rate(&mStream->common, rate);
    }
    
    Return<void> prepareForWriting(
            uint32_t frameSize,
            uint32_t framesCount,
            prepareForWriting_cb _hidl_cb) override {
        
        // 创建 FMQ
        size_t bufferSize = frameSize * framesCount;
        mFmq = std::make_unique<AidlMessageQueue>(
            bufferSize, true /* configureEventFlagWord */
        );
        
        if (!mFmq->isValid()) {
            _hidl_cb(Result::NOT_INITIALIZED, nullptr, nullptr);
            return Void();
        }
        
        // 创建 EventFlag
        EventFlag::createEventFlag(mFmq->getEventFlagWord(), &mEventFlag);
        
        _hidl_cb(Result::OK, mFmq->dupeDescriptor(), mStatusFmq->dupeDescriptor());
        return Void();
    }
    
    Return<Result> setCallback(const sp<IStreamOutCallback>& callback) override {
        mCallback = callback;
        return Result::OK;
    }
    
    Return<Result> standby() override {
        return mStream->common.standby(&mStream->common);
    }
    
    Return<Result> pause() override {
        if (mStream->pause) {
            return mStream->pause(mStream);
        }
        return Result::NOT_SUPPORTED;
    }
    
    Return<Result> resume() override {
        if (mStream->resume) {
            return mStream->resume(mStream);
        }
        return Result::NOT_SUPPORTED;
    }
    
    Return<Result> flush() override {
        if (mStream->flush) {
            return mStream->flush(mStream);
        }
        return Result::NOT_SUPPORTED;
    }
    
    Return<void> getLatency(getLatency_cb _hidl_cb) override {
        uint32_t latency = mStream->get_latency(mStream);
        _hidl_cb(Result::OK, latency);
        return Void();
    }
    
    Return<Result> setVolume(float left, float right) override {
        if (mStream->set_volume) {
            return mStream->set_volume(mStream, left, right);
        }
        return Result::NOT_SUPPORTED;
    }
    
    Return<void> getPresentationPosition(
            getPresentationPosition_cb _hidl_cb) override {
        uint64_t frames = 0;
        struct timespec timestamp = {0, 0};
        
        int result = mStream->get_presentation_position(
            mStream, &frames, &timestamp
        );
        
        TimeSpec ts;
        ts.tvSec = timestamp.tv_sec;
        ts.tvNSec = timestamp.tv_nsec;
        
        _hidl_cb(mapResult(result), frames, ts);
        return Void();
    }
    
    Return<Result> close() override {
        if (mStream) {
            // 由 Device 负责关闭
            mStream = nullptr;
        }
        return Result::OK;
    }

private:
    audio_stream_out_t* mStream;
    sp<IStreamOutCallback> mCallback;
    std::unique_ptr<AidlMessageQueue> mFmq;
    android::EventFlag* mEventFlag = nullptr;
};
```

---

## 5. 服务注册

### 5.1 Passthrough 服务注册

```cpp
// service.cpp (Passthrough)
#include <android/hardware/audio/6.0/IDevicesFactory.h>
#include <hidl/LegacySupport.h>

using android::hardware::audio::V6_0::IDevicesFactory;
using android::hardware::registerPassthroughServiceImplementation;

int main() {
    // 注册 Passthrough 服务
    auto status = registerPassthroughServiceImplementation<IDevicesFactory>();
    if (status != android::OK) {
        ALOGE("Failed to register IDevicesFactory: %d", status);
        return 1;
    }
    
    // 注册音效服务
    registerPassthroughServiceImplementation<IEffectsFactory>();
    
    // 加入线程池
    android::hardware::joinRpcThreadpool();
    return 0;
}
```

### 5.2 Binderized 服务注册

```cpp
// service.cpp (Binderized)
#include <android/hardware/audio/6.0/IDevicesFactory.h>
#include <hidl/HidlTransportSupport.h>

using android::hardware::audio::V6_0::IDevicesFactory;
using android::hardware::configureRpcThreadpool;
using android::hardware::joinRpcThreadpool;

int main() {
    // 配置线程池
    configureRpcThreadpool(4, true /* callerWillJoin */);
    
    // 创建并注册服务
    android::sp<IDevicesFactory> service = new DevicesFactory();
    auto status = service->registerAsService("default");
    if (status != android::OK) {
        ALOGE("Failed to register IDevicesFactory: %d", status);
        return 1;
    }
    
    // 加入线程池
    joinRpcThreadpool();
    return 0;
}
```

### 5.3 Legacy HAL 入口

```c
// audio_hw.c (Legacy HAL 实现)
#include <hardware/audio.h>

static struct hw_module_methods_t hal_module_methods = {
    .open = adev_open,
};

struct audio_module HAL_MODULE_INFO_SYM = {
    .common = {
        .tag = HARDWARE_MODULE_TAG,
        .module_api_version = AUDIO_MODULE_API_VERSION_0_1,
        .hal_api_version = HARDWARE_HAL_API_VERSION,
        .id = AUDIO_HARDWARE_MODULE_ID,
        .name = "QCOM Audio HAL",
        .author = "The Linux Foundation",
        .methods = &hal_module_methods,
    },
};

static int adev_open(const hw_module_t* module, const char* name,
                     hw_device_t** device) {
    struct audio_device* adev = calloc(1, sizeof(struct audio_device));
    
    adev->device.common.tag = HARDWARE_DEVICE_TAG;
    adev->device.common.version = AUDIO_DEVICE_API_VERSION_3_0;
    adev->device.common.module = (hw_module_t*)module;
    adev->device.common.close = adev_close;
    
    // 绑定函数指针
    adev->device.init_check = adev_init_check;
    adev->device.set_master_volume = adev_set_master_volume;
    adev->device.set_mic_mute = adev_set_mic_mute;
    adev->device.open_output_stream = adev_open_output_stream;
    adev->device.open_input_stream = adev_open_input_stream;
    // ... 更多函数
    
    *device = &adev->device.common;
    return 0;
}
```

---

## 6. Android.bp 配置

### 6.1 HIDL 接口定义

```bp
// hardware/interfaces/audio/6.0/Android.bp
hidl_interface {
    name: "android.hardware.audio@6.0",
    root: "android.hardware",
    srcs: [
        "types.hal",
        "IDevice.hal",
        "IDevicesFactory.hal",
        "IPrimaryDevice.hal",
        "IStream.hal",
        "IStreamIn.hal",
        "IStreamOut.hal",
        "IStreamOutCallback.hal",
    ],
    interfaces: [
        "android.hardware.audio.common@6.0",
    ],
    types: [
        "AudioChannelMask",
        "AudioConfig",
        "AudioDevice",
        "AudioFormat",
        "AudioInputFlag",
        "AudioOutputFlag",
        "AudioPatchHandle",
        "AudioPort",
        "AudioPortConfig",
        "Result",
    ],
}
```

### 6.2 HAL 实现模块

```bp
// HAL 实现
cc_library_shared {
    name: "android.hardware.audio@6.0-impl",
    vendor: true,
    srcs: [
        "DevicesFactory.cpp",
        "Device.cpp",
        "StreamOut.cpp",
        "StreamIn.cpp",
        "Conversions.cpp",
    ],
    shared_libs: [
        "libbase",
        "libcutils",
        "libfmq",
        "libhidlbase",
        "libhidlmemory",
        "liblog",
        "libutils",
        "android.hardware.audio@6.0",
        "android.hardware.audio.common@6.0",
    ],
    header_libs: [
        "libaudiohal_headers",
    ],
}

// 服务
cc_binary {
    name: "android.hardware.audio@6.0-service",
    vendor: true,
    srcs: ["service.cpp"],
    shared_libs: [
        "libbase",
        "libhidlbase",
        "liblog",
        "libutils",
        "android.hardware.audio@6.0",
    ],
    init_rc: ["android.hardware.audio@6.0-service.rc"],
    vintf_fragments: ["android.hardware.audio@6.0-service.xml"],
}
```

---

## 7. VINTF 配置

### 7.1 服务 RC 文件

```rc
# android.hardware.audio@6.0-service.rc
service vendor.audio-hal /vendor/bin/hw/android.hardware.audio@6.0-service
    class hal
    user audioserver
    group audio system
    capabilities BLOCK_SUSPEND
    onrestart restart audioserver
```

### 7.2 VINTF Manifest

```xml
<!-- android.hardware.audio@6.0-service.xml -->
<manifest version="1.0" type="device">
    <hal format="hidl">
        <name>android.hardware.audio</name>
        <transport>hwbinder</transport>
        <version>6.0</version>
        <interface>
            <name>IDevicesFactory</name>
            <instance>default</instance>
        </interface>
    </hal>
    <hal format="hidl">
        <name>android.hardware.audio.effect</name>
        <transport>hwbinder</transport>
        <version>6.0</version>
        <interface>
            <name>IEffectsFactory</name>
            <instance>default</instance>
        </interface>
    </hal>
</manifest>
```

### 7.3 兼容性矩阵

```xml
<!-- compatibility_matrix.xml -->
<hal format="hidl" optional="false">
    <name>android.hardware.audio</name>
    <version>5.0-6.0</version>
    <interface>
        <name>IDevicesFactory</name>
        <instance>default</instance>
    </interface>
</hal>
<hal format="hidl" optional="false">
    <name>android.hardware.audio.effect</name>
    <version>5.0-6.0</version>
    <interface>
        <name>IEffectsFactory</name>
        <instance>default</instance>
    </interface>
</hal>
```

---

## 8. 错误处理

### 8.1 Result 枚举

```cpp
// types.hal
enum Result : int32_t {
    OK = 0,
    NOT_INITIALIZED = 1,
    INVALID_ARGUMENTS = 2,
    INVALID_STATE = 3,
    NOT_SUPPORTED = 4,
    INVALID_HANDLE = 5,
    // v5.0+
    TOO_MANY_OPEN_STREAMS = 6,
    NO_MEMORY = 7,
    // v6.0+
    INTERNAL_ERROR = 8,
};
```

### 8.2 错误映射

```cpp
Result mapError(int err) {
    switch (err) {
        case 0:
            return Result::OK;
        case -EINVAL:
            return Result::INVALID_ARGUMENTS;
        case -ENODEV:
            return Result::NOT_INITIALIZED;
        case -ENOSYS:
            return Result::NOT_SUPPORTED;
        case -ENOMEM:
            return Result::NO_MEMORY;
        case -EBUSY:
            return Result::TOO_MANY_OPEN_STREAMS;
        default:
            return Result::INTERNAL_ERROR;
    }
}
```

---

## 9. 调试

### 9.1 日志

```cpp
#include <android-base/logging.h>

#define LOG_TAG "AudioHAL"

ALOGV("Verbose log");
ALOGD("Debug log");
ALOGI("Info log");
ALOGW("Warning log");
ALOGE("Error log: %s", strerror(errno));
```

### 9.2 dumpsys

```bash
# 查看 HIDL 服务
lshal | grep audio

# 查看音频状态
dumpsys audio

# 查看 HAL 详情
dumpsys audio | grep -A 50 "HAL"

# 检查服务状态
service list | grep audio
```

### 9.3 常见问题定位

```bash
# 1. HIDL 服务未启动
adb shell lshal | grep audio
adb logcat | grep "audio.*service"

# 2. HAL 加载失败
adb logcat | grep -E "hw_get_module|audio_hw_device"

# 3. VINTF 兼容性问题
adb shell dumpsys vintf | grep audio

# 4. 流打开失败
adb logcat | grep -E "openOutputStream|openInputStream"
```

---

## 📌 总结

| 类别 | 规则 |
|------|------|
| **接口** | IDevicesFactory / IDevice / IPrimaryDevice / IStreamOut / IStreamIn |
| **基类** | IDevicesFactory / IDevice / IStreamOut |
| **模式** | Passthrough (推荐) / Binderized |
| **入口** | HIDL_FETCH_IDevicesFactory / HAL_MODULE_INFO_SYM |
| **数据传输** | FMQ (非阻塞模式) / 直接调用 (阻塞模式) |
| **回调** | IStreamOutCallback (onWriteReady / onDrainReady / onError) |
| **错误** | Result 枚举 |
| **配置** | hidl_interface + vintf_fragments |
| **VINTF** | manifest.xml + compatibility_matrix |
| **调试** | lshal / dumpsys audio / logcat |
