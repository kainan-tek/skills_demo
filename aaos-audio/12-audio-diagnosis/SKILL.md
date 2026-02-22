---
name: "audio-diagnosis"
description: "Android 音频诊断工具集，涵盖 dumpsys 分析、日志过滤、内核调试、性能分析与常见问题定位"
version: "1.0.0"
triggers: ["dumpsys", "logcat", "debug", "diagnosis", "troubleshoot", "ftrace", "systrace", "atrace", "simpleperf", "audio issue", "no sound", "XRUN", "latency"]
---

> 参考来源：Android AOSP Debugging Documentation

# 🔍 Android 音频诊断工具集

---

## 1. 诊断工具概述

### 1.1 工具分类

```
┌─────────────────────────────────────────────────────────────────┐
│                     Audio Diagnosis Tools                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Framework 层                                                   │
│  ├── dumpsys audio - 音频服务状态                               │
│  ├── dumpsys media.audio_flinger - AudioFlinger 状态           │
│  ├── dumpsys media.audio_policy - AudioPolicy 状态             │
│  └── dumpsys car_service - AAOS 车载服务状态                   │
│                                                                 │
│  日志系统                                                       │
│  ├── logcat - 系统日志                                          │
│  ├── dmesg - 内核日志                                           │
│  └── /proc/kmsg - 实时内核日志                                  │
│                                                                 │
│  内核调试                                                       │
│  ├── /proc/asound/ - ALSA 状态                                  │
│  ├── /sys/kernel/debug/asoc/ - ASoC 状态                       │
│  └── ftrace - 内核事件追踪                                      │
│                                                                 │
│  性能分析                                                       │
│  ├── systrace/atrace - 系统追踪                                 │
│  ├── simpleperf - 性能采样                                      │
│  └── perfetto - 现代化追踪工具                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. dumpsys 命令

### 2.1 dumpsys audio

```bash
# 查看完整音频状态
adb shell dumpsys audio

# 输出结构:
# 1. Audio Policy Service 状态
# 2. Audio Flinger 状态
# 3. 活动的 Track 和录音
# 4. 音频设备状态
# 5. 音量设置
# 6. 音频焦点状态
```

### 2.2 关键信息提取

```bash
# 查看活动播放流
adb shell dumpsys audio | grep -A 20 "Output thread"

# 查看活动录音流
adb shell dumpsys audio | grep -A 20 "Input thread"

# 查看音频设备
adb shell dumpsys audio | grep -A 30 "Devices"

# 查看音量设置
adb shell dumpsys audio | grep -A 10 "Volumes"

# 查看音频焦点
adb shell dumpsys audio | grep -A 20 "Audio Focus"

# 查看 XRUN 统计
adb shell dumpsys audio | grep -i "underrun\|overrun"

# 查看延迟信息
adb shell dumpsys audio | grep -i latency
```

### 2.3 AudioFlinger 详细状态

```bash
# 查看 AudioFlinger 状态
adb shell dumpsys media.audio_flinger

# 查看线程状态
adb shell dumpsys media.audio_flinger | grep -A 50 "Thread"

# 查看 Track 状态
adb shell dumpsys media.audio_flinger | grep -A 30 "Active Track"

# 查看效果器状态
adb shell dumpsys media.audio_flinger | grep -A 20 "Effect"
```

### 2.4 AudioPolicy 状态

```bash
# 查看 AudioPolicy 状态
adb shell dumpsys media.audio_policy

# 查看路由策略
adb shell dumpsys media.audio_policy | grep -A 30 "Strategy"

# 查看设备连接状态
adb shell dumpsys media.audio_policy | grep -A 20 "Connected"
```

### 2.5 AAOS 车载服务

```bash
# 查看 CarAudioService 状态
adb shell dumpsys car_service | grep -A 100 "CarAudio"

# 查看音区配置
adb shell dumpsys car_service | grep -A 50 "AudioZone"

