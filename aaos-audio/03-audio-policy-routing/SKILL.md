---
name: "audio-policy-routing"
description: "AudioPolicy 策略管理与路由配置，适用于 AudioPolicyService、AudioPolicyManager、设备选择与路由策略开发，包含 AAOS 车载音频"
version: "2.1.0"
triggers: ["AudioPolicy", "AudioPolicyService", "AudioPolicyManager", "routing", "AudioPatch", "AudioPort", "AudioMix", "StreamType", "AudioDevice", "getDeviceForStrategy", "getOutputForAttr", "VolumeCurves", "CarAudioService", "CarAudioZone", "CarAudioFocus", "car_audio_configuration", "AUDIO_DEVICE_OUT_BUS", "CarAudioContext"]
---

> 参考来源：Android AOSP AudioPolicy 源码

# 🎛️ AudioPolicy 策略管理与路由

---

## 1. AudioPolicy 架构

### 1.1 核心组件

| 组件 | 路径 | 职责 |
|------|------|------|
| `AudioPolicyService` | `services/audiopolicy/` | 策略服务 Binder 接口 |
| `AudioPolicyManager` | `services/audiopolicy/common/managerdefinitions/` | 策略管理核心实现 |
| `AudioPolicyConfig` | `services/audiopolicy/config/` | 配置文件解析 |
| `AudioPolicyEffects` | `services/audiopolicy/effects/` | 策略相关音效 |

### 1.2 层次结构

```
Java AudioService
        ↓ Binder IPC
AudioPolicyService (Native)
        ├── AudioPolicyManager (策略核心)
        │   ├── DeviceSelection
        │   ├── RoutingStrategy
        │   └── VolumeManagement
        ├── AudioPolicyClient (回调)
        └── AudioPolicyEffects
        ↓
AudioFlinger (执行层)
        ↓
Audio HAL
```

---

## 2. AudioPolicyManager 核心类

### 2.1 主要接口

```cpp
class AudioPolicyManager : public AudioPolicyInterface {
public:
    // 设备管理
    virtual audio_devices_t getDeviceForStrategy(
            audio_stream_type_t stream,
            audio_devices_t device,
            bool fromCache);
    
    virtual audio_devices_t getDeviceForInputSource(
            audio_source_t inputSource);
    
    // 输出管理
    virtual audio_io_handle_t getOutputForAttr(
            const audio_attributes_t* attr,
            audio_session_t session,
            audio_stream_type_t* stream,
            uid_t uid,
            const audio_config_t* config,
            audio_output_flags_t flags,
            audio_port_handle_t* selectedDeviceId,
            audio_port_handle_t* portId);
    
    // 输入管理
    virtual audio_io_handle_t getInputForAttr(
            const audio_attributes_t* attr,
            audio_session_t session,
            uid_t uid,
            const audio_config_t* config,
            audio_input_flags_t flags,
            audio_port_handle_t* selectedDeviceId,
            audio_port_handle_t* portId);
    
    // 设备连接
    virtual status_t setDeviceConnectionState(
            audio_devices_t device,
            audio_policy_dev_state_t state,
            const char* device_address,
            const char* device_name);
    
    // 音量管理
    virtual status_t setStreamVolumeIndex(
            audio_stream_type_t stream,
            int index,
            audio_devices_t device);
    
    virtual status_t getStreamVolumeIndex(
            audio_stream_type_t stream,
            int* index,
            audio_devices_t device);
    
    // Patch 管理
    virtual status_t createAudioPatch(
            const struct audio_patch* patch,
            audio_patch_handle_t* handle);
    
    virtual status_t releaseAudioPatch(
            audio_patch_handle_t handle);
    
    // AudioPort 管理
    virtual status_t getAudioPort(
            struct audio_port* port);
    
    virtual status_t setAudioPortConfig(
            const struct audio_port_config* config);
};
```

### 2.2 关键成员变量

```cpp
class AudioPolicyManager {
protected:
    // 设备列表
    DeviceVector mAvailableOutputDevices;   // 可用输出设备
    DeviceVector mAvailableInputDevices;    // 可用输入设备
    DeviceVector mAttachedOutputDevices;    // 固定输出设备
    
    // 输出/输入列表
    KeyedVector<audio_io_handle_t, sp<SwAudioOutputDescriptor>> mOutputs;
    KeyedVector<audio_io_handle_t, sp<AudioInputDescriptor>> mInputs;
    
    // 策略配置
    AudioPolicyConfig mConfig;              // 配置文件解析结果
    HwModuleCollection mHwModules;          // 硬件模块集合
    
    // 音量曲线
    VolumeCurvesCollection mVolumeCurves;   // 各 Stream 音量曲线
    
    // 音效
    AudioPolicyEffects* mAudioPolicyEffects;
    
    // Phone 状态
    audio_mode_t mPhoneState;               // 通话模式
    
    // 锁
    mutable Mutex mLock;
};
```

---

## 3. 路由策略 (Routing Strategy)

### 3.1 架构演进

```
┌─────────────────────────────────────────────────────────────────┐
│                     旧架构 (Legacy)                              │
│  Attributes -> Stream Type -> Strategy -> Device                │
│                                                                 │
│  例: USAGE_MEDIA -> STREAM_MUSIC -> STRATEGY_MEDIA -> Speaker   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     新架构 (Android 10+)                         │
│  Attributes -> Product Strategy -> Device                       │
│                                                                 │
│  例: USAGE_MEDIA -> STRATEGY_MEDIA -> Speaker                   │
│      (audio_attributes_t 是一等公民，Engine 直接映射)            │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Engine 策略决策模块

```cpp
// Engine 是 APM 的策略决策模块 (Strategy Decision Module)
// 路径: frameworks/av/services/audiopolicy/enginedefault/src/Engine.cpp

class Engine {
public:
    // Product Strategy: 将 Audio Attributes 映射到 Strategy
    virtual product_strategy_t getProductStrategyForAttributes(
            const audio_attributes_t& attr);
    
    // 设备选择算法
    virtual audio_devices_t getDevicesForStrategy(
            product_strategy_t strategy,
            bool fromCache);
    
    // 输入设备选择
    virtual audio_devices_t getDeviceForInputSource(
            audio_source_t inputSource);
    
private:
    // 策略配置
    ProductStrategyCollection mProductStrategies;
    
    // 强制配置 (Force Use)
    audio_policy_forced_cfg_t mForceUse[AUDIO_POLICY_FORCE_USE_CNT];
};
```

### 3.3 Product Strategy 定义

```cpp
// Product Strategy ID
typedef uint32_t product_strategy_t;

enum {
    PRODUCT_STRATEGY_MEDIA = 0,
    PRODUCT_STRATEGY_PHONE = 1,
    PRODUCT_STRATEGY_SONIFICATION = 2,
    PRODUCT_STRATEGY_ENFORCED_AUDIBLE = 3,
    PRODUCT_STRATEGY_DTMF = 4,
    PRODUCT_STRATEGY_TRANSMITTED_THROUGH_SPEAKER = 5,
    PRODUCT_STRATEGY_ACCESSIBILITY = 6,
    PRODUCT_STRATEGY_REROUTING = 7,
};

