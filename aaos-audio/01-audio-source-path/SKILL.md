---
name: "audio-source-path"
description: "AAOS 音频系统源码路径索引，涵盖 Java 层、JNI 层、Native 框架层、Audio 服务层、HAL 层与 Vendor 实现层"
version: "1.0.0"
triggers: ["source path", "source code", "frameworks/av", "packages/services/Car", "hardware/interfaces/audio", "libaudioclient", "libaudioflinger", "CarAudioService", "AudioFlinger", "AudioPolicyService"]
---

> 参考来源：Android AOSP Source Code

# 📁 AAOS 音频系统源码路径索引

---

## 1. 架构分层概览

```
┌─────────────────────────────────────────────────────────────────┐
│                    AAOS Audio Source Layers                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Java 应用层 (android.media + android.car.media)            ││
│  │  frameworks/base/media/java/android/media/                  ││
│  │  packages/services/Car/car-lib/src/android/car/media/       ││
│  └─────────────────────────────────────────────────────────────┘│
│                          ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Java 服务层 (AudioService + CarAudioService)               ││
│  │  frameworks/base/services/core/java/com/android/server/     ││
│  │  packages/services/Car/service/src/com/android/car/audio/   ││
│  └─────────────────────────────────────────────────────────────┘│
│                          ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  JNI 层 (libmedia_jni.so + libandroid_runtime.so)           ││
│  │  frameworks/base/media/jni/                                 ││
│  │  frameworks/base/core/jni/                                  ││
│  └─────────────────────────────────────────────────────────────┘│
│                          ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Native 框架层 (libaudioclient + libmedia)                  ││
│  │  frameworks/av/media/libaudioclient/                        ││
│  │  frameworks/av/media/libmedia/                              ││
│  └─────────────────────────────────────────────────────────────┘│
│                          ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Audio 服务层 (AudioFlinger + AudioPolicy + AAudio)         ││
│  │  frameworks/av/services/audioflinger/                       ││
│  │  frameworks/av/services/audiopolicy/                        ││
│  │  frameworks/av/services/oboeservice/                        ││
│  └─────────────────────────────────────────────────────────────┘│
│                          ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  HAL 适配层 (libaudiohal.so)                                ││
│  │  frameworks/av/media/libaudiohal/                           ││
│  └─────────────────────────────────────────────────────────────┘│
│                          ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  AIDL/HIDL 接口层                                           ││
│  │  hardware/interfaces/audio/aidl/                            ││
│  │  hardware/interfaces/automotive/audiocontrol/aidl/          ││
│  └─────────────────────────────────────────────────────────────┘│
│                          ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Vendor 实现层 (QCOM PAL/AGM)                               ││
│  │  vendor/qcom/opensource/audio-hal-ar/                       ││
│  │  vendor/qcom/opensource/pal/                                ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Java 应用层

### 2.1 Standard Media (android.media)

```
路径: frameworks/base/media/java/android/media/

核心类:
├── AudioManager.java          # 音频管理器，音量、路由、焦点
├── AudioTrack.java            # 音频播放轨道
├── AudioRecord.java           # 音频录制
├── AudioSystem.java           # 音频系统底层接口
├── MediaPlayer.java           # 媒体播放器
├── MediaRecorder.java         # 媒体录制器
├── AudioAttributes.java       # 音频属性 (Usage, ContentType)
├── AudioFormat.java           # 音频格式
├── AudioDeviceInfo.java       # 音频设备信息
├── AudioFocusRequest.java     # 音频焦点请求
└── AudioPlaybackCapture.java  # 音频捕获

编译命令:
make framework
make update-api

输出路径:
out/target/product/<device>/system/framework/framework.jar
```

### 2.2 Car Media (android.car.media)

```
路径: packages/services/Car/car-lib/src/android/car/media/

核心类:
├── CarAudioManager.java       # 车载音频管理器
├── CarAudioPatchHandle.java   # 音频 Patch 句柄
├── CarVolumeGroupInfo.java    # 音量组信息
├── CarAudioZoneConfigInfo.java # 音区配置信息
└── ICarAudio.aidl             # 车载音频 AIDL 接口