# 查看音量组
adb shell dumpsys car_service | grep -A 30 "VolumeGroup"

# 查看音频焦点
adb shell dumpsys car_service | grep -A 20 "AudioFocus"
```

---

## 3. logcat 日志分析

### 3.1 基本用法

```bash
# 实时日志
adb logcat

# 清除日志缓冲区
adb logcat -c

# 查看特定标签
adb logcat -s AudioFlinger AudioPolicyService

# 查看特定进程
adb logcat --pid=$(adb shell pidof audioserver)

# 保存到文件
adb logcat -d > audio_log.txt
```

### 3.2 音频相关标签

```bash
# Framework 层标签
adb logcat -s AudioFlinger AudioPolicyService AudioTrack AudioRecord AudioManager

# HAL 层标签
adb logcat -s AudioHAL AudioA2DP AudioUSB

# 内核层标签
adb logcat -s ALSA ASoC

# AAOS 标签
adb logcat -s CarAudioService AudioControl

# 组合多个标签
adb logcat -s AudioFlinger:V AudioPolicyService:V AudioTrack:V
```

### 3.3 高级过滤

```bash
# 按优先级过滤
adb logcat *:W  # Warning 及以上

# 按正则表达式过滤
adb logcat | grep -E "Audio|Sound|PCM"

# 过滤 XRUN 相关日志
adb logcat | grep -iE "underrun|overrun|xrun"

# 过滤设备切换日志
adb logcat | grep -iE "device|route|switch"

# 过滤焦点日志
adb logcat | grep -iE "focus|duck|mute"

# 时间范围过滤
adb logcat -t '02-16 14:00:00.000' -T '02-16 14:30:00.000'
```

### 3.4 日志格式化

```bash
# 详细格式
adb logcat -v time

# 线程时间格式
adb logcat -v threadtime

# 紧凑格式
adb logcat -v brief

# JSON 格式
adb logcat -v json
```

---

## 4. 内核调试

### 4.1 /proc/asound/ 目录

```bash
# 列出声卡
adb shell cat /proc/asound/cards

# 列出 PCM 设备
adb shell cat /proc/asound/pcm

# 查看声卡信息
adb shell cat /proc/asound/card0/id

# 查看 PCM 状态
adb shell cat /proc/asound/card0/pcm0p/sub0/status

# 查看硬件参数
adb shell cat /proc/asound/card0/pcm0p/sub0/hw_params

# 查看软件参数
adb shell cat /proc/asound/card0/pcm0p/sub0/sw_params

# 查看支持的格式
adb shell cat /proc/asound/card0/pcm0p/sub0/info
```

### 4.2 状态字段解析

```
# /proc/asound/card0/pcm0p/sub0/status 输出示例

state: RUNNING              # PCM 状态
trigger_time: 1234567890.123456  # 触发时间
tstamp: 1234567890.123456   # 当前时间戳
delay: 1024                 # 延迟帧数
avail: 2048                 # 可用帧数
avail_max: 4096             # 最大可用帧数
hw_ptr: 102400              # 硬件指针
appl_ptr: 103424            # 应用指针
```

### 4.3 ASoC 调试

```bash
# 查看 DAPM 状态
adb shell cat /sys/kernel/debug/asoc/card0/dapm

# 查看 DAI 链接
adb shell cat /sys/kernel/debug/asoc/card0/dai

# 查看组件状态
adb shell cat /sys/kernel/debug/asoc/card0/components

# 启用 ASoC 调试
adb shell "echo 1 > /sys/module/snd_soc_core/parameters/debug"
```

### 4.4 ftrace 追踪

```bash
# 启用音频事件追踪
adb shell "echo 1 > /sys/kernel/debug/tracing/events/snd/enable"
adb shell "echo 1 > /sys/kernel/debug/tracing/events/asoc/enable"

# 开始追踪
adb shell "echo 1 > /sys/kernel/debug/tracing/tracing_on"

# 执行音频操作...