// Product Strategy 配置 (audio_policy_engine_configuration.xml)
struct ProductStrategy {
    std::string name;
    product_strategy_t id;
    std::vector<AttributesVector> attributesVectors;  // 支持的 attributes 列表
};
```

### 3.4 Attributes -> Strategy 映射 (新架构)

```cpp
// Engine::getProductStrategyForAttributes
// 直接从 audio_attributes_t 获取 Strategy，不再经过 Stream Type
product_strategy_t Engine::getProductStrategyForAttributes(
        const audio_attributes_t& attr) {
    
    // 1. 检查 flags
    if ((attr.flags & AUDIO_FLAG_AUDIBILITY_ENFORCED) != 0) {
        return PRODUCT_STRATEGY_ENFORCED_AUDIBLE;
    }
    
    // 2. 根据 usage 映射到 Strategy
    switch (attr.usage) {
        case AUDIO_USAGE_MEDIA:
        case AUDIO_USAGE_GAME:
        case AUDIO_USAGE_ASSISTANCE_ACCESSIBILITY:
        case AUDIO_USAGE_ASSISTANCE_NAVIGATION_GUIDANCE:
        case AUDIO_USAGE_ASSISTANCE_SONIFICATION:
            return PRODUCT_STRATEGY_MEDIA;
            
        case AUDIO_USAGE_VOICE_COMMUNICATION:
            return PRODUCT_STRATEGY_PHONE;
            
        case AUDIO_USAGE_VOICE_COMMUNICATION_SIGNALLING:
            return PRODUCT_STRATEGY_DTMF;
            
        case AUDIO_USAGE_ALARM:
        case AUDIO_USAGE_NOTIFICATION:
        case AUDIO_USAGE_NOTIFICATION_TELEPHONY_RINGTONE:
        case AUDIO_USAGE_NOTIFICATION_COMMUNICATION_REQUEST:
        case AUDIO_USAGE_NOTIFICATION_COMMUNICATION_INSTANT:
        case AUDIO_USAGE_NOTIFICATION_COMMUNICATION_DELAYED:
        case AUDIO_USAGE_NOTIFICATION_EVENT:
            return PRODUCT_STRATEGY_SONIFICATION;
            
        case AUDIO_USAGE_ASSISTANCE_SAFETY:
            return PRODUCT_STRATEGY_ENFORCED_AUDIBLE;
            
        default:
            return PRODUCT_STRATEGY_MEDIA;
    }
}
```

### 3.5 Legacy Strategy 定义 (兼容)

```cpp
// 旧版 routing_strategy (用于兼容旧代码)
enum routing_strategy {
    STRATEGY_MEDIA = 0,
    STRATEGY_PHONE = 1,
    STRATEGY_SONIFICATION = 2,
    STRATEGY_ENFORCED_AUDIBLE = 3,
    STRATEGY_DTMF = 4,
    STRATEGY_TRANSMITTED_THROUGH_SPEAKER = 5,
    STRATEGY_ACCESSIBILITY = 6,
    STRATEGY_REROUTING = 7,
    NUM_STRATEGIES
};

// Stream Type 到 Strategy 映射 (旧逻辑，仅用于兼容)
routing_strategy getStrategy(audio_stream_type_t stream) {
    switch (stream) {
        case AUDIO_STREAM_VOICE_CALL:
        case AUDIO_STREAM_BLUETOOTH_SCO:
            return STRATEGY_PHONE;
        case AUDIO_STREAM_RING:
        case AUDIO_STREAM_ALARM:
            return STRATEGY_SONIFICATION;
        case AUDIO_STREAM_NOTIFICATION:
        case AUDIO_STREAM_ENFORCED_AUDIBLE:
            return STRATEGY_ENFORCED_AUDIBLE;
        case AUDIO_STREAM_DTMF:
            return STRATEGY_DTMF;
        case AUDIO_STREAM_SYSTEM:
            return STRATEGY_SONIFICATION;
        case AUDIO_STREAM_TTS:
            return STRATEGY_TRANSMITTED_THROUGH_SPEAKER;
        case AUDIO_STREAM_ACCESSIBILITY:
            return STRATEGY_ACCESSIBILITY;
        case AUDIO_STREAM_MUSIC:
        case AUDIO_STREAM_ASSISTANT:
        default:
            return STRATEGY_MEDIA;
    }
}
```

### 3.6 设备选择逻辑

```cpp
audio_devices_t AudioPolicyManager::getDeviceForStrategy(
        routing_strategy strategy,
        bool fromCache) {
    
    if (fromCache) {
        return mStrategyDeviceCache[strategy];
    }
    
    audio_devices_t device = AUDIO_DEVICE_NONE;
    
    switch (strategy) {
        case STRATEGY_PHONE:
            // 电话策略：优先蓝牙 SCO，其次听筒
            if (isInCall()) {
                if (mAvailableOutputDevices.contains(AUDIO_DEVICE_OUT_BLUETOOTH_SCO)) {
                    device = AUDIO_DEVICE_OUT_BLUETOOTH_SCO;
                } else if (mAvailableOutputDevices.contains(AUDIO_DEVICE_OUT_EARPIECE)) {
                    device = AUDIO_DEVICE_OUT_EARPIECE;
                }
            }
            break;
            
        case STRATEGY_MEDIA:
            // 媒体策略：优先有线耳机，其次蓝牙 A2DP，最后扬声器
            if (mAvailableOutputDevices.contains(AUDIO_DEVICE_OUT_WIRED_HEADPHONE)) {
                device = AUDIO_DEVICE_OUT_WIRED_HEADPHONE;
            } else if (mAvailableOutputDevices.contains(AUDIO_DEVICE_OUT_BLUETOOTH_A2DP)) {
                device = AUDIO_DEVICE_OUT_BLUETOOTH_A2DP;
            } else if (mAvailableOutputDevices.contains(AUDIO_DEVICE_OUT_SPEAKER)) {
                device = AUDIO_DEVICE_OUT_SPEAKER;
            }
            break;
            
        case STRATEGY_SONIFICATION:
            // 铃声策略：优先耳机（如果已连接），否则扬声器
            if (mAvailableOutputDevices.contains(AUDIO_DEVICE_OUT_WIRED_HEADPHONE)) {
                device = AUDIO_DEVICE_OUT_WIRED_HEADPHONE;
            } else {
                device = AUDIO_DEVICE_OUT_SPEAKER;
            }
            break;
            
        default:
            device = AUDIO_DEVICE_OUT_SPEAKER;
            break;
    }
    
    mStrategyDeviceCache[strategy] = device;
    return device;
}
```

### 3.7 getOutputForAttr 流程 (新架构)

```cpp
// 新架构流程: Attributes -> Product Strategy -> Device
// Engine 是策略决策模块，APM 是执行层

