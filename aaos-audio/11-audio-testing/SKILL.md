---
name: "audio-testing"
description: "Android 音频测试与验证规范，涵盖 VTS 测试、CTS 测试、单元测试、自动化测试与音频环路测试"
version: "1.0.0"
triggers: ["VTS", "CTS", "GTest", "audio test", "unit test", "loopback", "audio verification", "test case", "mock", "fuzzer"]
---

> 参考来源：Android AOSP Testing Documentation

# 🧪 Android 音频测试与验证规范

---

## 1. 测试体系概述

### 1.1 测试层级

```
┌─────────────────────────────────────────────────────────────────┐
│                     Android Audio Testing                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  CTS (Compatibility Test Suite)                             ││
│  │  - 兼容性认证测试                                           ││
│  │  - 必须全部通过                                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                          ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  VTS (Vendor Test Suite)                                    ││
│  │  - HAL 接口测试                                             ││
│  │  - 验证厂商实现                                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                          ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Unit Tests (GTest)                                         ││
│  │  - 模块单元测试                                             ││
│  │  - 开发阶段验证                                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                          ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Manual / Integration Tests                                 ││
│  │  - 功能验证                                                 ││
│  │  - 性能测试                                                 ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 测试路径

```
# CTS 测试
cts/tests/tests/media/

# VTS 测试
hardware/interfaces/audio/aidl/vts/

# 单元测试
frameworks/av/services/audioflinger/tests/
frameworks/av/services/audiopolicy/tests/

# Audio HAL 测试
hardware/interfaces/audio/core/all-versions/default/tests/
```

---

## 2. VTS 测试

### 2.1 VTS 概述

```
VTS (Vendor Test Suite) 用于验证 HAL 实现的正确性
- 测试 HAL 接口契约
- 验证参数边界
- 检查返回值正确性
- 自动化测试框架
```

### 2.2 Audio HAL VTS 测试

```cpp
// hardware/interfaces/audio/aidl/vts/AudioCoreTests.cpp

#include <aidl/android/hardware/audio/core/IModule.h>
#include <gtest/gtest.h>

using aidl::android::hardware::audio::core::IModule;
using aidl::android::hardware::audio::core::IDevice;
using aidl::android::hardware::audio::core::IStreamOut;

class AudioCoreTest : public ::testing::TestWithParam<std::string> {
protected:
    void SetUp() override {
        std::string instance = GetParam();
        module = IModule::fromBinder(
            ndk::SpAIBinder(AServiceManager_waitForService(instance.c_str()))
        );
        ASSERT_NE(module, nullptr);
    }
    
    std::shared_ptr<IModule> module;
};

// 测试模块打开
TEST_P(AudioCoreTest, OpenModule) {
    int moduleId;
    auto status = module->getModuleId(&moduleId);
    ASSERT_TRUE(status.isOk());
    ASSERT_GT(moduleId, 0);
}

// 测试打开设备
TEST_P(AudioCoreTest, OpenDevice) {
    std::shared_ptr<IDevice> device;
    auto status = module->openDevice("primary", &device);
    ASSERT_TRUE(status.isOk());
    ASSERT_NE(device, nullptr);
}

// 测试打开输出流
TEST_P(AudioCoreTest, OpenOutputStream) {
    AudioConfig config = {
        .sampleRate = 48000,
        .channelMask = AudioChannelMask::OUT_STEREO,
        .format = AudioFormat::PCM_16_BIT,
    };
    
    std::shared_ptr<IStreamOut> stream;
    auto status = module->openOutputStream("primary", config, 
            AudioSource::SYS_RESERVED_NONE, 
            AudioOutputFlag::NONE, &stream);
    
    ASSERT_TRUE(status.isOk());
    ASSERT_NE(stream, nullptr);
}

// 测试参数边界
TEST_P(AudioCoreTest, OpenOutputStreamInvalidConfig) {
    AudioConfig config = {
        .sampleRate = 0,  // 无效采样率
        .channelMask = AudioChannelMask::INVALID,
        .format = AudioFormat::INVALID,
    };
    
    std::shared_ptr<IStreamOut> stream;
    auto status = module->openOutputStream("primary", config,
            AudioSource::SYS_RESERVED_NONE,
            AudioOutputFlag::NONE, &stream);
    
    // 应该返回错误
    EXPECT_FALSE(status.isOk());
}

INSTANTIATE_TEST_SUITE_P(
    AudioCoreTests,
    AudioCoreTest,
    ::testing::Values("android.hardware.audio.core.IModule/default")
);
```

### 2.3 AudioControl HAL VTS 测试

```cpp
// hardware/interfaces/automotive/audiocontrol/aidl/vts/AudioControlTest.cpp

#include <aidl/android/hardware/automotive/audiocontrol/IAudioControl.h>
#include <gtest/gtest.h>