特点:
- 支持按 Zone (音区) 查询音量和路由状态
- 与 CarAudioService 通过 Binder 通信
- 运行在调用方进程中
```

---

## 3. Java 服务层

### 3.1 AudioService

```
路径: frameworks/base/services/core/java/com/android/server/audio/

核心类:
├── AudioService.java          # 音频服务主类
├── AudioSystemAdapter.java    # AudioSystem 适配器
├── AudioDeviceBroker.java     # 设备管理代理
├── PlaybackActivityMonitor.java # 播放活动监控
├── RecordingActivityMonitor.java # 录音活动监控
├── AudioDeviceInventory.java  # 设备清单
├── SettingsAdapter.java       # 设置适配器
└── SystemServerAdapter.java   # SystemServer 适配器

职责:
- 音量管理 (Stream Volume, Master Volume)
- 音频设备管理
- 音频焦点管理
- 音频路由决策
```

### 3.2 CarAudioService

```
路径: packages/services/Car/service/src/com/android/car/audio/

核心类:
├── CarAudioService.java       # 车载音频服务主类
├── CarAudioContext.java       # Usage <-> Context 映射
├── CarAudioZonesHelper.java   # 音区配置解析
├── CarVolumeGroup.java        # 音量组管理
├── CarAudioZone.java          # 音区管理
├── CarAudioDeviceInfo.java    # 设备信息
├── AudioFocusHelper.java      # 焦点辅助类
├── AudioRoutingUtils.java     # 路由工具类
├── CoreAudioRoutingUtils.java # 核心路由工具
├── AudioAttributesWrapper.java # AudioAttributes 包装
├── CarAudioUtils.java         # 工具类
├── AudioControlWrapper.java   # AudioControl HAL 包装
├── AudioControlWrapperAidl.java # AIDL 实现
├── AudioControlWrapperV1.java # HIDL v1 实现
├── AudioControlWrapperV2.java # HIDL v2 实现
├── HalAudioFocus.java         # HAL 焦点管理
├── HalFocusListener.java      # HAL 焦点监听
└── ICarVolumeCallback.aidl    # 音量回调接口

配置文件:
packages/services/Car/service/res/raw/car_audio_configuration.xml

职责:
- 解析 car_audio_configuration.xml
- 将应用 Usage 映射到物理 Bus 地址
- 多音区管理
- 通过 AudioControl HAL 处理硬件增益
- Ducking/Muting 策略执行
```

---

## 4. JNI 层

### 4.1 libmedia_jni.so

```
路径: frameworks/base/media/jni/

核心文件:
├── android_media_MediaPlayer.cpp   # MediaPlayer JNI
├── android_media_MediaRecorder.cpp # MediaRecorder JNI
├── android_media_MediaMetadataRetriever.cpp
├── android_media_ResampleInputStream.cpp
└── android_media_Utils.cpp

编译:
mmm frameworks/base/media/jni
```

### 4.2 libandroid_runtime.so

```
路径: frameworks/base/core/jni/

音频相关文件:
├── android_media_AudioTrack.cpp    # AudioTrack JNI
├── android_media_AudioRecord.cpp   # AudioRecord JNI
├── android_media_AudioSystem.cpp   # AudioSystem JNI
├── android_media_AudioEffect.cpp   # 音效 JNI
├── android_media_AudioManager.cpp  # AudioManager JNI
├── android_media_MediaScanner.cpp
└── android_media_SoundPool.cpp

职责:
- 将 Java 调用转换为 Native 结构
- Android 16 中，JNI 直接映射 AIDL 生成的 C++ 类型
```

---

## 5. Native 框架层

### 5.1 libaudioclient.so

```
路径: frameworks/av/media/libaudioclient/