audio_io_handle_t AudioPolicyManager::getOutputForAttr(
        const audio_attributes_t* attr,
        audio_session_t session,
        audio_stream_type_t* stream,
        uid_t uid,
        const audio_config_t* config,
        audio_output_flags_t flags,
        audio_port_handle_t* selectedDeviceId,
        audio_port_handle_t* portId) {
    
    // 1. 新架构: 直接从 attributes 获取 Product Strategy
    //    不再经过 Stream Type 中间层
    product_strategy_t strategy = mEngine->getProductStrategyForAttributes(*attr);
    
    // 2. (可选) 兼容旧代码: 反推 stream type
    if (stream != nullptr) {
        *stream = streamTypeFromAttributes(*attr);
    }
    
    // 3. Engine 决策: 获取目标设备
    //    getDevicesForStrategyInt 是决策核心
    audio_devices_t device = mEngine->getDevicesForStrategy(strategy, false);
    
    // 4. 查找合适的输出
    sp<SwAudioOutputDescriptor> outputDesc = nullptr;
    
    // 4.1 检查 AudioPolicyMix (动态策略优先)
    outputDesc = checkPolicyMix(*attr);
    
    // 4.2 检查是否需要专用输出 (DIRECT)
    if (outputDesc == nullptr && (flags & AUDIO_OUTPUT_FLAG_DIRECT)) {
        outputDesc = findDirectOutput(device, config);
    }
    
    // 4.3 检查是否需要 Fast Track
    if (outputDesc == nullptr && (flags & AUDIO_OUTPUT_FLAG_FAST)) {
        outputDesc = findFastOutput(device);
    }
    
    // 4.4 使用混音输出
    if (outputDesc == nullptr) {
        outputDesc = findMixerOutput(device);
    }
    
    // 5. 创建 Track 客户端描述
    sp<TrackClientDescriptor> client = new TrackClientDescriptor(
            session, uid, attr, *stream, config, flags);
    outputDesc->addClient(client);
    
    // 6. 返回结果
    *selectedDeviceId = device;
    *portId = generatePortId();
    
    return outputDesc->mIoHandle;
}
```

### 3.8 Engine 设备选择核心 (getDevicesForStrategyInt)

```cpp
// Engine.cpp - 策略决策核心
audio_devices_t Engine::getDevicesForStrategyInt(
        product_strategy_t strategy,
        const DeviceVector& availableDevices,
        const SwAudioOutputCollection& outputs) {
    
    audio_devices_t device = AUDIO_DEVICE_NONE;
    
    switch (strategy) {
        case PRODUCT_STRATEGY_MEDIA: {
            // 1. 强制配置检查 (Force Use)
            if (getForceUse(AUDIO_POLICY_FORCE_FOR_MEDIA) == 
                    AUDIO_POLICY_FORCE_SPEAKER) {
                return AUDIO_DEVICE_OUT_SPEAKER;
            }
            
            // 2. 外部设备优先级: A2DP > Wired > USB > Speaker
            if (availableDevices.contains(AUDIO_DEVICE_OUT_BLUETOOTH_A2DP)) {
                device = AUDIO_DEVICE_OUT_BLUETOOTH_A2DP;
            } else if (availableDevices.contains(AUDIO_DEVICE_OUT_WIRED_HEADPHONE) ||
                       availableDevices.contains(AUDIO_DEVICE_OUT_WIRED_HEADSET)) {
                device = AUDIO_DEVICE_OUT_WIRED_HEADPHONE;
            } else if (availableDevices.contains(AUDIO_DEVICE_OUT_USB_DEVICE)) {
                device = AUDIO_DEVICE_OUT_USB_DEVICE;
            } else {
                device = AUDIO_DEVICE_OUT_SPEAKER;
            }
            break;
        }
        
        case PRODUCT_STRATEGY_PHONE: {
            // 通话策略: 蓝牙 SCO > 有线耳机 > 听筒
            if (getForceUse(AUDIO_POLICY_FORCE_FOR_COMMUNICATION) ==
                    AUDIO_POLICY_FORCE_BT_SCO) {
                device = AUDIO_DEVICE_OUT_BLUETOOTH_SCO;
            } else if (availableDevices.contains(AUDIO_DEVICE_OUT_WIRED_HEADSET)) {
                device = AUDIO_DEVICE_OUT_WIRED_HEADSET;
            } else {
                device = AUDIO_DEVICE_OUT_EARPIECE;
            }
            break;
        }
        
        case PRODUCT_STRATEGY_SONIFICATION: {
            // 铃声策略: Dual Routing (耳机 + 扬声器同时出声)
            audio_devices_t devices = AUDIO_DEVICE_NONE;
            if (availableDevices.contains(AUDIO_DEVICE_OUT_WIRED_HEADPHONE)) {
                devices |= AUDIO_DEVICE_OUT_WIRED_HEADPHONE;
            }
            devices |= AUDIO_DEVICE_OUT_SPEAKER;  // 始终包含扬声器
            device = devices;
            break;
        }
        
        default:
            device = AUDIO_DEVICE_OUT_SPEAKER;
            break;
    }
    
    return device;
}
```

---

## 4. 设备管理

### 4.1 设备类型

```cpp
// 输出设备
AUDIO_DEVICE_OUT_EARPIECE          = 0x1,       // 听筒
AUDIO_DEVICE_OUT_SPEAKER           = 0x2,       // 扬声器
AUDIO_DEVICE_OUT_WIRED_HEADSET     = 0x4,       // 有线耳机(带麦)
AUDIO_DEVICE_OUT_WIRED_HEADPHONE   = 0x8,       // 有线耳机(无麦)
AUDIO_DEVICE_OUT_BLUETOOTH_SCO     = 0x10,      // 蓝牙 SCO
AUDIO_DEVICE_OUT_BLUETOOTH_A2DP    = 0x100,     // 蓝牙 A2DP
AUDIO_DEVICE_OUT_AUX_DIGITAL       = 0x400,     // HDMI
AUDIO_DEVICE_OUT_USB_DEVICE        = 0x1000,    // USB 设备
AUDIO_DEVICE_OUT_USB_ACCESSORY     = 0x2000,    // USB 配件
AUDIO_DEVICE_OUT_DGTL_DOCK_HEADSET = 0x4000,    // 数字底座
AUDIO_DEVICE_OUT_ANLG_DOCK_HEADSET = 0x8000,    // 模拟底座
AUDIO_DEVICE_OUT_ALL_A2DP          = 0x300,     // 所有 A2DP
AUDIO_DEVICE_OUT_ALL_SCO           = 0x70,      // 所有 SCO
AUDIO_DEVICE_OUT_ALL_USB           = 0x3000,    // 所有 USB
AUDIO_DEVICE_OUT_BUS               = 0x100000,  // 车载 Bus