# 停止追踪
adb shell "echo 0 > /sys/kernel/debug/tracing/tracing_on"

# 查看结果
adb shell cat /sys/kernel/debug/tracing/trace
```

---

## 5. 性能分析

### 5.1 systrace / atrace

```bash
# 采集音频相关 trace
atrace --app audioflinger,audioserver audio freq sched -t 5 -o trace.html

# 或使用 Python 脚本
python $ANDROID/sdk/platform-tools/systrace/systrace.py \
    --app=audioflinger,audioserver \
    audio freq sched \
    -t 5 \
    -o trace.html

# 分析要点:
# 1. AudioFlinger 线程调度
# 2. 缓冲区填充情况
# 3. CPU 频率变化
# 4. 线程唤醒延迟
```

### 5.2 simpleperf

```bash
# 采集性能数据
adb shell simpleperf record -g -p $(adb shell pidof audioserver) --duration 10 -o /data/local/tmp/perf.data

# 拉取数据
adb pull /data/local/tmp/perf.data .

# 分析热点函数
simpleperf report -i perf.data --sort comm,dso,symbol

# 查看调用图
simpleperf report -i perf.data -g caller
```

### 5.3 perfetto

```bash
# 采集 perfetto trace
adb shell perfetto \
    -c - --txt \
    -o /data/local/tmp/trace.perfetto-trace <<EOF
buffers: {
    size_kb: 102400
    fill_policy: RING_BUFFER
}
data_sources: {
    config {
        name: "android.packages_list"
    }
}
data_sources: {
    config {
        name: "linux.process_stats"
    }
}
data_sources: {
    config {
        name: "linux.ftrace"
        ftrace_config {
            ftrace_events: "sched/*"
            ftrace_events: "power/*"
            atrace_apps: "audioflinger,audioserver"
            atrace_categories: "audio"
        }
    }
}
duration_ms: 10000
EOF

# 拉取并打开
adb pull /data/local/tmp/trace.perfetto-trace .
# 使用 https://ui.perfetto.dev/ 打开分析
```

---

## 6. 常见问题诊断

### 6.1 无声问题

```bash
# 诊断步骤:

# 1. 检查音量设置
adb shell dumpsys audio | grep -A 10 "Volumes"
adb shell "settings get system volume_music"

# 2. 检查静音状态
adb shell "settings get system volume_master_mute"
adb shell dumpsys audio | grep -i mute

# 3. 检查音频设备
adb shell cat /proc/asound/cards
adb shell dumpsys audio | grep -A 20 "Connected"

# 4. 检查音频路由
adb shell dumpsys audio | grep -A 30 "Routing"

# 5. 检查 Track 状态
adb shell dumpsys audio | grep -A 20 "Active Track"

# 6. 检查 HAL 状态
adb shell dumpsys audio | grep -A 20 "HAL"

# 7. 检查内核日志
adb shell dmesg | grep -i "audio\|i2s\|pcm\|codec"
```

### 6.2 XRUN 问题

```bash
# 诊断步骤:

# 1. 查看 XRUN 统计
adb shell dumpsys audio | grep -i "underrun\|overrun"

# 2. 查看 PCM 状态
adb shell cat /proc/asound/card0/pcm0p/sub0/status

# 3. 查看缓冲区配置
adb shell cat /proc/asound/card0/pcm0p/sub0/hw_params

# 4. 检查 CPU 占用
adb shell top -n 1 | grep -E "audio|Audio"

# 5. 检查线程优先级
adb shell "ps -T -p $(adb shell pidof audioserver)"

# 6. 查看相关日志
adb logcat | grep -iE "underrun|overrun|xrun|buffer"
```

### 6.3 延迟问题

```bash
# 诊断步骤:

# 1. 查看延迟信息
adb shell dumpsys audio | grep -i latency

# 2. 检查缓冲区大小
adb shell cat /proc/asound/card0/pcm0p/sub0/hw_params | grep buffer_size