核心文件:
├── AudioTrack.cpp              # 音频播放客户端
├── AudioRecord.cpp             # 音频录制客户端
├── AudioSystem.cpp             # 音频系统客户端
├── AudioEffect.cpp             # 音效客户端
├── IAudioFlinger.cpp           # AudioFlinger Binder 代理
├── IAudioFlingerClient.cpp     # AudioFlinger 客户端回调
├── IAudioPolicyService.cpp     # AudioPolicy Binder 代理
├── IAudioPolicyServiceClient.cpp
├── AudioPolicyServiceClient.cpp
├── PlayerBase.cpp              # 播放器基类
├── RecorderBase.cpp            # 录制器基类
├── AudioAttributes.cpp         # 音频属性
├── AudioDescriptor.cpp         # 音频描述符
├── AudioDeviceType.cpp         # 设备类型
├── AudioTimestamp.cpp          # 时间戳
└── TrackPlayerBase.cpp         # Track 播放器

职责:
- 提供 Native API 给 JNI 层调用
- 通过 Binder IPC 与 AudioFlinger 通信
```

### 5.2 libmedia.so

```
路径: frameworks/av/media/libmedia/

核心文件:
├── mediaplayer.cpp             # 媒体播放器
├── mediarecorder.cpp           # 媒体录制器
├── IMediaPlayer.cpp            # MediaPlayer Binder 代理
├── IMediaRecorder.cpp          # MediaRecorder Binder 代理
├── IMediaPlayerService.cpp     # MediaPlayerService Binder 代理
├── IMediaDeathNotifier.cpp     # 死亡通知
├── MediaPlayerService.cpp      # 媒体播放服务
└── MediaRecorderClient.cpp     # 媒体录制客户端
```

### 5.3 libaudioprocessing.so

```
路径: frameworks/av/media/libaudioprocessing/

核心文件:
├── AudioMixer.cpp              # 音频混音器
├── AudioMixerBase.cpp          # 混音器基类
├── AudioResampler.cpp          # 音频重采样器
├── AudioResamplerCubic.cpp     # Cubic 重采样
├── AudioResamplerSinc.cpp      # Sinc 重采样
├── AudioResamplerDynamic.cpp   # 动态重采样
├── AudioBufferProvider.cpp     # 缓冲区提供者
└── BufferProviders.cpp         # 缓冲区提供者集合

职责:
- 多路音频混音
- 采样率转换
- 音量应用
```

### 5.4 libaaudio.so / libaaudio_internal.so

```
路径: frameworks/av/media/libaaudio/

目录结构:
├── examples/                   # 示例代码
├── include/                    # 头文件
├── src/
│   ├── core/                   # 核心实现
│   │   ├── AAudioAudio.cpp
│   │   ├── AudioStream.cpp
│   │   ├── AudioStreamBuilder.cpp
│   │   ├── AAudioStreamParameters.cpp
│   │   └── AAudioVersion.cpp
│   ├── client/                 # 客户端实现
│   │   ├── AudioEndpoint.cpp
│   │   ├── AudioStreamInternal.cpp
│   │   ├── AudioStreamInternalPlay.cpp
│   │   ├── AudioStreamInternalCapture.cpp
│   │   └── AudioStreamInternal.cpp
│   ├── binding/                # Binder 绑定
│   │   ├── AAudioBinderClient.cpp
│   │   └── AAudioBinderAdapter.cpp
│   └── flowgraph/              # 数据流图
│       ├── AudioSourceCaller.cpp
│       └── FlowGraphNode.cpp

职责:
- 高性能音频 API
- mmap 低延迟模式支持
- 独占/共享模式
```

---

## 6. Audio 服务层

### 6.1 AudioFlinger

```
路径: frameworks/av/services/audioflinger/