// 输入设备
AUDIO_DEVICE_IN_BUILTIN_MIC        = 0x10000,   // 内置麦克风
AUDIO_DEVICE_IN_BLUETOOTH_SCO_HEADSET = 0x80000, // 蓝牙 SCO
AUDIO_DEVICE_IN_WIRED_HEADSET      = 0x400000,  // 有线耳机麦
AUDIO_DEVICE_IN_USB_DEVICE         = 0x200000,  // USB 麦克风
AUDIO_DEVICE_IN_FM_TUNER           = 0x4000,    // FM 调谐器
AUDIO_DEVICE_IN_BUS                = 0x4000000, // 车载 Bus
```

### 4.2 设备连接状态

```cpp
status_t AudioPolicyManager::setDeviceConnectionState(
        audio_devices_t device,
        audio_policy_dev_state_t state,
        const char* device_address,
        const char* device_name) {
    
    ALOGD("setDeviceConnectionState() device: 0x%X, state: %d, address: %s",
          device, state, device_address);
    
    // 1. 验证设备类型
    if (!audio_is_output_device(device) && !audio_is_input_device(device)) {
        return BAD_VALUE;
    }
    
    // 2. 处理输出设备
    if (audio_is_output_device(device)) {
        if (state == AUDIO_POLICY_DEVICE_STATE_AVAILABLE) {
            // 添加设备
            sp<DeviceDescriptor> devDesc = mHwModules.getDeviceDescriptor(
                    device, device_address, device_name);
            mAvailableOutputDevices.add(devDesc);
            
            // 打开对应的输出流
            if (!mOutputs.contains(devDesc->mModule->mHandle)) {
                openOutput(devDesc->mModule, devDesc);
            }
        } else {
            // 移除设备
            sp<DeviceDescriptor> devDesc = mAvailableOutputDevices.getDevice(
                    device, device_address);
            if (devDesc != nullptr) {
                mAvailableOutputDevices.remove(devDesc);
                
                // 关闭对应的输出流
                closeOutputIfNeeded(devDesc);
            }
        }
    }
    
    // 3. 处理输入设备 (类似逻辑)
    // ...
    
    // 4. 更新路由
    updateCallRouting();
    checkStrategyRoute(STRATEGY_MEDIA, AUDIO_IO_HANDLE_NONE);
    
    // 5. 通知 AudioFlinger
    mpClientInterface->onDeviceAvailable(device, device_address);
    
    return NO_ERROR;
}
```

### 4.3 DeviceVector

```cpp
class DeviceVector : public SortedVector<sp<DeviceDescriptor>> {
public:
    // 按类型查找
    sp<DeviceDescriptor> getDevice(audio_devices_t type,
                                    const String8& address) const;
    
    // 按类型过滤
    DeviceVector getDevicesByType(audio_devices_t type) const;
    
    // 获取所有类型
    audio_devices_t types() const;
    
    // 按模块过滤
    DeviceVector getDevicesFromHwModule(audio_module_handle_t module) const;
    
    // 检查是否包含
    bool contains(audio_devices_t type) const;
    
    // 按地址查找
    sp<DeviceDescriptor> getDeviceFromId(audio_port_handle_t id) const;
};

// 使用示例
DeviceVector devices = mAvailableOutputDevices;
if (devices.contains(AUDIO_DEVICE_OUT_SPEAKER)) {
    sp<DeviceDescriptor> speaker = devices.getDevice(
            AUDIO_DEVICE_OUT_SPEAKER, String8(""));
}
```

---

## 5. AudioPort 与 AudioPatch

### 5.1 AudioPort 结构

```cpp
struct audio_port {
    audio_port_role_t role;          // source/sink
    audio_port_type_t type;          // device/mix/session
    
    audio_port_handle_t id;          // 唯一标识
    char name[AUDIO_PORT_NAME_LEN];  // 端口名称
    
    // 支持的配置
    unsigned int num_sample_rates;
    unsigned int sample_rates[AUDIO_PORT_MAX_SAMPLING_RATES];
    
    unsigned int num_channel_masks;
    audio_channel_mask_t channel_masks[AUDIO_PORT_MAX_CHANNEL_MASKS];
    
    unsigned int num_formats;
    audio_format_t formats[AUDIO_PORT_MAX_FORMATS];
    
    // 增益
    unsigned int num_gains;
    struct audio_gain gains[AUDIO_PORT_MAX_GAINS];
    
    // 配置
    struct audio_port_config active_config;
};

// Port 类型
enum audio_port_type_t {
    AUDIO_PORT_TYPE_NONE = 0,
    AUDIO_PORT_TYPE_DEVICE = 1,   // 设备端口
    AUDIO_PORT_TYPE_MIX = 2,      // 混音端口 (AudioFlinger)
    AUDIO_PORT_TYPE_SESSION = 3,  // 会话端口
};

// Port 角色
enum audio_port_role_t {
    AUDIO_PORT_ROLE_NONE = 0,
    AUDIO_PORT_ROLE_SOURCE = 1,   // 源 (输出设备/输入流)
    AUDIO_PORT_ROLE_SINK = 2,     // 目标 (输入设备/输出流)
};
```

### 5.2 AudioPortConfig

```cpp
struct audio_port_config {
    audio_port_role_t role;
    audio_port_type_t type;
    audio_port_handle_t id;
    
    // 配置掩码 - 标识哪些字段有效
    unsigned int config_mask;
    
    // 配置值
    unsigned int sample_rate;
    audio_channel_mask_t channel_mask;
    audio_format_t format;
    struct audio_gain_config gain;
    
    // 设备相关
    audio_devices_t device;
    char address[AUDIO_DEVICE_MAX_ADDRESS_LEN];
    
    // Mix 相关
    audio_module_handle_t hw_module;
    audio_io_handle_t io_handle;      // AudioFlinger output/input handle
    audio_stream_type_t stream;
    audio_source_t source;
    audio_session_t session;
};

// config_mask 标志
enum {
    AUDIO_PORT_CONFIG_SAMPLE_RATE = 0x1,
    AUDIO_PORT_CONFIG_CHANNEL_MASK = 0x2,
    AUDIO_PORT_CONFIG_FORMAT = 0x4,
    AUDIO_PORT_CONFIG_GAIN = 0x8,
    AUDIO_PORT_CONFIG_ALL = 0xF,
};
```

### 5.3 AudioPatch

```cpp
struct audio_patch {
    audio_patch_handle_t id;          // Patch 唯一标识
    
    unsigned int num_sources;
    struct audio_port_config sources[AUDIO_PATCH_PORTS_MAX];
    
    unsigned int num_sinks;
    struct audio_port_config sinks[AUDIO_PATCH_PORTS_MAX];
};

// 创建 AudioPatch
status_t AudioPolicyManager::createAudioPatch(
        const struct audio_patch* patch,
        audio_patch_handle_t* handle) {
    
    ALOGD("createAudioPatch() sources: %d, sinks: %d",
          patch->num_sources, patch->num_sinks);
    
    // 1. 验证 Patch
    if (patch->num_sources == 0 || patch->num_sinks == 0) {
        return BAD_VALUE;
    }
    
    // 2. 判断 Patch 类型
    audio_port_type_t sourceType = patch->sources[0].type;
    audio_port_type_t sinkType = patch->sinks[0].type;
    
    // 3. 处理不同类型的 Patch
    if (sourceType == AUDIO_PORT_TYPE_MIX && sinkType == AUDIO_PORT_TYPE_DEVICE) {
        // AudioFlinger -> 设备 (正常播放路由)
        return createAudioPatchInternal(patch, handle);
        
    } else if (sourceType == AUDIO_PORT_TYPE_DEVICE && sinkType == AUDIO_PORT_TYPE_MIX) {
        // 设备 -> AudioFlinger (正常录音路由)
        return createAudioPatchInternal(patch, handle);
        
    } else if (sourceType == AUDIO_PORT_TYPE_DEVICE && sinkType == AUDIO_PORT_TYPE_DEVICE) {
        // 设备 -> 设备 (Loopback, 如 FM -> Speaker)
        return createAudioPatchInternal(patch, handle);
    }
    
    return INVALID_OPERATION;
}

