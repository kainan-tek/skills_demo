---
name: "audio-control-hal"
description: "AAOS AudioControl HAL 开发规范，处理音频焦点、音量控制、静音管理、Ducking 策略与车辆音频交互"
version: "1.0.0"
triggers: ["AudioControl", "IAudioControl", "CarAudioService", "AudioFocus", "Ducking", "Muting", "IFocusListener", "AudioGainConfig", "vehicle audio", "car audio", "audio zones", "balance", "fade"]
---

> 参考来源：Android AOSP hardware/interfaces/automotive/audiocontrol

# 🚗 AAOS AudioControl HAL 开发规范

---

## 1. AudioControl HAL 概述

### 1.1 简介

```
AudioControl HAL 是 AAOS 车载音频的核心组件
- 作用: 车辆音频引擎与 Android 音频框架的桥梁
- 功能: 音频焦点管理、音量控制、静音、Ducking、平衡/衰减
- 位置: hardware/interfaces/automotive/audiocontrol/
- 版本: HIDL v1.0/v2.0 → AIDL (Android 13+)
```

### 1.2 架构位置

```
┌─────────────────────────────────────────────────────────────────┐
│                    AAOS Audio Architecture                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  CarAudioService (Java)                                    ││
│  │  - CarAudioZones (多音区管理)                              ││
│  │  - CarAudioFocus (焦点管理)                                ││
│  │  - CarVolumeCallback (音量回调)                            ││
│  └─────────────────────────────────────────────────────────────┘│
│                          ↓ AIDL Binder                          │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  AudioControl HAL (AIDL)                                   ││
│  │  - IAudioControl (主接口)                                  ││
│  │  - IFocusListener (焦点监听)                               ││
│  │  - AudioGainCallback (增益回调)                            ││
│  └─────────────────────────────────────────────────────────────┘│
│                          ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Vehicle Audio System                                      ││
│  │  - 功放控制                                                ││
│  │  - 音频路由                                                ││
│  │  - DSP 处理                                                ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 源码路径

```
hardware/interfaces/automotive/audiocontrol/
├── aidl/                          # AIDL 接口 (Android 13+)
│   └── android/hardware/automotive/audiocontrol/
│       ├── IAudioControl.aidl     # 主接口
│       ├── IFocusListener.aidl    # 焦点监听
│       ├── IAudioGainCallback.aidl # 增益回调
│       ├── AudioFocusChange.aidl  # 焦点变化类型
│       ├── DuckingInfo.aidl       # Ducking 信息
│       ├── MutingInfo.aidl        # 静音信息
│       ├── AudioGainConfigInfo.aidl # 增益配置
│       └── Reasons.aidl           # 原因枚举
├── 1.0/                           # HIDL v1.0
│   └── IAudioControl.hal
├── 2.0/                           # HIDL v2.0
│   └── IAudioControl.hal
└── default/                       # 默认实现
```

---

## 2. AIDL 接口定义

### 2.1 IAudioControl (主接口)

```aidl
package android.hardware.automotive.audiocontrol;

interface IAudioControl {
    // 注册焦点监听器
    void registerFocusListener(in IFocusListener listener);
    
    // 设置音量组音量
    void setGroupVolume(int groupId, int volumeIndex);
    
    // 获取音量组音量
    int getGroupVolume(int groupId);
    
    // 设置音量组静音
    void setGroupMute(int groupId, boolean mute);
    
    // 获取音量组静音状态
    boolean isGroupMuted(int groupId);
    
    // 设置左右平衡 (-1.0 左 ~ 1.0 右)
    void setBalanceTowardRight(float value);
    
    // 设置前后衰减 (-1.0 前 ~ 1.0 后)
    void setFadeTowardFront(float value);
    
    // 注册增益回调
    void registerGainCallback(in IAudioGainCallback callback);
    
    // 请求音频焦点
    void onAudioFocusChange(in String usage, int zoneId, 
                            in AudioFocusChange focusChange);
    
    // 设置 Ducking
    void onDevicesToDuckChange(in DuckingInfo[] duckingInfos);
    
    // 设置静音
    void onDevicesToMuteChange(in MutingInfo[] mutingInfos);
    
    // 获取 Bus 映射
    int getBusForContext(int contextNumber);
}
```

### 2.2 IFocusListener (焦点监听)

```aidl
package android.hardware.automotive.audiocontrol;

