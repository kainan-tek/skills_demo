---
name: "tinyalsa"
description: "TinyALSA 用户空间音频库，用于 PCM 数据传输、Mixer 控件操作与音频路由配置"
version: "2.0.0"
triggers: ["TinyALSA", "tinymix", "tinyplay", "tinycap", "mixer_ctl", "pcm_open", "pcm_write", "pcm_read", "pcm_mmap", "pcm_config", "pcm_state", "mixer_open", "controlC", "pcmC"]
---

> 参考来源：Android AOSP external/tinyalsa

# 🎛️ TinyALSA 开发规范

---

## 1. TinyALSA 概述

### 1.1 简介

```
TinyALSA 是 Google 在 Android 4.0 之后推出的轻量级 ALSA 用户空间库
- 替代: alsa-lib (GPL 许可证问题)
- 特点: 体积小、接口简洁、满足基本音频需求
- 位置: external/tinyalsa/
```

### 1.2 源码结构

```
external/tinyalsa/
├── include/
│   └── tinyalsa/
│       ├── pcm.h          # PCM 接口定义
│       ├── mixer.h        # Mixer 接口定义
│       └── asoundlib.h    # 兼容旧版本 (已废弃)
├── pcm.c                  # PCM 实现
├── mixer.c                # Mixer 实现
├── tinyplay.c             # 播放工具
├── tinycap.c              # 录音工具
├── tinymix.c              # 混音器控制工具
└── tinypcminfo.c          # PCM 信息查询工具
```

### 1.3 设备节点

```
/dev/snd/
├── controlC0              # 控制设备 (card 0)
├── controlC1              # 控制设备 (card 1)
├── pcmC0D0p               # Playback PCM (card 0, device 0, playback)
├── pcmC0D0c               # Capture PCM (card 0, device 0, capture)
├── pcmC0D1p               # Playback PCM (card 0, device 1, playback)
├── pcmC1D0c               # Capture PCM (card 1, device 0, capture)
└── timer                  # 定时器

命名规则: pcmC<card>D<device><direction>
- C: card (声卡编号)
- D: device (设备编号)
- p: playback (播放)
- c: capture (录音)
```

---

## 2. Mixer 控制

### 2.1 头文件

```cpp
#include <tinyalsa/mixer.h>
```

### 2.2 打开/关闭 Mixer

```cpp
// 打开 Mixer
struct mixer* mixer = mixer_open(card);
if (!mixer) {
    ALOGE("Failed to open mixer for card %d", card);
    return -ENODEV;
}

// 关闭 Mixer
void mixer_close(struct mixer* mixer);
```

### 2.3 Mixer 控件类型

```cpp
enum mixer_ctl_type {
    MIXER_CTL_TYPE_BOOL,        // 布尔类型
    MIXER_CTL_TYPE_INT,         // 整数类型
    MIXER_CTL_TYPE_ENUM,        // 枚举类型
    MIXER_CTL_TYPE_BYTE,        // 字节类型
    MIXER_CTL_TYPE_IEC958,      // IEC958 (S/PDIF) 类型
    MIXER_CTL_TYPE_INT64,       // 64位整数类型
    MIXER_CTL_TYPE_UNKNOWN,     // 未知类型
};
```

### 2.4 获取控件

```cpp
// 按名称获取控件
struct mixer_ctl* ctl = mixer_get_ctl_by_name(mixer, "Master Playback Volume");
if (!ctl) {
    ALOGE("Control not found");
    return -EINVAL;
}

// 按索引获取控件
struct mixer_ctl* ctl = mixer_get_ctl(mixer, index);

// 获取控件数量
unsigned int num_ctls = mixer_get_num_ctls(mixer);

// 遍历所有控件
for (unsigned int i = 0; i < num_ctls; i++) {
    struct mixer_ctl* ctl = mixer_get_ctl(mixer, i);
    const char* name = mixer_ctl_get_name(ctl);
    ALOGD("Control %u: %s", i, name);
}
```

### 2.5 获取控件信息