# 3. 检查是否使用 Fast Track
adb shell dumpsys audio | grep -i fast

# 4. 检查 mmap 模式
adb shell dumpsys audio | grep -i mmap

# 5. 分析 systrace
atrace --app audioflinger audio freq sched -t 5 -o latency_trace.html
```

### 6.4 设备切换问题

```bash
# 诊断步骤:

# 1. 查看设备连接状态
adb shell dumpsys audio | grep -A 20 "Connected devices"

# 2. 查看路由策略
adb shell dumpsys media.audio_policy | grep -A 30 "Strategy"

# 3. 检查 AudioPatch
adb shell dumpsys audio | grep -A 20 "Patch"

# 4. 查看设备切换日志
adb logcat | grep -iE "device|route|switch|connect"

# 5. 检查 HAL 设备处理
adb logcat -s AudioHAL | grep -i device
```

### 6.5 音频焦点问题

```bash
# 诊断步骤:

# 1. 查看焦点状态
adb shell dumpsys audio | grep -A 20 "Audio Focus"

# 2. 查看焦点持有者
adb shell dumpsys audio | grep -A 10 "Focus owner"

# 3. 检查焦点日志
adb logcat | grep -iE "focus|duck"

# 4. AAOS 焦点状态
adb shell dumpsys car_service | grep -A 20 "AudioFocus"
```

---

## 7. 诊断脚本

### 7.1 综合诊断脚本

```bash
#!/bin/bash
# audio_diagnosis.sh

echo "=== Audio Diagnosis Report ==="
echo "Date: $(date)"
echo ""

echo "=== Audio Devices ==="
adb shell cat /proc/asound/cards
echo ""

echo "=== PCM Devices ==="
adb shell cat /proc/asound/pcm
echo ""

echo "=== Mixer Controls ==="
adb shell tinymix -D 0
echo ""

echo "=== Active Audio Streams ==="
adb shell dumpsys audio | grep -A 20 "Active Track"
echo ""

echo "=== Audio Focus ==="
adb shell dumpsys audio | grep -A 15 "Audio Focus"
echo ""

echo "=== XRUN Statistics ==="
adb shell dumpsys audio | grep -iE "underrun|overrun"
echo ""

echo "=== Volume Settings ==="
adb shell dumpsys audio | grep -A 10 "Volumes"
echo ""

echo "=== Connected Devices ==="
adb shell dumpsys audio | grep -A 20 "Connected"
echo ""

echo "=== Recent Audio Logs ==="
adb logcat -d -s AudioFlinger AudioPolicyService AudioTrack | tail -50
echo ""

echo "=== Diagnosis Complete ==="
```

### 7.2 实时监控脚本

```bash
#!/bin/bash
# audio_monitor.sh

echo "Monitoring audio status (Ctrl+C to stop)..."

while true; do
    clear
    echo "=== Audio Monitor $(date) ==="
    echo ""
    
    echo "PCM Status:"
    adb shell cat /proc/asound/card0/pcm0p/sub0/status | head -10
    echo ""
    
    echo "CPU Usage:"
    adb shell top -n 1 -o %CPU,CMDLINE | grep -i audio | head -5
    echo ""
    
    echo "Recent Logs:"
    adb logcat -d -t 5 -s AudioFlinger AudioPolicyService
    echo ""
    
    sleep 2
done
```

---

## 📌 总结

| 类别 | 工具/命令 |
|------|----------|
| **Framework 状态** | dumpsys audio, dumpsys media.audio_flinger |
| **AAOS 状态** | dumpsys car_service |
| **日志分析** | logcat, dmesg |
| **内核调试** | /proc/asound/, /sys/kernel/debug/asoc/ |
| **事件追踪** | ftrace, systrace, atrace |
| **性能分析** | simpleperf, perfetto |
| **常见问题** | 无声、XRUN、延迟、设备切换、焦点 |
| **诊断脚本** | audio_diagnosis.sh, audio_monitor.sh |