核心文件:
├── AudioFlinger.cpp            # AudioFlinger 主类
├── AudioFlinger.h
├── Threads.cpp                 # 播放/录音线程
├── Threads.h
├── Tracks.cpp                  # 音频轨道
├── Tracks.h
├── FastThread.cpp              # 快速线程
├── FastMixer.cpp               # 快速混音器
├── FastCapture.cpp             # 快速捕获器
├── FastMixerState.cpp          # 快速混音状态
├── AudioStreamOut.cpp          # 输出流
├── AudioStreamIn.cpp           # 输入流
├── AudioHwDevice.cpp           # HAL 设备管理
├── AudioHwDevice.h
├── Effects.cpp                 # 音效处理
├── Effects.h
├── PatchPanel.cpp              # Patch 面板
├── PatchPanel.h
├── AudioWatchdog.cpp           # 看门狗
├── AudioWatchdog.h
├── SpdifStreamOut.cpp          # S/PDIF 输出
├── AAudioService.cpp           # AAudio 服务 (部分)
└── ServiceUtilities.cpp        # 服务工具

编译输出:
libaudioflinger.so

职责:
- 管理音频流
- 混音处理
- 音效处理
- 与 HAL 层交互
```

### 6.2 AudioPolicyService

```
路径: frameworks/av/services/audiopolicy/

核心文件:
├── service/
│   ├── AudioPolicyService.cpp  # AudioPolicy 服务
│   ├── AudioPolicyService.h
│   ├── AudioPolicyClientImpl.cpp
│   └── AudioPolicyClientImpl.h
├── managerdefault/
│   ├── AudioPolicyManager.cpp  # 默认策略管理器
│   └── AudioPolicyManager.h
├── enginedefault/
│   └── src/
│       ├── Engine.cpp          # 默认引擎
│       ├── Engine.h
│       ├── ProductStrategy.cpp
│       └── VolumeGroup.cpp
├── common/
│   ├── AudioPolicyManagerBase.cpp
│   └── AudioPolicyManagerBase.h
└── config/
    ├── AudioPolicyConfig.cpp   # 配置解析
    └── AudioPolicyConfig.h

编译输出:
libaudiopolicyservice.so

职责:
- 音频路由策略
- 设备选择
- 音量曲线管理
- 音频焦点管理
```

### 6.3 AAudioService / OboeService

```
路径: frameworks/av/services/oboeservice/

核心文件:
├── AAudioService.cpp           # AAudio 服务主类
├── AAudioService.h
├── AAudioServiceStreamBase.cpp # 流基类
├── AAudioServiceStreamShared.cpp # 共享流
├── AAudioServiceStreamMMAP.cpp # mmap 流
├── AAudioServiceEndpoint.cpp   # 端点基类
├── AAudioServiceEndpointPlay.cpp # 播放端点
├── AAudioServiceEndpointCapture.cpp # 录制端点
├── AAudioServiceEndpointMMAP.cpp # mmap 端点
├── AAudioServiceEndpointShared.cpp # 共享端点
├── AAudioClientTracker.cpp     # 客户端追踪
├── AAudioStreamTracker.cpp     # 流追踪
├── AAudioMixer.cpp             # 混音器
├── AAudioThread.cpp            # 线程管理
└── SharedMemoryProxy.cpp       # 共享内存代理

编译输出:
libaaudioservice.so

职责:
- AAudio 高性能音频服务
- mmap 模式支持
- 独占模式流管理
```

---

## 7. HAL 适配层

### 7.1 libaudiohal.so

```
路径: frameworks/av/media/libaudiohal/

核心文件:
├── DevicesFactoryHalInterface.cpp  # 设备工厂接口
├── DevicesFactoryHalInterface.h
├── EffectsFactoryHalInterface.cpp  # 音效工厂接口
├── EffectsFactoryHalInterface.h
├── ConversionHelperAidl.cpp         # AIDL 转换辅助
├── impl/
│   ├── DevicesFactoryHalHidl.cpp    # HIDL 实现
│   ├── DevicesFactoryHalAidl.cpp    # AIDL 实现
│   ├── DeviceHalHidl.cpp            # HIDL 设备
│   ├── DeviceHalAidl.cpp            # AIDL 设备
│   ├── StreamOutHalHidl.cpp         # HIDL 输出流
│   ├── StreamOutHalAidl.cpp         # AIDL 输出流
│   ├── StreamInHalHidl.cpp          # HIDL 输入流
│   ├── StreamInHalAidl.cpp          # AIDL 输入流
│   ├── EffectsFactoryHalHidl.cpp    # HIDL 音效
│   └── EffectsFactoryHalAidl.cpp    # AIDL 音效