```cpp
// 获取控件名称
const char* name = mixer_ctl_get_name(ctl);

// 获取控件类型
enum mixer_ctl_type type = mixer_ctl_get_type(ctl);

// 获取类型名称字符串
const char* type_str = mixer_ctl_get_type_string(ctl);

// 获取值的数量 (如立体声有 2 个值)
unsigned int num_values = mixer_ctl_get_num_values(ctl);

// 获取枚举值数量 (仅枚举类型)
unsigned int num_enums = mixer_ctl_get_num_enums(ctl);

// 获取枚举值字符串 (仅枚举类型)
const char* enum_str = mixer_ctl_get_enum_string(ctl, enum_index);

// 获取整数范围 (仅整数类型)
int min = mixer_ctl_get_range_min(ctl);
int max = mixer_ctl_get_range_max(ctl);
```

### 2.6 读取控件值

```cpp
// 读取单个值
int value;
int ret = mixer_ctl_get_value(ctl, index, &value);
if (ret < 0) {
    ALOGE("Failed to get value: %d", ret);
}

// 读取多个值
int values[2];
ret = mixer_ctl_get_array(ctl, values, 2);

// 读取字符串值 (枚举类型)
const char* str_value = mixer_ctl_get_enum_string(ctl, 0);
```

### 2.7 写入控件值

```cpp
// 写入单个值
int ret = mixer_ctl_set_value(ctl, index, value);
if (ret < 0) {
    ALOGE("Failed to set value: %d", ret);
}

// 写入多个值 (如立体声音量)
int values[2] = {80, 80};  // 左右声道
ret = mixer_ctl_set_array(ctl, values, 2);

// 通过字符串设置枚举值
ret = mixer_ctl_set_enum_by_string(ctl, "Speaker");
```

### 2.8 完整示例

```cpp
#include <tinyalsa/mixer.h>

int set_master_volume(int card, int volume) {
    struct mixer* mixer = mixer_open(card);
    if (!mixer) {
        return -ENODEV;
    }
    
    struct mixer_ctl* ctl = mixer_get_ctl_by_name(mixer, "Master Playback Volume");
    if (!ctl) {
        mixer_close(mixer);
        return -EINVAL;
    }
    
    // 检查范围
    int min = mixer_ctl_get_range_min(ctl);
    int max = mixer_ctl_get_range_max(ctl);
    if (volume < min || volume > max) {
        ALOGW("Volume %d out of range [%d, %d]", volume, min, max);
        volume = (volume < min) ? min : max;
    }
    
    // 设置左右声道
    int values[2] = {volume, volume};
    int ret = mixer_ctl_set_array(ctl, values, 2);
    
    mixer_close(mixer);
    return ret;
}

int enable_speaker(int card, bool enable) {
    struct mixer* mixer = mixer_open(card);
    if (!mixer) {
        return -ENODEV;
    }
    
    struct mixer_ctl* ctl = mixer_get_ctl_by_name(mixer, "Speaker Playback Switch");
    if (!ctl) {
        mixer_close(mixer);
        return -EINVAL;
    }
    
    int ret = mixer_ctl_set_value(ctl, 0, enable ? 1 : 0);
    mixer_close(mixer);
    return ret;
}

int set_input_mux(int card, const char* input) {
    struct mixer* mixer = mixer_open(card);
    if (!mixer) {
        return -ENODEV;
    }
    
    struct mixer_ctl* ctl = mixer_get_ctl_by_name(mixer, "Input Mux");
    if (!ctl) {
        mixer_close(mixer);
        return -EINVAL;
    }
    
    int ret = mixer_ctl_set_enum_by_string(ctl, input);
    mixer_close(mixer);
    return ret;
}
```

---

## 3. PCM 操作

### 3.1 头文件

```cpp
#include <tinyalsa/pcm.h>
```

### 3.2 PCM 标志