using aidl::android::hardware::automotive::audiocontrol::IAudioControl;
using aidl::android::hardware::automotive::audiocontrol::AudioFocusChange;

class AudioControlTest : public ::testing::Test {
protected:
    void SetUp() override {
        audioControl = IAudioControl::fromBinder(
            ndk::SpAIBinder(AServiceManager_waitForService(
                "android.hardware.automotive.audiocontrol.IAudioControl/default"))
        );
        ASSERT_NE(audioControl, nullptr);
    }
    
    std::shared_ptr<IAudioControl> audioControl;
};

// 测试音量设置
TEST_F(AudioControlTest, SetGroupVolume) {
    auto status = audioControl->setGroupVolume(0, 50);
    EXPECT_TRUE(status.isOk());
    
    int32_t volume;
    status = audioControl->getGroupVolume(0, &volume);
    EXPECT_TRUE(status.isOk());
    EXPECT_EQ(50, volume);
}

// 测试静音
TEST_F(AudioControlTest, SetGroupMute) {
    auto status = audioControl->setGroupMute(0, true);
    EXPECT_TRUE(status.isOk());
    
    bool muted;
    status = audioControl->isGroupMuted(0, &muted);
    EXPECT_TRUE(status.isOk());
    EXPECT_TRUE(muted);
}

// 测试平衡
TEST_F(AudioControlTest, SetBalance) {
    auto status = audioControl->setBalanceTowardRight(0.5f);
    EXPECT_TRUE(status.isOk());
}

// 测试焦点变化
TEST_F(AudioControlTest, OnAudioFocusChange) {
    auto status = audioControl->onAudioFocusChange(
        "MEDIA", 0, AudioFocusChange::GAIN);
    EXPECT_TRUE(status.isOk());
}
```

### 2.4 运行 VTS 测试

```bash
# 运行所有 Audio HAL VTS 测试
vts-tradefed run vts -m VtsAidlAudioCoreTargetTest

# 运行 AudioControl VTS 测试
vts-tradefed run vts -m VtsAidlAudioControlTargetTest

# 运行特定测试用例
vts-tradefed run vts -m VtsAidlAudioCoreTargetTest -t AudioCoreTest.OpenModule

# 查看测试结果
vts-tradefed list results
```

---

## 3. CTS 测试

### 3.1 CTS 概述

```
CTS (Compatibility Test Suite) 用于验证设备兼容性
- 必须全部通过才能获得认证
- 包含大量音频相关测试
- 测试 API 行为正确性
```

### 3.2 音频相关 CTS 测试

```
cts/tests/tests/media/src/android/media/cts/

├── AudioTrackTest.java          # AudioTrack API 测试
├── AudioRecordTest.java         # AudioRecord API 测试
├── AudioManagerTest.java        # AudioManager API 测试
├── AudioFocusTest.java          # 音频焦点测试
├── AudioDeviceTest.java         # 音频设备测试
├── AudioFormatTest.java         # 音频格式测试
├── AudioAttributesTest.java     # AudioAttributes 测试
└── AudioPlaybackConfigurationTest.java
```

### 3.3 运行 CTS 测试

```bash
# 下载 CTS 包
# https://source.android.com/compatibility/cts/downloads

# 解压并运行
cd android-cts
./tools/cts-tradefed

# 运行音频相关测试
run cts -m CtsMediaAudioTestCases

# 运行特定测试
run cts -m CtsMediaAudioTestCases -t android.media.cts.AudioTrackTest

# 运行所有测试
run cts

# 查看测试计划
list plans
```

### 3.4 CTS Verifier

```
CTS Verifier 用于测试无法自动化的功能
- 需要人工参与验证
- 包含音频环路测试
- 包含延迟测试

# 安装 CTS Verifier
adb install CtsVerifier.apk

# 运行音频测试
# 1. 打开 CTS Verifier 应用
# 2. 选择 Audio Tests
# 3. 按提示进行测试
```

---

## 4. 单元测试 (GTest)

### 4.1 GTest 框架

```cpp
#include <gtest/gtest.h>
#include <gmock/gmock.h>

// 测试夹具
class AudioFlingerTest : public ::testing::Test {
protected:
    void SetUp() override {
        // 测试前初始化
    }
    
    void TearDown() override {
        // 测试后清理
    }
};

// 简单测试
TEST(AudioFlingerTest, SampleRateConversion) {
    EXPECT_EQ(48000, convertSampleRate(48000));
    EXPECT_EQ(44100, convertSampleRate(44100));
}

// 参数化测试
class SampleRateTest : public ::testing::TestWithParam<int> {};

TEST_P(SampleRateTest, IsValid) {
    int rate = GetParam();
    EXPECT_TRUE(isValidSampleRate(rate));
}