// 释放 AudioPatch
status_t AudioPolicyManager::releaseAudioPatch(audio_patch_handle_t handle) {
    ALOGD("releaseAudioPatch() handle: %d", handle);
    
    // 1. 查找 Patch
    sp<AudioPatch> patch = mAudioPatches.valueFor(handle);
    if (patch == nullptr) {
        return BAD_VALUE;
    }
    
    // 2. 通知 HAL 释放
    mpClientInterface->releaseAudioPatch(handle, mPatchHandle);
    
    // 3. 移除记录
    mAudioPatches.removeItem(handle);
    
    return NO_ERROR;
}
```

### 5.4 Bus 设备 (车载场景)

```cpp
// Bus 设备定义
audio_devices_t busDevice = AUDIO_DEVICE_OUT_BUS;

// Bus 地址格式
"bus0_media_out"      // 媒体音频
"bus1_navigation_out" // 导航音频
"bus2_voice_call_out" // 通话音频
"bus3_chime_out"      // 提示音

// audio_policy_configuration.xml 配置
<devicePort tagName="Bus 0 Media Out"
            role="sink"
            type="AUDIO_DEVICE_OUT_BUS"
            address="bus0_media_out">
    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
             samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
    <gains>
        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
              minValueMB="-3200" maxValueMB="600" defaultValueMB="0"/>
    </gains>
</devicePort>

// HAL 层处理 Bus 设备
int set_device(struct audio_stream_out *stream, audio_devices_t device) {
    if (device & AUDIO_DEVICE_OUT_BUS) {
        const char* address = stream->get_device_address(stream);
        int bus_id = parse_bus_id(address);
        configure_dsp_route(bus_id);
    }
}
```

---

## 6. AudioMix

### 6.1 AudioMix 定义

```cpp
struct AudioMix {
    audio_mix_t mix;                  // Mix 配置
    sp<AudioOutputDescriptor> output; // 关联的输出
    Vector<audio_attributes_t> attributes; // 匹配的属性
};

struct audio_mix_t {
    audio_mix_rule_t rules[MAX_MIX_RULES]; // 匹配规则
    audio_format_t format;                  // 格式
    uint32_t sample_rate;                   // 采样率
    audio_channel_mask_t channel_mask;      // 通道
    audio_output_flags_t flags;             // 标志
    audio_devices_t device;                 // 目标设备
    char address[AUDIO_DEVICE_MAX_ADDRESS_LEN];
};
```

### 6.2 AudioMix 规则

```cpp
enum audio_mix_rule_type {
    RULE_TYPE_MATCH_ATTR = 0x1,        // 匹配 attributes
    RULE_TYPE_EXCLUDE_ATTR = 0x2,      // 排除 attributes
    RULE_TYPE_MATCH_UID = 0x4,         // 匹配 UID
    RULE_TYPE_EXCLUDE_UID = 0x8,       // 排除 UID
    RULE_TYPE_MATCH_STREAM = 0x10,     // 匹配 stream type
    RULE_TYPE_EXCLUDE_STREAM = 0x20,   // 排除 stream type
};

struct audio_mix_rule_t {
    audio_mix_rule_type type;
    audio_attributes_t attr;
    uid_t uid;
    audio_stream_type_t stream;
};

// 注册 AudioMix
status_t AudioPolicyManager::registerAudioMix(const audio_mix_t& mix) {
    // 创建专用输出
    audio_io_handle_t output = openOutputForMix(mix);
    
    // 保存 Mix
    sp<AudioMix> audioMix = new AudioMix(mix, output);
    mAudioMixes.add(audioMix);
    
    return NO_ERROR;
}
```

### 6.3 多区音频 (Multi-Zone Audio)

```cpp
// 主区音频
audio_attributes_t mainZoneAttr = {
    .usage = AUDIO_USAGE_MEDIA,
    .content_type = AUDIO_CONTENT_TYPE_MUSIC,
    .tags = ""
};

// 后排娱乐区
audio_attributes_t rearZoneAttr = {
    .usage = AUDIO_USAGE_MEDIA,
    .content_type = AUDIO_CONTENT_TYPE_MUSIC,
    .tags = "bus=rear_entertainment"
};

// 配置不同 Bus 的 Mix
audio_mix_t mainMix = {
    .device = AUDIO_DEVICE_OUT_BUS,
    .address = "bus0_media_out",
    // ...
};

audio_mix_t rearMix = {
    .device = AUDIO_DEVICE_OUT_BUS,
    .address = "bus4_rear_entertainment",
    // ...
};
```

---

## 7. 音量管理

### 7.1 VolumeCurves

```cpp
class VolumeCurves {
public:
    // 获取音量索引
    status_t getVolumeIndex(audio_stream_type_t stream,
                            audio_devices_t device,
                            int* index);
    
    // 设置音量索引
    status_t setVolumeIndex(audio_stream_type_t stream,
                            audio_devices_t device,
                            int index);
    
    // 获取音量曲线
    const VolumeCurvePoint* getVolumeCurve(audio_stream_type_t stream,
                                           audio_devices_t device);
    
    // 转换为 dB
    float volIndexToDb(int index, audio_stream_type_t stream,
                       audio_devices_t device);
    
private:
    KeyedVector<audio_stream_type_t, VolumeCurvePoint*> mCurves;
};

// 音量曲线点
struct VolumeCurvePoint {
    int mIndex;    // 音量索引 (0-100)
    float mDb;     // 对应 dB 值
};