```cpp
// PCM 打开标志
#define PCM_OUT         0x00000001  // 播放
#define PCM_IN          0x00000002  // 录音
#define PCM_MMAP        0x00000004  // 内存映射模式
#define PCM_NOIRQ       0x00000008  // 无中断模式
#define PCM_MONOTONIC   0x00000010  // 单调时钟
```

### 3.3 PCM 格式

```cpp
enum pcm_format {
    PCM_FORMAT_INVALID = -1,
    PCM_FORMAT_S16_LE = 0,       // 有符号 16 位小端
    PCM_FORMAT_S32_LE,           // 有符号 32 位小端
    PCM_FORMAT_S8,               // 有符号 8 位
    PCM_FORMAT_S24_LE,           // 有符号 24 位小端 (4 字节)
    PCM_FORMAT_S24_3LE,          // 有符号 24 位小端 (3 字节)
    PCM_FORMAT_FLOAT32_LE,       // 浮点 32 位小端
    PCM_FORMAT_A_LAW,            // A-law 压缩
    PCM_FORMAT_MU_LAW,           // μ-law 压缩
};
```

### 3.4 PCM 配置

```cpp
struct pcm_config {
    unsigned int channels;           // 通道数 (1=单声道, 2=立体声)
    unsigned int rate;               // 采样率 (Hz)
    unsigned int period_size;        // 周期大小 (帧数)
    unsigned int period_count;       // 周期数量
    enum pcm_format format;          // 采样格式
    
    // 以下参数控制 PCM 状态机
    unsigned int start_threshold;    // 启动阈值 (帧数)
    unsigned int stop_threshold;     // 停止阈值 (帧数)
    unsigned int silence_threshold;  // 静音阈值 (帧数)
    unsigned int silence_size;       // 静音大小
    unsigned int avail_min;          // 最小可用空间
};

// 配置示例
struct pcm_config config = {
    .channels = 2,
    .rate = 48000,
    .format = PCM_FORMAT_S16_LE,
    .period_size = 1024,         // 每周期 1024 帧
    .period_count = 4,           // 4 个周期
    .start_threshold = 1024,     // 缓冲 1024 帧后启动
    .stop_threshold = 4096,      // 缓冲区满时停止
    .silence_threshold = 0,
    .silence_size = 0,
    .avail_min = 1024,
};
```

### 3.5 打开/关闭 PCM

```cpp
// 打开播放设备
struct pcm* pcm = pcm_open(card, device, PCM_OUT, &config);
if (!pcm || !pcm_is_ready(pcm)) {
    ALOGE("Failed to open PCM: %s", pcm_get_error(pcm));
    return -ENODEV;
}

// 打开录音设备
struct pcm* pcm_in = pcm_open(card, device, PCM_IN, &config);

// 打开 mmap 模式
struct pcm* pcm_mmap = pcm_open(card, device, PCM_OUT | PCM_MMAP, &config);

// 关闭 PCM
int pcm_close(struct pcm* pcm);
```

### 3.6 PCM 状态

```cpp
enum pcm_state {
    PCM_STATE_OPEN = 0,          // 已打开
    PCM_STATE_SETUP,             // 已配置
    PCM_STATE_PREPARED,          // 已准备
    PCM_STATE_RUNNING,           // 运行中
    PCM_STATE_XRUN,              // 缓冲区溢出/欠载
    PCM_STATE_DRAINING,          // 正在排空
    PCM_STATE_PAUSED,            // 已暂停
    PCM_STATE_SUSPENDED,         // 已挂起
    PCM_STATE_DISCONNECTED,      // 已断开
};

// 获取当前状态
enum pcm_state state = pcm_get_state(pcm);
ALOGD("PCM state: %d", state);
```

### 3.7 PCM 控制