INSTANTIATE_TEST_SUITE_P(
    ValidSampleRates,
    SampleRateTest,
    ::testing::Values(8000, 16000, 44100, 48000, 96000)
);
```

### 4.2 Mock 对象

```cpp
#include <gmock/gmock.h>

// Mock 类
class MockAudioHAL : public IAudioHAL {
public:
    MOCK_METHOD(int, openOutputStream, (const AudioConfig&), (override));
    MOCK_METHOD(int, closeOutputStream, (int), (override));
    MOCK_METHOD(int, write, (int, const void*, size_t), (override));
};

// 使用 Mock
TEST(AudioFlingerTest, OpenOutputStreamSuccess) {
    MockAudioHAL mockHal;
    
    EXPECT_CALL(mockHal, openOutputStream(testing::_))
        .Times(1)
        .WillOnce(testing::Return(0));
    
    AudioFlinger flinger(&mockHal);
    int result = flinger.openStream();
    
    EXPECT_EQ(0, result);
}

// 测试错误情况
TEST(AudioFlingerTest, OpenOutputStreamFailure) {
    MockAudioHAL mockHal;
    
    EXPECT_CALL(mockHal, openOutputStream(testing::_))
        .Times(1)
        .WillOnce(testing::Return(-ENODEV));
    
    AudioFlinger flinger(&mockHal);
    int result = flinger.openStream();
    
    EXPECT_EQ(-ENODEV, result);
}
```

### 4.3 运行单元测试

```bash
# 编译测试
mmm frameworks/av/services/audioflinger/tests/

# 推送到设备
adb push out/target/product/<device>/data/nativetest/audioflinger_tests/audioflinger_tests /data/local/tmp/

# 运行测试
adb shell /data/local/tmp/audioflinger_tests

# 运行特定测试
adb shell /data/local/tmp/audioflinger_tests --gtest_filter=AudioFlingerTest.*

# 生成 XML 报告
adb shell /data/local/tmp/audioflinger_tests --gtest_output=xml:/data/local/tmp/results.xml
```

---

## 5. 自动化测试

### 5.1 Python 测试脚本

```python
#!/usr/bin/env python3
# audio_test.py

import subprocess
import time
import wave
import numpy as np

class AudioTest:
    def __init__(self, device_id=None):
        self.device_id = device_id
        self.adb_prefix = ['adb']
        if device_id:
            self.adb_prefix.extend(['-s', device_id])
    
    def adb(self, cmd):
        """执行 adb 命令"""
        full_cmd = self.adb_prefix + cmd.split()
        result = subprocess.run(full_cmd, capture_output=True, text=True)
        return result.returncode, result.stdout, result.stderr
    
    def play_audio(self, file_path, duration=5):
        """播放音频文件"""
        # 推送文件
        self.adb(f'push {file_path} /data/local/tmp/')
        
        # 播放
        filename = file_path.split('/')[-1]
        self.adb(f'shell tinyplay /data/local/tmp/{filename}')
    
    def record_audio(self, output_path, duration=5):
        """录音"""
        cmd = f'shell tinycap /data/local/tmp/record.wav -r 48000 -c 2 -b 16 -T {duration}'
        self.adb(cmd)
        
        # 拉取文件
        self.adb(f'pull /data/local/tmp/record.wav {output_path}')
    
    def check_audio_devices(self):
        """检查音频设备"""
        _, stdout, _ = self.adb('shell cat /proc/asound/cards')
        print("Audio devices:")
        print(stdout)
        return stdout
    
    def check_mixer_controls(self):
        """检查 Mixer 控件"""
        _, stdout, _ = self.adb('shell tinymix -D 0')
        print("Mixer controls:")
        print(stdout)
        return stdout
    
    def set_volume(self, control, value):
        """设置音量"""
        self.adb(f'shell tinymix "{control}" {value}')
    
    def measure_latency(self):
        """测量延迟"""
        # 使用 Audio Latency Test
        self.adb('shell am instrument -w -e class com.android.cts.verifier.audio.AudioLatencyTest '
                 'com.android.cts.verifier/.AudioTestActivity')
    
    def run_loopback_test(self):
        """环路测试"""
        # 录制播放的音频
        self.adb('shell tinycap /data/local/tmp/loopback.wav -r 48000 -c 2 -b 16 -T 5 &')
        time.sleep(0.5)
        
        # 播放测试信号
        self.adb('shell tinyplay /data/local/tmp/test_tone.wav')
        
        time.sleep(6)
        
        # 分析录制的音频
        self.adb('pull /data/local/tmp/loopback.wav .')
        
        # 检查是否录制到音频
        return self.analyze_audio('loopback.wav')
    
    def analyze_audio(self, file_path):
        """分析音频文件"""
        with wave.open(file_path, 'rb') as wf:
            frames = wf.readframes(wf.getnframes())
            data = np.frombuffer(frames, dtype=np.int16)
            
            # 计算能量
            energy = np.sqrt(np.mean(data**2))
            print(f"Audio energy: {energy}")
            
            return energy > 1000  # 阈值