// 默认曲线示例
static const VolumeCurvePoint defaultCurve[] = {
    { 0, -9600.0f },   // 静音
    { 1, -4800.0f },   // 最小音量
    { 20, -2400.0f },
    { 60, -1200.0f },
    { 100, 0.0f },     // 最大音量
};
```

### 7.2 音量设置流程

```cpp
status_t AudioPolicyManager::setStreamVolumeIndex(
        audio_stream_type_t stream,
        int index,
        audio_devices_t device) {
    
    ALOGD("setStreamVolumeIndex() stream: %d, index: %d, device: 0x%X",
          stream, index, device);
    
    // 1. 验证参数
    if (index < 0 || index > 100) {
        return BAD_VALUE;
    }
    
    // 2. 更新音量曲线
    mVolumeCurves.setVolumeIndex(stream, device, index);
    
    // 3. 计算实际增益
    float gainDb = mVolumeCurves.volIndexToDb(index, stream, device);
    
    // 4. 应用到相关输出
    for (size_t i = 0; i < mOutputs.size(); i++) {
        sp<SwAudioOutputDescriptor> outputDesc = mOutputs.valueAt(i);
        
        // 检查输出是否使用该设备
        if (outputDesc->devices() & device) {
            // 设置音量到 AudioFlinger
            mpClientInterface->setStreamVolume(
                    stream, gainDb, outputDesc->mIoHandle);
        }
    }
    
    return NO_ERROR;
}
```

### 7.3 Ducking (压低音量)

```cpp
// Ducking 状态
class AudioPolicyManager {
    void checkDucking() {
        // 1. 检查是否有高优先级音频
        bool hasHighPriority = false;
        for (size_t i = 0; i < mOutputs.size(); i++) {
            sp<SwAudioOutputDescriptor> outputDesc = mOutputs.valueAt(i);
            if (outputDesc->hasHighPriorityClient()) {
                hasHighPriority = true;
                break;
            }
        }
        
        // 2. 对低优先级音频应用 Ducking
        for (size_t i = 0; i < mOutputs.size(); i++) {
            sp<SwAudioOutputDescriptor> outputDesc = mOutputs.valueAt(i);
            
            float volume = 1.0f;
            if (hasHighPriority && outputDesc->canBeDucked()) {
                volume = 0.3f;  // Ducking 到 30%
            }
            
            mpClientInterface->setStreamVolume(
                    AUDIO_STREAM_MUSIC, volume, outputDesc->mIoHandle);
        }
    }
};
```

---

## 8. 配置文件解析

### 8.1 audio_policy_configuration.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<audioPolicyConfiguration version="1.0">
    
    <!-- 全局配置 -->
    <globalConfiguration speaker_drc_enabled="true"/>
    
    <!-- 硬件模块 -->
    <modules>
        <module name="primary" halVersion="2.0">
            <attachedDevices>
                <item>Speaker</item>
                <item>Built-In Mic</item>
            </attachedDevices>
            
            <defaultOutputDevice>Speaker</defaultOutputDevice>
            
            <!-- 混音端口 -->
            <mixPorts>
                <mixPort name="primary output" role="source" flags="AUDIO_OUTPUT_FLAG_PRIMARY">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
                <mixPort name="primary input" role="sink">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000" channelMasks="AUDIO_CHANNEL_IN_MONO"/>
                </mixPort>
            </mixPorts>
            
            <!-- 设备端口 -->
            <devicePorts>
                <devicePort tagName="Speaker" type="AUDIO_DEVICE_OUT_SPEAKER" role="sink">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </devicePort>
                <devicePort tagName="Built-In Mic" type="AUDIO_DEVICE_IN_BUILTIN_MIC" role="source">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000" channelMasks="AUDIO_CHANNEL_IN_MONO"/>
                </devicePort>
            </devicePorts>
            
            <!-- 路由规则 -->
            <routes>
                <route type="mix" sink="Speaker" sources="primary output"/>
                <route type="mix" sink="primary input" sources="Built-In Mic"/>
            </routes>
        </module>
    </modules>
</audioPolicyConfiguration>
```

### 8.2 audio_policy_engine_configuration.xml (新架构)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<audioPolicyEngineConfiguration version="1.0">
    
    <!-- Product Strategies 定义 -->
    <productStrategies>
        <productStrategy name="strategy_media" id="0">
            <attributesStream>
                <item>
                    <attributes>
                        <usage>AUDIO_USAGE_MEDIA</usage>
                        <contentType>AUDIO_CONTENT_TYPE_MUSIC</contentType>
                    </attributes>
                </item>
                <item>
                    <attributes>
                        <usage>AUDIO_USAGE_GAME</usage>
                    </attributes>
                </item>
            </attributesStream>
        </productStrategy>
        
        <productStrategy name="strategy_phone" id="1">
            <attributesStream>
                <item>
                    <attributes>
                        <usage>AUDIO_USAGE_VOICE_COMMUNICATION</usage>
                    </attributes>
                </item>
            </attributesStream>
        </productStrategy>
        
        <productStrategy name="strategy_sonification" id="2">
            <attributesStream>
                <item>
                    <attributes>
                        <usage>AUDIO_USAGE_ALARM</usage>
                    </attributes>
                </item>
                <item>
                    <attributes>
                        <usage>AUDIO_USAGE_NOTIFICATION</usage>
                    </attributes>
                </item>
            </attributesStream>
        </productStrategy>
    </productStrategies>
    
    <!-- Volume Groups -->
    <volumeGroups>
        <volumeGroup name="volume_group_media" stream="AUDIO_STREAM_MUSIC">
            <productStrategy ref="strategy_media"/>
        </volumeGroup>
        <volumeGroup name="volume_group_ring" stream="AUDIO_STREAM_RING">
            <productStrategy ref="strategy_sonification"/>
        </volumeGroup>
    </volumeGroups>
    
</audioPolicyEngineConfiguration>
```

### 8.3 配置解析流程

```cpp
status_t AudioPolicyConfig::parse(const char* path) {
    XMLDocument doc;
    doc.LoadFile(path);
    
    XMLElement* root = doc.RootElement();
    
    // 解析模块
    XMLElement* modules = root->FirstChildElement("modules");
    for (XMLElement* module = modules->FirstChildElement("module");
         module != nullptr;
         module = module->NextSiblingElement("module")) {
        
        sp<HwModule> hwModule = parseModule(module);
        mHwModules.add(hwModule);
    }
    
    return NO_ERROR;
}

sp<HwModule> AudioPolicyConfig::parseModule(XMLElement* elem) {
    const char* name = elem->Attribute("name");
    sp<HwModule> module = new HwModule(name);
    
    // 解析 mixPorts
    XMLElement* mixPorts = elem->FirstChildElement("mixPorts");
    for (XMLElement* port = mixPorts->FirstChildElement("mixPort");
         port != nullptr;
         port = port->NextSiblingElement("mixPort")) {
        sp<IOProfile> profile = parseMixPort(port);
        module->addProfile(profile);
    }
    
    // 解析 devicePorts
    XMLElement* devicePorts = elem->FirstChildElement("devicePorts");
    for (XMLElement* port = devicePorts->FirstChildElement("devicePort");
         port != nullptr;
         port = port->NextSiblingElement("devicePort")) {
        sp<DeviceDescriptor> device = parseDevicePort(port);
        module->addDevice(device);
    }
    
    // 解析 routes
    XMLElement* routes = elem->FirstChildElement("routes");
    for (XMLElement* route = routes->FirstChildElement("route");
         route != nullptr;
         route = route->NextSiblingElement("route")) {
        parseRoute(route, module);
    }
    
    return module;
}
```

---

## 9. AudioPolicyClientInterface

### 9.1 回调接口

```cpp
class AudioPolicyClientInterface {
public:
    // 输出管理
    virtual audio_io_handle_t openOutput(audio_module_handle_t module,
                                         audio_devices_t* devices,
                                         audio_config_t* config,
                                         audio_output_flags_t flags) = 0;
    