oneway interface IFocusListener {
    // 请求音频焦点
    void requestAudioFocus(in String usage, int zoneId, 
                           in AudioFocusChange focusGain);
    
    // 放弃音频焦点
    void abandonAudioFocus(in String usage, int zoneId);
}
```

### 2.3 AudioFocusChange (焦点变化类型)

```aidl
package android.hardware.automotive.audiocontrol;

@Backing(type="int")
enum AudioFocusChange {
    GAIN = 1,              // 获得焦点
    GAIN_TRANSIENT = 2,    // 获得临时焦点
    GAIN_TRANSIENT_MAY_DUCK = 3, // 获得临时焦点 + Ducking
    LOSS = -1,             // 失去焦点
    LOSS_TRANSIENT = -2,   // 临时失去焦点
    LOSS_TRANSIENT_CAN_DUCK = -3, // 临时失去焦点 + 可 Duck
}
```

### 2.4 DuckingInfo (Ducking 信息)

```aidl
package android.hardware.automotive.audiocontrol;

parcelable DuckingInfo {
    int zoneId;                    // 音区 ID
    String[] usagesHoldingFocus;   // 持有焦点的 Usage
    float duckingFactor;           // Ducking 系数 (0.0 ~ 1.0)
    String[] deviceAddresses;      // 设备地址列表
}
```

### 2.5 MutingInfo (静音信息)

```aidl
package android.hardware.automotive.audiocontrol;

parcelable MutingInfo {
    int zoneId;                    // 音区 ID
    String[] deviceAddresses;      // 需要静音的设备
    String[] unmutedDeviceAddresses; // 不静音的设备
}
```

---

## 3. HAL 实现

### 3.1 继承 BnAudioControl

```cpp
#include <aidl/android/hardware/automotive/audiocontrol/BnAudioControl.h>

namespace aidl::android::hardware::automotive::audiocontrol {

class AudioControl : public BnAudioControl {
public:
    AudioControl();
    ~AudioControl() override;
    
    // 焦点管理
    ndk::ScopedAStatus registerFocusListener(
            const std::shared_ptr<IFocusListener>& listener) override;
    ndk::ScopedAStatus onAudioFocusChange(
            const std::string& usage, int32_t zoneId,
            AudioFocusChange focusChange) override;
    
    // 音量控制
    ndk::ScopedAStatus setGroupVolume(int32_t groupId, 
                                       int32_t volumeIndex) override;
    ndk::ScopedAStatus getGroupVolume(int32_t groupId, 
                                       int32_t* volumeIndex) override;
    ndk::ScopedAStatus setGroupMute(int32_t groupId, bool mute) override;
    ndk::ScopedAStatus isGroupMuted(int32_t groupId, bool* muted) override;
    
    // 平衡/衰减
    ndk::ScopedAStatus setBalanceTowardRight(float value) override;
    ndk::ScopedAStatus setFadeTowardFront(float value) override;
    
    // Ducking/Muting
    ndk::ScopedAStatus onDevicesToDuckChange(
            const std::vector<DuckingInfo>& duckingInfos) override;
    ndk::ScopedAStatus onDevicesToMuteChange(
            const std::vector<MutingInfo>& mutingInfos) override;
    
    // Bus 映射
    ndk::ScopedAStatus getBusForContext(int32_t contextNumber, 
                                         int32_t* busNumber) override;
    
    // 增益回调
    ndk::ScopedAStatus registerGainCallback(
            const std::shared_ptr<IAudioGainCallback>& callback) override;

private:
    std::shared_ptr<IFocusListener> mFocusListener;
    std::shared_ptr<IAudioGainCallback> mGainCallback;
    std::mutex mLock;
    std::unordered_map<int32_t, int32_t> mGroupVolumes;
    std::unordered_map<int32_t, bool> mGroupMutes;
    float mBalance = 0.0f;
    float mFade = 0.0f;
};

}  // namespace aidl::android::hardware::automotive::audiocontrol
```

### 3.2 焦点管理实现

```cpp
ndk::ScopedAStatus AudioControl::registerFocusListener(
        const std::shared_ptr<IFocusListener>& listener) {
    std::lock_guard<std::mutex> lock(mLock);
    mFocusListener = listener;
    return ndk::ScopedAStatus::ok();
}