职责:
- 将 Framework 内部逻辑转换为对底层 HAL 的调用
- 支持 HIDL 和 AIDL 两种 HAL 接口
```

---

## 8. AIDL/HIDL 接口层

### 8.1 Standard Audio HAL

```
路径: hardware/interfaces/audio/aidl/

核心接口:
├── IModule.aidl                # 模块接口 (主入口)
├── IDevice.aidl                # 设备接口
├── IStreamOut.aidl             # 输出流接口
├── IStreamIn.aidl              # 输入流接口
├── IStream.aidl                # 流基类接口
├── IConfig.aidl                # 配置接口
├── IEffect.aidl                # 音效接口
├── IEffectsFactory.aidl        # 音效工厂接口
├── types.aidl                  # 类型定义
├── AudioConfig.aidl            # 音频配置
├── AudioDevice.aidl            # 音频设备
├── AudioFormat.aidl            # 音频格式
├── AudioChannelMask.aidl       # 声道掩码
└── AudioPort.aidl              # 音频端口

Android 16 标准:
- 定义了物理音频端口、跳线 (Patch) 和流控制的硬性规范
- 替代了旧的 HIDL 接口
```

### 8.2 AudioControl HAL (AAOS)

```
路径: hardware/interfaces/automotive/audiocontrol/aidl/

核心接口:
├── IAudioControl.aidl          # 主接口
├── IFocusListener.aidl         # 焦点监听
├── IAudioGainCallback.aidl     # 增益回调
├── AudioFocusChange.aidl       # 焦点变化类型
├── DuckingInfo.aidl            # Ducking 信息
├── MutingInfo.aidl             # 静音信息
├── AudioGainConfigInfo.aidl    # 增益配置
└── Reasons.aidl                # 原因枚举

AAOS 特有:
- 处理多音区音频避让 (Ducking)
- Mute 状态同步
- 硬件 Gain 映射
```

---

## 9. Vendor 实现层

### 9.1 QCOM Primary HAL

```
路径: vendor/qcom/opensource/audio-hal-ar/primary-hal/

核心目录:
├── hal-pal/
│   ├── android.hardware.audio.service_64.rc
│   ├── AudioDevice.cpp         # 设备实现
│   ├── AudioStream.cpp         # 流实现
│   ├── AudioVoice.cpp          # 通话实现
│   └── zeekr_audio/            # 车厂定制
├── configs/
│   ├── msmnile_au/
│   │   ├── card-defs.xml       # 声卡定义
│   │   └── mixer_paths_*.xml   # Mixer 路径
│   └── common_au/
│       └── audio_policy_configuration.xml
└── Android.mk / Android.bp

编译输出:
android.hardware.audio.service_64
```

### 9.2 PAL (Audio Reach)

```
路径: vendor/qcom/opensource/pal/

核心文件:
├── pal_defs.h                  # PAL 定义
├── PalDefs.h
├── Stream.cpp                  # 流管理
├── Device.cpp                  # 设备管理
├── Session.cpp                 # 会话管理
├── ResourceManager.cpp         # 资源管理器
├── SpeakerProtection.cpp       # 扬声器保护
└── soundtrigger/               # 语音触发

配置文件:
/vendor/etc/audio_ar/
├── audio_policy_configuration.xml
├── usecaseKvManager.xml
├── resourcemanager_gvmauto8295_adp_star.xml
└── card-defs.xml

编译输出:
libar-pal.so
```

### 9.3 AGM (Audio Graph Manager)

```
路径: vendor/qcom/opensource/agm/