    virtual status_t closeOutput(audio_io_handle_t output) = 0;
    
    // 输入管理
    virtual audio_io_handle_t openInput(audio_module_handle_t module,
                                        audio_devices_t* devices,
                                        audio_config_t* config) = 0;
    
    virtual status_t closeInput(audio_io_handle_t input) = 0;
    
    // 流控制
    virtual status_t setStreamVolume(audio_stream_type_t stream,
                                     float volume,
                                     audio_io_handle_t output) = 0;
    
    virtual status_t setParameters(audio_io_handle_t ioHandle,
                                   const String8& keyValuePairs) = 0;
    
    // 设备通知
    virtual status_t setDeviceConnectionState(audio_devices_t device,
                                              audio_policy_dev_state_t state,
                                              const char* address) = 0;
    
    // 音效
    virtual status_t createAudioEffect(const effect_uuid_t* uuid,
                                       int sessionId,
                                       int ioHandle,
                                       effect_handle_t* handle) = 0;
    
    // Patch
    virtual status_t createAudioPatch(const struct audio_patch* patch,
                                      audio_patch_handle_t* handle) = 0;
    
    virtual status_t releaseAudioPatch(audio_patch_handle_t handle) = 0;
};
```

### 9.2 AudioPolicyClientImpl

```cpp
class AudioPolicyClientImpl : public AudioPolicyClientInterface {
public:
    AudioPolicyClientImpl(AudioPolicyService* service)
            : mService(service) {}
    
    audio_io_handle_t openOutput(audio_module_handle_t module,
                                 audio_devices_t* devices,
                                 audio_config_t* config,
                                 audio_output_flags_t flags) override {
        // 调用 AudioFlinger
        return mService->audioFlinger()->openOutput(module, devices, config, flags);
    }
    
    status_t setStreamVolume(audio_stream_type_t stream,
                             float volume,
                             audio_io_handle_t output) override {
        return mService->audioFlinger()->setStreamVolume(stream, volume, output);
    }
    
private:
    AudioPolicyService* mService;
};
```

---

## 10. 调试技巧

### 10.1 dumpsys

```bash
# 完整 AudioPolicy 状态
dumpsys audio

# 查看可用设备
dumpsys audio | grep -A 30 "Available devices"

# 查看输出配置
dumpsys audio | grep -A 50 "Outputs"

# 查看路由策略
dumpsys audio | grep -A 20 "Strategy"

# 查看 AudioPatch
dumpsys audio | grep -A 10 "Patch"

# 查看音量曲线
dumpsys audio | grep -A 10 "Volume"
```

### 10.2 日志过滤

```bash
# AudioPolicyService 日志
adb logcat -s AudioPolicyService

# AudioPolicyManager 日志
adb logcat | grep -E "AudioPolicyManager|AudioPolicy"

# 设备连接日志
adb logcat | grep -E "setDeviceConnectionState|device.*connected"

# 路由日志
adb logcat | grep -E "getOutputForAttr|getDeviceForStrategy"

# 音量日志
adb logcat | grep -E "setStreamVolume|VolumeIndex"
```

### 10.3 常见问题定位

```bash
# 1. 设备未识别
dumpsys audio | grep -i "available"
adb logcat | grep -i "device.*not.*found"

# 2. 路由错误
dumpsys audio | grep -A 5 "Strategy"
adb logcat | grep -E "getDeviceForStrategy|route"

# 3. 音量问题
dumpsys audio | grep -i "volume"
adb logcat | grep -i "volume"

# 4. 配置解析问题
adb logcat | grep -E "audio_policy|parse.*error"
```

### 10.4 配置文件调试

```bash
# 查看当前配置
adb shell cat /vendor/etc/audio_policy_configuration.xml

# 检查配置是否正确加载
dumpsys audio | grep -A 5 "Config"

# 强制重新加载配置
adb shell "killall audioserver"
```

---

## 11. AAOS 车载音频

### 11.1 AAOS 音频架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    AAOS Audio Architecture                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  CarAudioService (Java)                                    ││
│  │  - CarAudioZones (多音区管理)                              ││
│  │  - CarAudioFocus (音区焦点管理)                            ││
│  │  - CarVolumeCallback (音量回调)                            ││
│  └─────────────────────────────────────────────────────────────┘│
│                          ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  AudioPolicyService (Native)                               ││
│  │  - Dynamic Routing (动态路由)                              ││
│  │  - Bus Device Mapping (Bus 设备映射)                       ││
│  └─────────────────────────────────────────────────────────────┘│
│                          ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Audio HAL                                                 ││
│  │  - AUDIO_DEVICE_OUT_BUS (车载 Bus 设备)                    ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 11.2 启用 AAOS 路由

```xml
<!-- frameworks/base/core/res/res/values/config.xml -->
<resources>
    <bool name="audioUseDynamicRouting">true</bool>
</resources>
```

### 11.3 car_audio_configuration.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<carAudioConfiguration version="2">
    
    <!-- 音区定义 -->
    <zones>
        <!-- 主音区 (驾驶员) -->
        <zone name="primary zone" isPrimary="true">
            <volumeGroups>
                <!-- 音量组 1: 媒体 -->
                <group>
                    <device address="bus0_media_out">
                        <context context="music"/>
                        <context context="announcement"/>
                    </device>
                    <device address="bus1_navigation_out">
                        <context context="navigation"/>
                    </device>
                </group>
                
                <!-- 音量组 2: 通话 -->
                <group>
                    <device address="bus2_voice_call_out">
                        <context context="call"/>
                    </device>
                    <device address="bus3_call_ring_out">
                        <context context="call_ring"/>
                    </device>
                </group>
                
                <!-- 音量组 3: 系统 -->
                <group>
                    <device address="bus4_system_out">
                        <context context="system"/>
                        <context context="safety"/>
                        <context context="emergency"/>
                    </device>
                </group>
            </volumeGroups>
        </zone>
        
        <!-- 后排音区 -->
        <zone name="rear zone">
            <volumeGroups>
                <group>
                    <device address="bus10_rear_media_out">
                        <context context="music"/>
                    </device>
                </group>
            </volumeGroups>
        </zone>
    </zones>
    
</carAudioConfiguration>
```

### 11.4 Audio Context

```java
// Audio Context 定义 (AAOS 特有)
public static final int[] CONTEXTS = {
    CarAudioContext.MUSIC,           // 音乐
    CarAudioContext.NAVIGATION,      // 导航
    CarAudioContext.VOICE_COMMAND,   // 语音助手
    CarAudioContext.CALL_RING,       // 来电铃声
    CarAudioContext.CALL,            // 通话
    CarAudioContext.ALARM,           // 闹钟
    CarAudioContext.NOTIFICATION,    // 通知
    CarAudioContext.SYSTEM,          // 系统音
    CarAudioContext.SAFETY,          // 安全警告
    CarAudioContext.EMERGENCY,       // 紧急情况
    CarAudioContext.VEHICLE_STATUS,  // 车辆状态
    CarAudioContext.ANNOUNCEMENT,    // 广播
};

// Usage -> Context 映射
CarAudioContext.getContextForUsage(AudioAttributes.USAGE_MEDIA)
    -> CarAudioContext.MUSIC
```