ndk::ScopedAStatus AudioControl::onAudioFocusChange(
        const std::string& usage, int32_t zoneId,
        AudioFocusChange focusChange) {
    ALOGD("AudioFocusChange: usage=%s, zone=%d, change=%d",
          usage.c_str(), zoneId, static_cast<int>(focusChange));
    
    switch (focusChange) {
        case AudioFocusChange::GAIN:
            // 恢复正常播放
            handleFocusGain(usage, zoneId);
            break;
            
        case AudioFocusChange::GAIN_TRANSIENT_MAY_DUCK:
            // Ducking 模式
            handleFocusGainWithDuck(usage, zoneId);
            break;
            
        case AudioFocusChange::LOSS:
            // 停止播放
            handleFocusLoss(usage, zoneId);
            break;
            
        case AudioFocusChange::LOSS_TRANSIENT_CAN_DUCK:
            // 可以 Duck
            handleFocusLossWithDuck(usage, zoneId);
            break;
            
        default:
            break;
    }
    
    return ndk::ScopedAStatus::ok();
}

// 请求焦点 (HAL -> CarAudioService)
void AudioControl::requestFocusFromCar(const std::string& usage, 
                                        int32_t zoneId,
                                        AudioFocusChange gainType) {
    if (mFocusListener) {
        mFocusListener->requestAudioFocus(usage, zoneId, gainType);
    }
}

// 放弃焦点 (HAL -> CarAudioService)
void AudioControl::abandonFocusFromCar(const std::string& usage, 
                                        int32_t zoneId) {
    if (mFocusListener) {
        mFocusListener->abandonAudioFocus(usage, zoneId);
    }
}
```

### 3.3 音量控制实现

```cpp
ndk::ScopedAStatus AudioControl::setGroupVolume(int32_t groupId, 
                                                 int32_t volumeIndex) {
    std::lock_guard<std::mutex> lock(mLock);
    
    ALOGD("setGroupVolume: groupId=%d, volume=%d", groupId, volumeIndex);
    
    // 保存音量值
    mGroupVolumes[groupId] = volumeIndex;
    
    // 设置到硬件
    setHardwareVolume(groupId, volumeIndex);
    
    return ndk::ScopedAStatus::ok();
}

ndk::ScopedAStatus AudioControl::getGroupVolume(int32_t groupId, 
                                                 int32_t* volumeIndex) {
    std::lock_guard<std::mutex> lock(mLock);
    
    auto it = mGroupVolumes.find(groupId);
    if (it != mGroupVolumes.end()) {
        *volumeIndex = it->second;
    } else {
        *volumeIndex = 0;  // 默认值
    }
    
    return ndk::ScopedAStatus::ok();
}

ndk::ScopedAStatus AudioControl::setGroupMute(int32_t groupId, bool mute) {
    std::lock_guard<std::mutex> lock(mLock);
    
    ALOGD("setGroupMute: groupId=%d, mute=%d", groupId, mute);
    
    mGroupMutes[groupId] = mute;
    setHardwareMute(groupId, mute);
    
    return ndk::ScopedAStatus::ok();
}
```

### 3.4 Ducking/Muting 实现

```cpp
ndk::ScopedAStatus AudioControl::onDevicesToDuckChange(
        const std::vector<DuckingInfo>& duckingInfos) {
    for (const auto& info : duckingInfos) {
        ALOGD("Ducking: zone=%d, factor=%.2f, devices=%zu",
              info.zoneId, info.duckingFactor, info.deviceAddresses.size());
        
        // 对每个设备应用 Ducking
        for (const auto& address : info.deviceAddresses) {
            applyDucking(address, info.duckingFactor);
        }
    }
    
    return ndk::ScopedAStatus::ok();
}