核心文件:
├── ipc/
│   └── HwBinders/
│       └── agm_ipc_service/    # IPC 服务
├── service/
│   └── agm_service.cpp         # AGM 服务
└── plugins/                    # 插件

编译输出:
libagm.so
agm_ipc_service
```

### 9.4 GSL (Graph Service Library)

```
路径: vendor/qcom/proprietary/args/gsl/

职责:
- 音频图管理
- DSP 模块配置
- Codec 配置
```

---

## 10. Audio Base 定义

```
路径: system/media/audio/include/system/

核心文件:
├── audio.h                     # 音频基础定义
├── audio-base.h                # 基础类型 (自动生成)
├── audio-base-utils.h          # 工具函数
├── audio-hal-enums.h           # HAL 枚举定义
├── audio_effect.h              # 音效定义
├── audio_policy.h              # 音频策略定义
└── sound_trigger.h             # 语音触发定义

关键定义:
- audio_stream_type_t           # 流类型
- audio_usage_t                 # 使用场景
- audio_content_type_t          # 内容类型
- audio_device_t                # 设备类型
- audio_format_t                # 格式类型
- audio_channel_mask_t          # 声道掩码
- audio_output_flags_t          # 输出标志
- audio_input_flags_t           # 输入标志
```

---

## 11. 配置文件路径

### 11.1 标准配置

```
# Audio Policy 配置
/vendor/etc/audio_policy_configuration.xml
/system/etc/audio_policy_configuration.xml

# Audio Effects 配置
/vendor/etc/audio_effects.conf
/system/etc/audio_effects.conf
/vendor/etc/audio_effects.xml
/system/etc/audio_effects.xml

# Audio Policy Engine 配置
/vendor/etc/audio_policy_engine_configuration.xml
```

### 11.2 AAOS 配置

```
# 车载音频配置
/vendor/etc/car_audio_configuration.xml
/packages/services/Car/service/res/raw/car_audio_configuration.xml

# 车辆属性配置
/vendor/etc/automotive/vehicle_properties.xml
```

### 11.3 QCOM 配置

```
# PAL 配置
/vendor/etc/audio_ar/audio_policy_configuration.xml
/vendor/etc/audio_ar/usecaseKvManager.xml
/vendor/etc/audio_ar/resourcemanager_*.xml

# Mixer 配置
/vendor/etc/mixer_paths_*.xml
/vendor/etc/card-defs.xml
```

---

## 📌 总结

| 层级 | 路径 | 核心库/服务 |
|------|------|------------|
| **Java 应用** | frameworks/base/media/java/android/media/ | framework.jar |
| **Car Media** | packages/services/Car/car-lib/src/android/car/media/ | CarService.apk |
| **Java 服务** | frameworks/base/services/core/java/com/android/server/audio/ | services.jar |
| **CarAudioService** | packages/services/Car/service/src/com/android/car/audio/ | CarService.apk |
| **JNI** | frameworks/base/core/jni/ | libandroid_runtime.so |
| **Native 框架** | frameworks/av/media/libaudioclient/ | libaudioclient.so |
| **AudioFlinger** | frameworks/av/services/audioflinger/ | libaudioflinger.so |
| **AudioPolicy** | frameworks/av/services/audiopolicy/ | libaudiopolicyservice.so |
| **AAudio** | frameworks/av/services/oboeservice/ | libaaudioservice.so |
| **HAL 适配** | frameworks/av/media/libaudiohal/ | libaudiohal.so |
| **AIDL 接口** | hardware/interfaces/audio/aidl/ | android.hardware.audio |
| **AudioControl** | hardware/interfaces/automotive/audiocontrol/aidl/ | android.hardware.automotive.audiocontrol |
| **Vendor HAL** | vendor/qcom/opensource/audio-hal-ar/ | android.hardware.audio.service |
| **PAL** | vendor/qcom/opensource/pal/ | libar-pal.so |
| **Audio Base** | system/media/audio/include/system/ | libaudioutils |