```cpp
// 检查 PCM 是否就绪
bool ready = pcm_is_ready(pcm);

// 获取错误信息
const char* error = pcm_get_error(pcm);

// 准备 PCM (SETUP -> PREPARED)
int ret = pcm_prepare(pcm);
if (ret < 0) {
    ALOGE("pcm_prepare failed: %s", pcm_get_error(pcm));
}

// 启动 PCM (PREPARED -> RUNNING)
ret = pcm_start(pcm);

// 停止 PCM (RUNNING -> SETUP)
ret = pcm_stop(pcm);

// 丢弃缓冲区 (立即停止)
ret = pcm_drop(pcm);

// 排空缓冲区 (等待播放完成)
ret = pcm_drain(pcm);

// 暂停/恢复
ret = pcm_pause(pcm, true);   // 暂停
ret = pcm_pause(pcm, false);  // 恢复
```

### 3.8 数据读写

```cpp
// 写入音频数据 (阻塞模式)
int bytes_per_frame = pcm_frames_to_bytes(pcm, 1);
int frames = 1024;
char* buffer = malloc(frames * bytes_per_frame);

// 填充音频数据到 buffer...
int ret = pcm_write(pcm, buffer, frames * bytes_per_frame);
if (ret < 0) {
    ALOGE("pcm_write failed: %s", pcm_get_error(pcm));
}

// 读取音频数据 (阻塞模式)
ret = pcm_read(pcm, buffer, frames * bytes_per_frame);
if (ret < 0) {
    ALOGE("pcm_read failed: %s", pcm_get_error(pcm));
}

free(buffer);
```

### 3.9 PCM 信息获取

```cpp
// 获取缓冲区大小 (帧数)
unsigned int buffer_size = pcm_get_buffer_size(pcm);

// 获取周期大小 (帧数)
unsigned int period_size = pcm_get_period_size(pcm);

// 获取参数
unsigned int channels = pcm_get_channels(pcm);
unsigned int rate = pcm_get_rate(pcm);
enum pcm_format format = pcm_get_format(pcm);

// 帧数与字节数转换
unsigned int bytes = pcm_frames_to_bytes(pcm, frames);
unsigned int frames = pcm_bytes_to_frames(pcm, bytes);

// 获取文件描述符
int fd = pcm_get_file_descriptor(pcm);

// 获取延迟 (微秒)
unsigned int latency = pcm_get_latency(pcm);
```

### 3.10 完整播放示例

```cpp
#include <tinyalsa/pcm.h>

int play_audio(const char* file_path, int card, int device) {
    FILE* fp = fopen(file_path, "rb");
    if (!fp) {
        ALOGE("Failed to open file: %s", file_path);
        return -ENOENT;
    }
    
    // 跳过 WAV 头 (44 字节)
    fseek(fp, 44, SEEK_SET);
    
    // 配置 PCM
    struct pcm_config config = {
        .channels = 2,
        .rate = 48000,
        .format = PCM_FORMAT_S16_LE,
        .period_size = 1024,
        .period_count = 4,
        .start_threshold = 1024,
        .stop_threshold = 0,
        .silence_threshold = 0,
    };
    
    struct pcm* pcm = pcm_open(card, device, PCM_OUT, &config);
    if (!pcm || !pcm_is_ready(pcm)) {
        ALOGE("Failed to open PCM: %s", pcm_get_error(pcm));
        fclose(fp);
        return -ENODEV;
    }
    
    // 分配缓冲区
    size_t buffer_size = pcm_get_buffer_size(pcm) * 
                         pcm_frames_to_bytes(pcm, 1);
    char* buffer = malloc(buffer_size);
    
    // 播放循环
    while (!feof(fp)) {
        size_t bytes_read = fread(buffer, 1, buffer_size, fp);
        if (bytes_read > 0) {
            int ret = pcm_write(pcm, buffer, bytes_read);
            if (ret < 0) {
                ALOGE("pcm_write failed: %s", pcm_get_error(pcm));
                break;
            }
        }
    }
    
    // 排空缓冲区
    pcm_drain(pcm);
    
    free(buffer);
    pcm_close(pcm);
    fclose(fp);
    
    return 0;
}
```

---

## 4. mmap 内存映射

### 4.1 mmap 模式优势

```
- 减少用户态与内核态之间的数据拷贝
- 降低延迟
- 适合实时音频处理
```