def main():
    test = AudioTest()
    
    print("=== Audio Device Check ===")
    test.check_audio_devices()
    
    print("\n=== Mixer Controls ===")
    test.check_mixer_controls()
    
    print("\n=== Loopback Test ===")
    result = test.run_loopback_test()
    print(f"Loopback test: {'PASS' if result else 'FAIL'}")


if __name__ == '__main__':
    main()
```

### 5.2 Shell 测试脚本

```bash
#!/bin/bash
# audio_smoke_test.sh

echo "=== Audio Smoke Test ==="

# 1. 检查音频设备
echo "1. Checking audio devices..."
adb shell cat /proc/asound/cards

# 2. 检查 PCM 设备
echo "2. Checking PCM devices..."
adb shell cat /proc/asound/pcm

# 3. 检查 Mixer
echo "3. Checking mixer controls..."
adb shell tinymix

# 4. 播放测试
echo "4. Testing playback..."
adb shell "tinyplay /vendor/etc/test_tone.wav" &
sleep 2

# 5. 录音测试
echo "5. Testing capture..."
adb shell "tinycap /data/local/tmp/test_record.wav -r 48000 -c 2 -b 16 -T 3"
sleep 4

# 6. 检查录音文件
echo "6. Checking recorded file..."
adb shell "ls -la /data/local/tmp/test_record.wav"

# 7. 检查 AudioFlinger 状态
echo "7. Checking AudioFlinger..."
adb shell dumpsys audio | head -50

# 8. 检查音频焦点
echo "8. Checking audio focus..."
adb shell dumpsys audio | grep -A 10 "Audio Focus"

echo "=== Test Complete ==="
```

---

## 6. 音频环路测试

### 6.1 硬件环路测试

```
┌─────────────────────────────────────────────────────────────────┐
│                   Hardware Loopback Test                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────┐      ┌─────────┐      ┌─────────┐                 │
│  │  音频线 │ ───→ │  Codec  │ ───→ │  ADC    │                 │
│  │ (环路)  │      │         │      │         │                 │
│  └─────────┘      └─────────┘      └────┬────┘                 │
│       ↑                                 │                       │
│       │                                 ↓                       │
│  ┌────┴────┐      ┌─────────┐      ┌─────────┐                 │
│  │  DAC    │ ←─── │  Codec  │ ←─── │  DSP    │                 │
│  │         │      │         │      │         │                 │
│  └─────────┘      └─────────┘      └─────────┘                 │
│                                                                 │
│  测试步骤:                                                      │
│  1. 用音频线连接 Line-out 和 Line-in                           │
│  2. 播放测试信号 (正弦波)                                       │
│  3. 同步录音                                                   │
│  4. 分析录制信号                                               │
│  5. 计算延迟、频率响应、THD 等                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 软件环路测试

```bash
# 使用 snd-aloop 内核模块
adb shell modprobe snd-aloop

# 播放到 loopback 设备
adb shell "aplay -D hw:Loopback,0,0 test.wav &"

# 从 loopback 设备录音
adb shell "arecord -D hw:Loopback,1,0 -d 5 record.wav"

# 分析延迟
# 比较播放和录音的时间戳
```

---

## 7. 测试报告

### 7.1 测试结果格式

```
┌─────────────────────────────────────────────────────────────────┐
│                     Audio Test Report                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Device: <device_name>                                          │
│  Date: <test_date>                                              │
│  Android Version: <version>                                     │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ Test Summary                                                ││
│  ├─────────────────────────────────────────────────────────────┤│
│  │ Total Tests:  100                                           ││
│  │ Passed:      95                                             ││
│  │ Failed:      3                                              ││
│  │ Skipped:     2                                              ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ Failed Tests                                                 ││
│  ├─────────────────────────────────────────────────────────────┤│
│  │ 1. AudioTrackTest.testFastTrack - XRUN detected             ││
│  │ 2. AudioFocusTest.testDucking - Ducking not applied         ││
│  │ 3. AudioLatencyTest.testRoundTrip - Latency > 50ms          ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📌 总结

| 类别 | 工具/框架 |
|------|----------|
| **VTS 测试** | vts-tradefed, VtsAidlAudioCoreTargetTest |
| **CTS 测试** | cts-tradefed, CtsMediaAudioTestCases |
| **CTS Verifier** | CtsVerifier.apk, 手动测试 |
| **单元测试** | GTest, GMock |
| **自动化测试** | Python, Shell 脚本 |
| **环路测试** | snd-aloop, 硬件环路 |
| **性能测试** | 延迟测试, XRUN 统计 |