### 11.5 CarAudioService

```java
// CarAudioService 初始化
public class CarAudioService extends ICarAudio.Stub {
    
    // 音区管理
    private final SparseArray<CarAudioZone> mCarAudioZones;
    
    // 初始化
    @Override
    public void init() {
        // 1. 解析 car_audio_configuration.xml
        CarAudioConfiguration config = parseCarAudioConfiguration();
        
        // 2. 创建音区
        for (ZoneConfiguration zoneConfig : config.getZones()) {
            CarAudioZone zone = new CarAudioZone(zoneConfig);
            mCarAudioZones.put(zone.getId(), zone);
        }
        
        // 3. 设置动态路由
        setupDynamicRouting();
        
        // 4. 初始化音量组
        setupVolumeGroups();
    }
    
    // 设置音量
    @Override
    public void setGroupVolume(int zoneId, int groupId, int index, int flags) {
        CarAudioZone zone = mCarAudioZones.get(zoneId);
        CarVolumeGroup group = zone.getVolumeGroup(groupId);
        group.setVolumeIndex(index);
        
        // 通知回调
        dispatchVolumeChange(zoneId, groupId, index);
    }
    
    // 获取音量
    @Override
    public int getGroupVolume(int zoneId, int groupId) {
        return mCarAudioZones.get(zoneId)
                .getVolumeGroup(groupId)
                .getVolumeIndex();
    }
}
```

### 11.6 CarAudioFocus

```java
// AAOS 音频焦点管理 (每个音区独立)
public class CarAudioFocus {
    
    // 焦点请求
    public int requestAudioFocus(int zoneId, 
                                  AudioFocusInfo focusInfo,
                                  int requestResult) {
        
        CarAudioZone zone = mCarAudioZones.get(zoneId);
        
        // 检查当前焦点持有者
        AudioFocusInfo currentHolder = zone.getFocusHolder();
        
        // 焦点策略判断
        switch (focusInfo.getGainRequest()) {
            case AudioManager.AUDIOFOCUS_GAIN:
                // 永久焦点
                break;
            case AudioManager.AUDIOFOCUS_GAIN_TRANSIENT:
                // 临时焦点 (如导航)
                break;
            case AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK:
                // 临时焦点 + Ducking
                break;
        }
        
        // 通知焦点变化
        dispatchFocusChange(zoneId, focusInfo);
        
        return AudioManager.AUDIOFOCUS_REQUEST_GRANTED;
    }
    
    // 放弃焦点
    public void abandonAudioFocus(int zoneId, AudioFocusInfo focusInfo) {
        CarAudioZone zone = mCarAudioZones.get(zoneId);
        zone.clearFocusHolder(focusInfo);
    }
}
```

### 11.7 CarVolumeCallback

```java
// 音量变化回调
public abstract class CarVolumeCallback {
    
    // 音量组变化
    public void onGroupVolumeChanged(int zoneId, int groupId, int flags) {}
    
    // 主音量变化
    public void onMasterMuteChanged(int zoneId, int flags) {}
}

// 注册回调
CarAudioManager carAudioManager = (CarAudioManager) car.getCarManager(Car.AUDIO_SERVICE);
carAudioManager.registerCarVolumeCallback(new CarVolumeCallback() {
    @Override
    public void onGroupVolumeChanged(int zoneId, int groupId, int flags) {
        // 更新 UI
        updateVolumeUI(zoneId, groupId);
    }
});
```

### 11.8 Bus 设备配置

```xml
<!-- audio_policy_configuration.xml 中的 Bus 设备 -->
<devicePort tagName="Bus 0 Media Out"
            role="sink"
            type="AUDIO_DEVICE_OUT_BUS"
            address="bus0_media_out">
    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
             samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
    <gains>
        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
              minValueMB="-3200" maxValueMB="600" defaultValueMB="0"/>
    </gains>
</devicePort>

<devicePort tagName="Bus 1 Navigation Out"
            role="sink"
            type="AUDIO_DEVICE_OUT_BUS"
            address="bus1_navigation_out">
    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
             samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
</devicePort>
```

### 11.9 AAOS 调试

```bash
# 查看 CarAudioService 状态
dumpsys car_service | grep -A 50 "CarAudio"

# 查看音区配置
dumpsys car_service | grep -A 20 "AudioZone"

# 查看音量组
dumpsys car_service | grep -A 10 "VolumeGroup"

# 查看焦点状态
dumpsys car_service | grep -A 10 "AudioFocus"

# 查看 car_audio_configuration.xml
adb shell cat /vendor/etc/car_audio_configuration.xml

# 设置音量
adb shell cmd car_service set-group-volume <zoneId> <groupId> <index>

# 获取音量
adb shell cmd car_service get-group-volume <zoneId> <groupId>
```

---

## 📌 总结

| 类别 | 关键类/接口 |
|------|------------|
| **核心服务** | AudioPolicyService / AudioPolicyManager |
| **策略决策** | Engine / ProductStrategy |
| **新架构路由** | Attributes -> Product Strategy -> Device |
| **旧架构路由** | Attributes -> Stream Type -> Strategy -> Device |
| **设备管理** | DeviceVector / setDeviceConnectionState |
| **输出选择** | getOutputForAttr / getInputForAttr |
| **AudioPort** | audio_port / audio_port_config |
| **AudioPatch** | createAudioPatch / releaseAudioPatch |
| **AudioMix** | audio_mix_t / registerAudioMix |
| **音量管理** | VolumeCurves / VolumeGroup |
| **配置文件** | audio_policy_configuration.xml / audio_policy_engine_configuration.xml |
| **AAOS 核心** | CarAudioService / CarAudioZone / CarAudioFocus |
| **AAOS 配置** | car_audio_configuration.xml |
| **AAOS Context** | CarAudioContext (MUSIC/NAVIGATION/CALL/etc.) |
| **AAOS 设备** | AUDIO_DEVICE_OUT_BUS |
| **调试** | dumpsys audio / dumpsys car_service / logcat |

### 架构演进要点

```
┌─────────────────────────────────────────────────────────────────┐
│  旧架构: Stream Type 是路由的核心                                │
│  新架构: audio_attributes_t 是一等公民                           │
├─────────────────────────────────────────────────────────────────┤
│  Engine 是策略决策模块 (Strategy Decision Module)               │
│  APM 是执行层，根据 Engine 决策操作 AudioPolicyClient           │
├─────────────────────────────────────────────────────────────────┤
│  分层设计:                                                       │
│  1. 静态配置层 (HwModule/IOProfile) - 定义硬件能力              │
│  2. 动态状态层 (DeviceDescriptor/SwAudioOutputDescriptor)       │
│  3. 策略决策层 (Engine) - getDevicesForStrategyInt 是核心       │
│  4. 执行层 (APM) - 操作 AudioPolicyClient                       │
└─────────────────────────────────────────────────────────────────┘
```