### 4.2 打开 mmap PCM

```cpp
struct pcm_config config = {
    .channels = 2,
    .rate = 48000,
    .format = PCM_FORMAT_S16_LE,
    .period_size = 1024,
    .period_count = 4,
};

struct pcm* pcm = pcm_open(card, device, PCM_OUT | PCM_MMAP, &config);
if (!pcm || !pcm_is_ready(pcm)) {
    ALOGE("Failed to open mmap PCM: %s", pcm_get_error(pcm));
    return -ENODEV;
}
```

### 4.3 mmap 写入

```cpp
struct snd_pcm_mmap_status* status;
struct snd_pcm_mmap_control* control;
void* mmap_buffer;

// 获取 mmap 区域
int ret = pcm_mmap_begin(pcm, &mmap_buffer, &offset, &frames);
if (ret < 0) {
    ALOGE("pcm_mmap_begin failed: %s", pcm_get_error(pcm));
    return ret;
}

// 复制音频数据到 mmap 区域
size_t bytes = pcm_frames_to_bytes(pcm, frames);
memcpy((char*)mmap_buffer + offset, audio_data, bytes);

// 提交写入
ret = pcm_mmap_commit(pcm, offset, frames);
if (ret < 0) {
    ALOGE("pcm_mmap_commit failed: %s", pcm_get_error(pcm));
    return ret;
}
```

### 4.4 mmap 读取

```cpp
void* mmap_buffer;
unsigned int offset, frames;

// 获取可读取区域
int ret = pcm_mmap_begin(pcm, &mmap_buffer, &offset, &frames);
if (ret < 0) {
    ALOGE("pcm_mmap_begin failed");
    return ret;
}

// 处理音频数据
size_t bytes = pcm_frames_to_bytes(pcm, frames);
process_audio_data((char*)mmap_buffer + offset, bytes);

// 提交已读取
ret = pcm_mmap_commit(pcm, offset, frames);
```

### 4.5 mmap 同步

```cpp
// 同步硬件指针
int pcm_mmap_sync(struct pcm* pcm, unsigned int frames);

// 获取 mmap 偏移
unsigned int pcm_mmap_offset(struct pcm* pcm);

// 获取可用帧数
int pcm_mmap_avail(struct pcm* pcm);

// 等待数据可用
int pcm_wait(struct pcm* pcm, int timeout_ms);
```

---

## 5. 命令行工具

### 5.1 tinymix (混音器控制)

```bash
# 列出所有控件
tinymix

# 列出指定声卡控件
tinymix -D 0

# 获取控件值
tinymix "Master Playback Volume"

# 设置控件值
tinymix "Master Playback Volume" 80

# 设置立体声值
tinymix "Master Playback Volume" 80 80

# 设置枚举值
tinymix "Input Mux" "Mic1"

# 详细输出
tinymix -a
```

### 5.2 tinyplay (播放)

```bash
# 播放 WAV 文件
tinyplay /sdcard/test.wav

# 指定设备
tinyplay /sdcard/test.wav -D 0 -d 0

# 指定参数
tinyplay /sdcard/test.raw -c 2 -r 48000 -b 16

# 列出 PCM 设备
tinyplay --list-pcms
```

### 5.3 tinycap (录音)

```bash
# 录音到文件
tinycap /sdcard/recording.wav

# 指定参数
tinycap /sdcard/recording.wav -D 0 -d 0 -c 2 -r 48000 -b 16 -p 1024 -n 4

# 限制录音时长 (秒)
tinycap /sdcard/recording.wav -T 10
```

### 5.4 tinypcminfo (PCM 信息)

```bash
# 查看 PCM 设备信息
tinypcminfo -D 0

# 输出示例:
# Info for card 0, device 0:
# PCM out:
#       Format: S16_LE
#       Channels: 2
#       Rate: 48000
#       Period size: 1024
#       Period count: 4
```

---

## 6. 错误处理

### 6.1 常见错误