ndk::ScopedAStatus AudioControl::onDevicesToMuteChange(
        const std::vector<MutingInfo>& mutingInfos) {
    for (const auto& info : mutingInfos) {
        ALOGD("Muting: zone=%d, muted=%zu, unmuted=%zu",
              info.zoneId, 
              info.deviceAddresses.size(),
              info.unmutedDeviceAddresses.size());
        
        // 静音设备
        for (const auto& address : info.deviceAddresses) {
            setDeviceMute(address, true);
        }
        
        // 取消静音
        for (const auto& address : info.unmutedDeviceAddresses) {
            setDeviceMute(address, false);
        }
    }
    
    return ndk::ScopedAStatus::ok();
}
```

### 3.5 平衡/衰减实现

```cpp
ndk::ScopedAStatus AudioControl::setBalanceTowardRight(float value) {
    std::lock_guard<std::mutex> lock(mLock);
    
    // value 范围: -1.0 (完全左) ~ 1.0 (完全右)
    if (value < -1.0f || value > 1.0f) {
        return ndk::ScopedAStatus::fromServiceSpecificError(
            STATUS_BAD_VALUE);
    }
    
    mBalance = value;
    
    ALOGD("setBalanceTowardRight: %.2f", value);
    
    // 设置到 DSP 或功放
    setHardwareBalance(value);
    
    return ndk::ScopedAStatus::ok();
}

ndk::ScopedAStatus AudioControl::setFadeTowardFront(float value) {
    std::lock_guard<std::mutex> lock(mLock);
    
    // value 范围: -1.0 (完全后) ~ 1.0 (完全前)
    if (value < -1.0f || value > 1.0f) {
        return ndk::ScopedAStatus::fromServiceSpecificError(
            STATUS_BAD_VALUE);
    }
    
    mFade = value;
    
    ALOGD("setFadeTowardFront: %.2f", value);
    
    // 设置到 DSP 或功放
    setHardwareFade(value);
    
    return ndk::ScopedAStatus::ok();
}
```

---

## 4. 服务注册

### 4.1 main.cpp

```cpp
#include <aidl/android/hardware/automotive/audiocontrol/IAudioControl.h>
#include <android/binder_manager.h>
#include <android/binder_process.h>

using aidl::android::hardware::automotive::audiocontrol::IAudioControl;
using aidl::android::hardware::automotive::audiocontrol::AudioControl;

int main() {
    ABinderProcess_setThreadPoolMaxThreadCount(4);
    
    auto audioControl = ndk::SharedRefBase::make<AudioControl>();
    
    const std::string instance = 
        std::string(IAudioControl::descriptor) + "/default";
    
    binder_status_t status = AServiceManager_addService(
        audioControl->asBinder().get(),
        instance.c_str()
    );
    
    if (status != STATUS_OK) {
        ALOGE("Failed to register AudioControl service: %d", status);
        return 1;
    }
    
    ALOGI("AudioControl HAL service registered: %s", instance.c_str());
    
    ABinderProcess_joinThreadPool();
    return 0;
}
```

### 4.2 Android.bp

```bp
aidl_interface {
    name: "android.hardware.automotive.audiocontrol",
    vendor_available: true,
    srcs: ["android/hardware/automotive/audiocontrol/*.aidl"],
    imports: [
        "android.hardware.audio.common-V1",
    ],
    stability: "vintf",
    backend: {
        java: {
            sdk_version: "module_current",
        },
        cpp: {
            enabled: true,
        },
    },
}

cc_binary {
    name: "android.hardware.automotive.audiocontrol-service",
    vendor: true,
    srcs: [
        "main.cpp",
        "AudioControl.cpp",
    ],
    shared_libs: [
        "libbase",
        "libbinder_ndk",
        "liblog",
        "libutils",
        "android.hardware.automotive.audiocontrol-V1-ndk",
    ],
    init_rc: ["audiocontrol-default.rc"],
    vintf_fragments: ["audiocontrol-default.xml"],
}
```

### 4.3 RC 文件

```rc
# audiocontrol-default.rc
service vendor.audiocontrol-default /vendor/bin/hw/android.hardware.automotive.audiocontrol-service
    class hal
    user audioserver
    group audio system
    onrestart restart audioserver
```

### 4.4 VINTF Manifest

```xml
<!-- audiocontrol-default.xml -->
<manifest version="1.0" type="device">
    <hal format="aidl">
        <name>android.hardware.automotive.audiocontrol</name>
        <version>1</version>
        <fqname>IAudioControl/default</fqname>
    </hal>
</manifest>
```

---

## 5. 与 CarAudioService 交互

### 5.1 CarAudioService 调用流程

```java
// CarAudioService.java
public class CarAudioService extends ICarAudio.Stub {
    
    private IAudioControl mAudioControl;
    