```cpp
// 检查 PCM 状态
if (pcm_get_state(pcm) == PCM_STATE_XRUN) {
    ALOGW("Buffer XRUN detected");
    
    // 恢复 XRUN
    pcm_prepare(pcm);
    pcm_start(pcm);
}

// 检查返回值
int ret = pcm_write(pcm, buffer, size);
if (ret < 0) {
    const char* error = pcm_get_error(pcm);
    ALOGE("PCM write error: %s", error);
    
    // 根据错误类型处理
    if (pcm_get_state(pcm) == PCM_STATE_XRUN) {
        pcm_prepare(pcm);
    }
}
```

### 6.2 XRUN 处理

```cpp
// XRUN (Underrun/Overrun) 处理
void handle_xrun(struct pcm* pcm) {
    ALOGW("Handling XRUN");
    
    // 1. 停止 PCM
    pcm_stop(pcm);
    
    // 2. 重新准备
    int ret = pcm_prepare(pcm);
    if (ret < 0) {
        ALOGE("pcm_prepare failed: %s", pcm_get_error(pcm));
        return;
    }
    
    // 3. 重新启动
    ret = pcm_start(pcm);
    if (ret < 0) {
        ALOGE("pcm_start failed: %s", pcm_get_error(pcm));
    }
}
```

### 6.3 错误码

```cpp
// 常见错误码
#define PCM_ERROR_NOT_READY     -ENODEV
#define PCM_ERROR_BAD_VALUE     -EINVAL
#define PCM_ERROR_NO_MEMORY     -ENOMEM
#define PCM_ERROR_BUSY          -EBUSY
#define PCM_ERROR_PIPE          -EPIPE    // XRUN
```

---

## 7. 调试

### 7.1 查看 ALSA 设备

```bash
# 列出声卡
cat /proc/asound/cards

# 列出 PCM 设备
cat /proc/asound/pcm

# 查看设备信息
cat /proc/asound/card0/pcm0p/info

# 查看运行状态
cat /proc/asound/card0/pcm0p/sub0/status

# 查看缓冲区状态
cat /proc/asound/card0/pcm0p/sub0/hw_params
cat /proc/asound/card0/pcm0p/sub0/sw_params
```

### 7.2 调试日志

```cpp
// 添加调试信息
ALOGD("PCM config: channels=%d, rate=%d, format=%d",
      config.channels, config.rate, config.format);
ALOGD("Buffer size: %u frames, period size: %u frames",
      pcm_get_buffer_size(pcm), pcm_get_period_size(pcm));
ALOGD("PCM state: %d", pcm_get_state(pcm));
```

### 7.3 常见问题

```bash
# 1. 设备打开失败
# 检查设备节点权限
ls -la /dev/snd/

# 2. 配置不支持
# 查看支持的格式
tinypcminfo -D 0

# 3. XRUN 问题
# 增大缓冲区
config.period_count = 8;

# 4. Mixer 控件找不到
# 列出所有控件
tinymix -a
```

---

## 📌 总结

| 类别 | 关键 API |
|------|----------|
| **Mixer 打开** | mixer_open / mixer_close |
| **Mixer 控件** | mixer_get_ctl_by_name / mixer_get_ctl |
| **控件信息** | mixer_ctl_get_name / mixer_ctl_get_type / mixer_ctl_get_range_min/max |
| **控件读写** | mixer_ctl_get_value / mixer_ctl_set_value / mixer_ctl_set_enum_by_string |
| **PCM 打开** | pcm_open / pcm_close / pcm_is_ready |
| **PCM 配置** | struct pcm_config |
| **PCM 控制** | pcm_prepare / pcm_start / pcm_stop / pcm_drain / pcm_drop |
| **PCM 读写** | pcm_write / pcm_read |
| **mmap** | pcm_mmap_begin / pcm_mmap_commit |
| **状态查询** | pcm_get_state / pcm_get_error / pcm_get_buffer_size |
| **命令行** | tinymix / tinyplay / tinycap / tinypcminfo |