    // 初始化
    private void initAudioControl() {
        String instanceName = IAudioControl.DESCRIPTOR + "/default";
        mAudioControl = IAudioControl.Stub.asInterface(
            ServiceManager.waitForService(instanceName)
        );
        
        // 注册焦点监听
        mAudioControl.registerFocusListener(mFocusListener);
    }
    
    // 设置音量
    public void setGroupVolume(int zoneId, int groupId, int index, int flags) {
        mAudioControl.setGroupVolume(groupId, index);
    }
    
    // 处理焦点变化
    private final IFocusListener mFocusListener = new IFocusListener.Stub() {
        @Override
        public void requestAudioFocus(String usage, int zoneId, 
                                       AudioFocusChange focusGain) {
            // 处理来自 HAL 的焦点请求
            handleFocusRequestFromHal(usage, zoneId, focusGain);
        }
        
        @Override
        public void abandonAudioFocus(String usage, int zoneId) {
            // 处理来自 HAL 的焦点放弃
            handleFocusAbandonFromHal(usage, zoneId);
        }
    };
}
```

### 5.2 焦点请求流程

```
┌─────────────────────────────────────────────────────────────────┐
│                     Audio Focus Flow                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  App 请求焦点                                                   │
│      │                                                          │
│      ↓                                                          │
│  ┌─────────────────┐                                            │
│  │ AudioManager    │                                            │
│  │ requestFocus()  │                                            │
│  └────────┬────────┘                                            │
│           ↓                                                     │
│  ┌─────────────────┐                                            │
│  │ CarAudioService │                                            │
│  │ (Java)          │                                            │
│  └────────┬────────┘                                            │
│           ↓ AIDL                                                │
│  ┌─────────────────┐                                            │
│  │ AudioControl HAL│                                            │
│  │ onAudioFocus    │                                            │
│  │ Change()        │                                            │
│  └────────┬────────┘                                            │
│           ↓                                                     │
│  ┌─────────────────┐                                            │
│  │ Vehicle Audio   │                                            │
│  │ System          │                                            │
│  └─────────────────┘                                            │
│                                                                 │
│  车辆请求焦点 (如导航、电话)                                    │
│      │                                                          │
│      ↓                                                          │
│  ┌─────────────────┐                                            │
│  │ AudioControl HAL│                                            │
│  │ requestAudio    │                                            │
│  │ Focus()         │                                            │
│  └────────┬────────┘                                            │
│           ↓ AIDL                                                │
│  ┌─────────────────┐                                            │
│  │ CarAudioService │                                            │
│  │ IFocusListener  │                                            │
│  └────────┬────────┘                                            │
│           ↓                                                     │
│  ┌─────────────────┐                                            │
│  │ App 处理焦点    │                                            │
│  │ 变化            │                                            │
│  └─────────────────┘                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. 调试

### 6.1 dumpsys

```bash
# 查看 AudioControl 服务
dumpsys android.hardware.automotive.audiocontrol.IAudioControl/default

# 查看 CarAudioService 状态
dumpsys car_service | grep -A 50 "CarAudio"

# 查看焦点状态
dumpsys car_service | grep -A 20 "AudioFocus"
```

### 6.2 日志

```bash
# AudioControl HAL 日志
adb logcat -s AudioControl

# CarAudioService 日志
adb logcat -s CarAudioService

# 焦点相关日志
adb logcat | grep -E "AudioFocus|FocusListener"
```

### 6.3 常见问题

```bash
# 1. 服务未启动
adb shell service list | grep audiocontrol

# 2. 焦点请求失败
adb logcat | grep -E "requestAudioFocus|onAudioFocusChange"

# 3. 音量设置无效
adb logcat | grep -E "setGroupVolume|getGroupVolume"

# 4. Ducking/Muting 问题
adb logcat | grep -E "Ducking|Muting"
```

---

## 📌 总结

| 类别 | 关键接口/结构 |
|------|--------------|
| **主接口** | IAudioControl |
| **焦点管理** | IFocusListener, AudioFocusChange |
| **音量控制** | setGroupVolume, setGroupMute |
| **Ducking** | DuckingInfo, onDevicesToDuckChange |
| **Muting** | MutingInfo, onDevicesToMuteChange |
| **平衡/衰减** | setBalanceTowardRight, setFadeTowardFront |
| **Bus 映射** | getBusForContext |
| **服务注册** | AServiceManager_addService |
| **调试** | dumpsys car_service, logcat |
