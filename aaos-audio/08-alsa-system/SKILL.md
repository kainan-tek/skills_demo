---
name: "alsa-system"
description: "Linux ALSA 音频系统开发规范，包含 alsa-lib 用户空间 API 与 ASoC 内核驱动开发"
version: "3.0.0"
triggers: ["ALSA", "ASoC", "DAPM", "snd_soc", "snd_pcm", "alsa-lib", "snd_pcm_open", "snd_pcm_writei", "snd_pcm_readi", "snd_mixer", "codec", "platform", "machine", "dai", "widget", "kcontrol", "/proc/asound", "XRUN", "aplay", "arecord", "amixer"]
---

> 参考来源：Linux Kernel Sound Subsystem Documentation, ALSA Project

# 🔊 Linux ALSA 音频系统开发规范

---

## 1. ALSA 架构概述

### 1.1 分层架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    User Space                                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Applications: aplay, arecord, amixer, PulseAudio, etc.    ││
│  └─────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  alsa-lib (libasound) - 用户空间 API 库                     ││
│  │  - PCM API (snd_pcm_*)                                      ││
│  │  - Control API (snd_ctl_*)                                  ││
│  │  - Mixer API (snd_mixer_*)                                  ││
│  └─────────────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────────┤
│                    Kernel Space                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  ALSA Core (sound/core/)                                   ││
│  │  - PCM, Control, Timer, MIDI 等逻辑设备                    ││
│  └─────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  ASoC Core (sound/soc/)                                    ││
│  │  - Machine, Platform, Codec 驱动框架                       ││
│  │  - DAPM (Dynamic Audio Power Management)                   ││
│  └─────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Hardware Drivers                                           ││
│  │  - Codec Driver (sound/soc/codecs/)                        ││
│  │  - Platform Driver (sound/soc/<vendor>/)                   ││
│  │  - Machine Driver (sound/soc/<vendor>/)                    ││
│  └─────────────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────────┤
│                    Hardware                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │    Codec     │←→│    I2S/PCM   │←→│     DMA      │          │
│  │   (ES8316)   │  │   (CPU DAI)  │  │  (Platform)  │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 源码路径

```
# 用户空间
/usr/include/alsa/           # alsa-lib 头文件
external/alsa-lib/           # Android alsa-lib 源码

# 内核空间
kernel/sound/core/           # ALSA 核心层
kernel/sound/soc/            # ASoC 核心层
kernel/sound/soc/codecs/     # Codec 驱动
kernel/sound/soc/<vendor>/   # 厂商 Platform/Machine 驱动
kernel/include/sound/        # 内核头文件
```

---

## 2. 基本概念

### 2.1 PCM 音频术语

```
┌─────────────────────────────────────────────────────────────────┐
│                     PCM 音频术语                                │
├─────────────────────────────────────────────────────────────────┤
│  Sample (样本)    : 单次采样的数据，位宽决定大小 (8/16/24/32bit)│
│  Channel (声道)   : 单声道(1) / 立体声(2) / 多声道(5.1, 7.1)   │
│  Frame (帧)       : 一个完整的声音单元 = Sample × Channels     │
│  Rate (采样率)    : 每秒采样次数 (8000/44100/48000/96000 Hz)   │
│  Period (周期)    : 两次硬件中断之间的帧数                      │
│  Buffer (缓冲区)  : 整个环形缓冲区大小，通常为 Period × N      │
│                                                                 │
│  计算示例 (48kHz, 16bit, 立体声):                               │
│  - 每帧大小 = 2 × 2 = 4 字节                                    │
│  - 每秒数据量 = 48000 × 4 = 192000 字节                         │
│  - Period = 4800 帧 = 19200 字节 → 每 100ms 产生一次中断        │
│  - Buffer = 4 × Period = 19200 帧 = 76800 字节                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 设备命名

```
# 设备节点
/dev/snd/controlC0           # 控制设备 (card 0)
/dev/snd/pcmC0D0p            # Playback PCM (card 0, device 0)
/dev/snd/pcmC0D0c            # Capture PCM (card 0, device 0)

# 设备名称格式
pcmC<card>D<device><direction>
     │       │      │
     │       │      └── p: playback, c: capture
     │       └── 设备编号
     └── 声卡编号

# alsa-lib 设备名
"default"                    # 默认设备
"hw:0,0"                     # 硬件设备 (card 0, device 0)
"plughw:0,0"                 # 带插件转换的设备
```

---

## 3. alsa-lib 用户空间 API

### 3.1 头文件与编译

```c
#include <alsa/asoundlib.h>  // 主头文件

// 编译链接
// gcc -o app app.c -lasound
```

### 3.2 PCM 打开/关闭

```c
// 打开 PCM 设备
// name: 设备名 ("default", "hw:0,0", "plughw:0,0")
// stream: SND_PCM_STREAM_PLAYBACK 或 SND_PCM_STREAM_CAPTURE
// mode: 0 (阻塞) 或 SND_PCM_NONBLOCK
int snd_pcm_open(snd_pcm_t **pcmp, const char *name,
                 snd_pcm_stream_t stream, int mode);

// 关闭 PCM 设备
int snd_pcm_close(snd_pcm_t *pcm);

// 示例
snd_pcm_t *handle;
int err = snd_pcm_open(&handle, "hw:0,0", SND_PCM_STREAM_PLAYBACK, 0);
if (err < 0) {
    fprintf(stderr, "PCM open error: %s\n", snd_strerror(err));
    return -1;
}
// ...
snd_pcm_close(handle);
```

### 3.3 硬件参数配置

```c
// 分配硬件参数结构体 (栈上分配，自动释放)
snd_pcm_hw_params_t *params;
snd_pcm_hw_params_alloca(&params);

// 初始化硬件参数
snd_pcm_hw_params_any(handle, params);

// 设置访问类型
snd_pcm_hw_params_set_access(handle, params, SND_PCM_ACCESS_RW_INTERLEAVED);

// 设置采样格式
snd_pcm_hw_params_set_format(handle, params, SND_PCM_FORMAT_S16_LE);

// 设置声道数
snd_pcm_hw_params_set_channels(handle, params, 2);

// 设置采样率 (近似)
unsigned int rate = 48000;
snd_pcm_hw_params_set_rate_near(handle, params, &rate, NULL);

// 设置缓冲时间 (微秒)
unsigned int buffer_time = 500000;
snd_pcm_hw_params_set_buffer_time_near(handle, params, &buffer_time, NULL);

// 设置周期时间 (微秒)
unsigned int period_time = buffer_time / 4;
snd_pcm_hw_params_set_period_time_near(handle, params, &period_time, NULL);

// 应用硬件参数
snd_pcm_hw_params(handle, params);

// 获取周期大小
snd_pcm_uframes_t period_frames;
snd_pcm_hw_params_get_period_size(params, &period_frames, NULL);
```

### 3.4 软件参数配置

```c
snd_pcm_sw_params_t *sw_params;
snd_pcm_sw_params_alloca(&sw_params);
snd_pcm_sw_params_current(handle, sw_params);

// 设置启动阈值 (缓冲多少帧后自动启动)
snd_pcm_sw_params_set_start_threshold(handle, sw_params, period_frames);

// 设置停止阈值
snd_pcm_sw_params_set_stop_threshold(handle, sw_params, buffer_frames);

// 设置可用最小帧数
snd_pcm_sw_params_set_avail_min(handle, sw_params, period_frames);

// 应用软件参数
snd_pcm_sw_params(handle, sw_params);
```

### 3.5 数据读写

```c
// 写入数据 (交错模式)
// 返回: 写入的帧数，或负错误码
snd_pcm_sframes_t snd_pcm_writei(snd_pcm_t *pcm, const void *buffer,
                                  snd_pcm_uframes_t size);

// 读取数据 (交错模式)
// 返回: 读取的帧数，或负错误码
snd_pcm_sframes_t snd_pcm_readi(snd_pcm_t *pcm, void *buffer,
                                 snd_pcm_uframes_t size);

// 帧数与字节数转换
ssize_t snd_pcm_frames_to_bytes(snd_pcm_t *pcm, snd_pcm_sframes_t frames);
snd_pcm_sframes_t snd_pcm_bytes_to_frames(snd_pcm_t *pcm, ssize_t bytes);
```

### 3.6 PCM 控制

```c
// 准备 PCM (SETUP -> PREPARED)
int snd_pcm_prepare(snd_pcm_t *pcm);

// 启动 PCM (PREPARED -> RUNNING)
int snd_pcm_start(snd_pcm_t *pcm);

// 停止 PCM (RUNNING -> SETUP)
int snd_pcm_drop(snd_pcm_t *pcm);

// 排空缓冲区 (等待播放完成)
int snd_pcm_drain(snd_pcm_t *pcm);

// 暂停/恢复
int snd_pcm_pause(snd_pcm_t *pcm, int enable);

// 恢复 XRUN
int snd_pcm_recover(snd_pcm_t *pcm, int err, int silent);
```

### 3.7 状态查询

```c
// 获取 PCM 状态
snd_pcm_state_t snd_pcm_state(snd_pcm_t *pcm);

// 状态值:
// SND_PCM_STATE_OPEN        - 已打开
// SND_PCM_STATE_SETUP       - 已配置
// SND_PCM_STATE_PREPARED    - 已准备
// SND_PCM_STATE_RUNNING     - 运行中
// SND_PCM_STATE_XRUN        - XRUN
// SND_PCM_STATE_DRAINING    - 正在排空
// SND_PCM_STATE_PAUSED      - 已暂停
// SND_PCM_STATE_SUSPENDED   - 已挂起
// SND_PCM_STATE_DISCONNECTED - 已断开

// 获取延迟 (帧数)
int snd_pcm_delay(snd_pcm_t *pcm, snd_pcm_sframes_t *delayp);

// 获取可用空间 (帧数)
snd_pcm_sframes_t snd_pcm_avail(snd_pcm_t *pcm);

// 等待可读写
int snd_pcm_wait(snd_pcm_t *pcm, int timeout);
```

### 3.8 完整播放示例

```c
#include <stdio.h>
#include <alsa/asoundlib.h>

int main(int argc, char *argv[]) {
    snd_pcm_t *handle;
    snd_pcm_hw_params_t *params;
    snd_pcm_uframes_t period_frames;
    int err;
    
    // 1. 打开 PCM 设备
    err = snd_pcm_open(&handle, "hw:0,0", SND_PCM_STREAM_PLAYBACK, 0);
    if (err < 0) {
        fprintf(stderr, "PCM open error: %s\n", snd_strerror(err));
        return -1;
    }
    
    // 2. 配置硬件参数
    snd_pcm_hw_params_alloca(&params);
    snd_pcm_hw_params_any(handle, params);
    snd_pcm_hw_params_set_access(handle, params, SND_PCM_ACCESS_RW_INTERLEAVED);
    snd_pcm_hw_params_set_format(handle, params, SND_PCM_FORMAT_S16_LE);
    snd_pcm_hw_params_set_channels(handle, params, 2);
    
    unsigned int rate = 48000;
    snd_pcm_hw_params_set_rate_near(handle, params, &rate, NULL);
    
    unsigned int buffer_time = 500000;
    snd_pcm_hw_params_set_buffer_time_near(handle, params, &buffer_time, NULL);
    
    unsigned int period_time = buffer_time / 4;
    snd_pcm_hw_params_set_period_time_near(handle, params, &period_time, NULL);
    
    snd_pcm_hw_params(handle, params);
    snd_pcm_hw_params_get_period_size(params, &period_frames, NULL);
    
    // 3. 打开音频文件
    FILE *fp = fopen(argv[1], "rb");
    if (!fp) {
        snd_pcm_close(handle);
        return -1;
    }
    
    // 4. 播放循环
    int period_bytes = snd_pcm_frames_to_bytes(handle, period_frames);
    char *buffer = malloc(period_bytes);
    
    while (fread(buffer, 1, period_bytes, fp) > 0) {
        snd_pcm_sframes_t frames = snd_pcm_writei(handle, buffer, period_frames);
        if (frames == -EPIPE) {
            snd_pcm_prepare(handle);
            snd_pcm_writei(handle, buffer, period_frames);
        } else if (frames < 0) {
            fprintf(stderr, "Write error: %s\n", snd_strerror(frames));
            break;
        }
    }
    
    // 5. 清理
    snd_pcm_drain(handle);
    free(buffer);
    fclose(fp);
    snd_pcm_close(handle);
    return 0;
}
```

### 3.9 完整录音示例

```c
#include <stdio.h>
#include <alsa/asoundlib.h>

int main(int argc, char *argv[]) {
    snd_pcm_t *handle;
    snd_pcm_hw_params_t *params;
    snd_pcm_uframes_t period_frames;
    
    // 1. 打开 PCM 设备
    int err = snd_pcm_open(&handle, "hw:0,0", SND_PCM_STREAM_CAPTURE, 0);
    if (err < 0) return -1;
    
    // 2. 配置硬件参数
    snd_pcm_hw_params_alloca(&params);
    snd_pcm_hw_params_any(handle, params);
    snd_pcm_hw_params_set_access(handle, params, SND_PCM_ACCESS_RW_INTERLEAVED);
    snd_pcm_hw_params_set_format(handle, params, SND_PCM_FORMAT_S16_LE);
    snd_pcm_hw_params_set_channels(handle, params, 2);
    
    unsigned int rate = 48000;
    snd_pcm_hw_params_set_rate_near(handle, params, &rate, NULL);
    
    unsigned int buffer_time = 500000;
    snd_pcm_hw_params_set_buffer_time_near(handle, params, &buffer_time, NULL);
    
    unsigned int period_time = buffer_time / 4;
    snd_pcm_hw_params_set_period_time_near(handle, params, &period_time, NULL);
    
    snd_pcm_hw_params(handle, params);
    snd_pcm_hw_params_get_period_size(params, &period_frames, NULL);
    
    // 3. 打开输出文件
    FILE *fp = fopen(argv[1], "wb");
    if (!fp) {
        snd_pcm_close(handle);
        return -1;
    }
    
    // 4. 录音循环
    int period_bytes = snd_pcm_frames_to_bytes(handle, period_frames);
    char *buffer = malloc(period_bytes);
    
    int count = 100;
    while (count-- > 0) {
        snd_pcm_sframes_t frames = snd_pcm_readi(handle, buffer, period_frames);
        if (frames == -EPIPE) {
            snd_pcm_prepare(handle);
            continue;
        } else if (frames < 0) {
            break;
        }
        fwrite(buffer, 1, snd_pcm_frames_to_bytes(handle, frames), fp);
    }
    
    // 5. 清理
    free(buffer);
    fclose(fp);
    snd_pcm_close(handle);
    return 0;
}
```

### 3.10 XRUN 处理

```c
// XRUN 类型:
// - Underrun: 播放时数据供应不足 (snd_pcm_writei 返回 -EPIPE)
// - Overrun:  录音时数据处理不及时 (snd_pcm_readi 返回 -EPIPE)

// 处理方法 1: 手动恢复
if (frames == -EPIPE) {
    fprintf(stderr, "XRUN occurred\n");
    snd_pcm_prepare(handle);
}

// 处理方法 2: 使用 snd_pcm_recover
if (frames < 0) {
    if (snd_pcm_recover(handle, frames, 0) < 0) {
        return -1;
    }
}

// 预防 XRUN:
// 1. 增大缓冲区 (buffer_time)
// 2. 减小周期时间 (period_time)
// 3. 提高数据处理优先级
// 4. 使用 poll/select 等待数据可用
```

### 3.11 Control/Mixer 接口

```c
#include <alsa/control.h>
#include <alsa/mixer.h>

// 打开混音器
snd_mixer_t *mixer;
snd_mixer_open(&mixer, 0);
snd_mixer_attach(mixer, "hw:0");
snd_mixer_selem_register(mixer, NULL, NULL);
snd_mixer_load(mixer);

// 获取混音器元素
snd_mixer_selem_id_t *sid;
snd_mixer_selem_id_alloca(&sid);
snd_mixer_selem_id_set_name(sid, "Master");
snd_mixer_selem_id_set_index(sid, 0);

snd_mixer_elem_t *elem = snd_mixer_find_selem(mixer, sid);

// 获取/设置音量
long min, max, value;
snd_mixer_selem_get_playback_volume_range(elem, &min, &max);
snd_mixer_selem_get_playback_volume(elem, SND_MIXER_SCHN_FRONT_LEFT, &value);
snd_mixer_selem_set_playback_volume(elem, SND_MIXER_SCHN_FRONT_LEFT, value);

// 获取/设置静音
int muted;
snd_mixer_selem_get_playback_switch(elem, SND_MIXER_SCHN_FRONT_LEFT, &muted);
snd_mixer_selem_set_playback_switch(elem, SND_MIXER_SCHN_FRONT_LEFT, !muted);

// 关闭混音器
snd_mixer_close(mixer);
```

---

## 4. ASoC 内核驱动

### 4.1 ASoC 驱动三要素

```
┌─────────────────────────────────────────────────────────────────┐
│                        Machine Driver                            │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  - 描述硬件连接关系 (CPU DAI <-> Codec DAI)                 ││
│  │  - 定义 snd_soc_dai_link                                    ││
│  │  - 注册 snd_soc_card                                        ││
│  └─────────────────────────────────────────────────────────────┘│
│           ↙                    ↘                                │
│  ┌────────────────┐      ┌────────────────┐                     │
│  │ Platform Driver│      │  Codec Driver  │                     │
│  ├────────────────┤      ├────────────────┤                     │
│  │ - CPU DAI      │      │ - Codec DAI    │                     │
│  │ - DMA Engine   │      │ - DAPM Widgets │                     │
│  │ - PCM DMA Ops  │      │ - Kcontrols    │                     │
│  │ - I2S/PCM 控制器│      │ - Regmap 配置  │                     │
│  └────────────────┘      └────────────────┘                     │
│           │                    │                                │
│           └──────────┬─────────┘                                │
│                      ↓                                          │
│              ┌──────────────┐                                   │
│              │   I2S/PCM    │                                   │
│              │    总线      │                                   │
│              └──────────────┘                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Machine Driver

```c
// DAI Link 定义
SND_SOC_DAILINK_DEFS(pcm,
    DAILINK_COMP_ARRAY(COMP_CPU("fe-dai")),
    DAILINK_COMP_ARRAY(COMP_CODEC("es8316.0-0010", "ES8316 HiFi")),
    DAILINK_COMP_ARRAY(COMP_PLATFORM("dma-engine")));

static struct snd_soc_dai_link rk_dai_link = {
    .name = "ES8316",
    .stream_name = "ES8316 PCM",
    .dai_fmt = SND_SOC_DAIFMT_I2S |
               SND_SOC_DAIFMT_NB_NF |
               SND_SOC_DAIFMT_CBS_CFS,
    .ops = &rk_ops,
    SND_SOC_DAILINK_REG(pcm),
};

static struct snd_soc_card rk_card = {
    .name = "rockchip-es8316",
    .owner = THIS_MODULE,
    .dai_link = &rk_dai_link,
    .num_links = 1,
    .dapm_widgets = rk_dapm_widgets,
    .num_dapm_widgets = ARRAY_SIZE(rk_dapm_widgets),
    .dapm_routes = rk_audio_map,
    .num_dapm_routes = ARRAY_SIZE(rk_audio_map),
};

static int rk_probe(struct platform_device *pdev) {
    struct snd_soc_card *card = &rk_card;
    card->dev = &pdev->dev;
    return devm_snd_soc_register_card(&pdev->dev, card);
}
```

### 4.3 Platform Driver (CPU DAI)

```c
static const struct snd_soc_dai_ops rk_i2s_dai_ops = {
    .hw_params = rk_i2s_hw_params,
    .set_sysclk = rk_i2s_set_sysclk,
    .set_fmt = rk_i2s_set_fmt,
    .trigger = rk_i2s_trigger,
};

static struct snd_soc_dai_driver rk_i2s_dai = {
    .name = "rockchip-i2s",
    .playback = {
        .stream_name = "Playback",
        .channels_min = 2,
        .channels_max = 8,
        .rates = SNDRV_PCM_RATE_8000_192000,
        .formats = SNDRV_PCM_FMTBIT_S16_LE |
                   SNDRV_PCM_FMTBIT_S24_LE |
                   SNDRV_PCM_FMTBIT_S32_LE,
    },
    .capture = {
        .stream_name = "Capture",
        .channels_min = 2,
        .channels_max = 8,
        .rates = SNDRV_PCM_RATE_8000_192000,
        .formats = SNDRV_PCM_FMTBIT_S16_LE |
                   SNDRV_PCM_FMTBIT_S24_LE |
                   SNDRV_PCM_FMTBIT_S32_LE,
    },
    .ops = &rk_i2s_dai_ops,
};

static const struct snd_soc_component_driver rk_i2s_component = {
    .name = "rockchip-i2s",
};

static int rk_i2s_probe(struct platform_device *pdev) {
    return devm_snd_soc_register_component(&pdev->dev,
            &rk_i2s_component, &rk_i2s_dai, 1);
}
```

### 4.4 Codec Driver

```c
static const struct snd_soc_dapm_widget es8316_dapm_widgets[] = {
    SND_SOC_DAPM_INPUT("MIC1"),
    SND_SOC_DAPM_INPUT("MIC2"),
    SND_SOC_DAPM_DAC("DAC", "Playback", ES8316_DAC_ENABLE, 0, 0),
    SND_SOC_DAPM_ADC("ADC", "Capture", ES8316_ADC_ENABLE, 0, 0),
    SND_SOC_DAPM_OUTPUT("HPOL"),
    SND_SOC_DAPM_OUTPUT("HPOR"),
    SND_SOC_DAPM_SUPPLY("Bias", ES8316_SYS_LP1, 5, 0, NULL, 0),
};

static const struct snd_soc_dapm_route es8316_dapm_routes[] = {
    {"DAC", NULL, "Bias"},
    {"ADC", NULL, "Bias"},
    {"HPOL", NULL, "DAC"},
    {"HPOR", NULL, "DAC"},
    {"ADC", NULL, "MIC1"},
};

static const struct snd_kcontrol_new es8316_controls[] = {
    SOC_SINGLE("Master Playback Volume", ES8316_DAC_VOLL, 0, 0xff, 0),
    SOC_SINGLE("Master Playback Switch", ES8316_DAC_MUTE, 0, 1, 0),
};

static const struct snd_soc_component_driver es8316_component = {
    .name = "ES8316",
    .controls = es8316_controls,
    .num_controls = ARRAY_SIZE(es8316_controls),
    .dapm_widgets = es8316_dapm_widgets,
    .num_dapm_widgets = ARRAY_SIZE(es8316_dapm_widgets),
    .dapm_routes = es8316_dapm_routes,
    .num_dapm_routes = ARRAY_SIZE(es8316_dapm_routes),
};

static struct snd_soc_dai_driver es8316_dai = {
    .name = "ES8316 HiFi",
    .playback = {
        .stream_name = "Playback",
        .channels_min = 1,
        .channels_max = 2,
        .rates = SNDRV_PCM_RATE_8000_48000,
        .formats = SNDRV_PCM_FMTBIT_S16_LE | SNDRV_PCM_FMTBIT_S24_LE,
    },
    .capture = {
        .stream_name = "Capture",
        .channels_min = 1,
        .channels_max = 2,
        .rates = SNDRV_PCM_RATE_8000_48000,
        .formats = SNDRV_PCM_FMTBIT_S16_LE | SNDRV_PCM_FMTBIT_S24_LE,
    },
    .ops = &es8316_dai_ops,
};
```

---

## 5. DAPM 动态电源管理

### 5.1 Widget 类型

```c
enum snd_soc_dapm_type {
    snd_soc_dapm_input,         // 输入端点
    snd_soc_dapm_output,        // 输出端点
    snd_soc_dapm_mux,           // 多路选择器
    snd_soc_dapm_mixer,         // 混音器
    snd_soc_dapm_pga,           // 可编程增益放大器
    snd_soc_dapm_adc,           // ADC
    snd_soc_dapm_dac,           // DAC
    snd_soc_dapm_mic,           // 麦克风
    snd_soc_dapm_hp,            // 耳机
    snd_soc_dapm_spk,           // 扬声器
    snd_soc_dapm_switch,        // 开关
    snd_soc_dapm_supply,        // 电源供应
    snd_soc_dapm_dai_in,        // DAI 输入
    snd_soc_dapm_dai_out,       // DAI 输出
};
```

### 5.2 Widget 定义宏

```c
// 输入/输出端点
SND_SOC_DAPM_INPUT(name)
SND_SOC_DAPM_OUTPUT(name)

// ADC/DAC
SND_SOC_DAPM_ADC(name, stream_name, reg, shift, on)
SND_SOC_DAPM_DAC(name, stream_name, reg, shift, on)

// 麦克风/扬声器/耳机
SND_SOC_DAPM_MIC(name, event)
SND_SOC_DAPM_SPK(name, event)
SND_SOC_DAPM_HP(name, event)

// 混音器/开关
SND_SOC_DAPM_MIXER(name, reg, shift, invert, controls, num_controls)
SND_SOC_DAPM_SWITCH(name, reg, shift, invert, controls)

// 电源
SND_SOC_DAPM_SUPPLY(name, reg, shift, invert, event, flags)
```

### 5.3 Route 定义

```c
// 音频路径路由
// 格式: {Sink, Control, Source}

static const struct snd_soc_dapm_route audio_map[] = {
    // DAC -> 输出
    {"Output Mixer", "DAC Switch", "DAC"},
    {"HP", NULL, "Output Mixer"},
    
    // 输入 -> ADC
    {"ADC", NULL, "Input Mux"},
    {"Input Mux", "MIC1", "MIC1"},
    
    // 电源依赖
    {"DAC", NULL, "DAC Power"},
    {"ADC", NULL, "ADC Power"},
};
```

### 5.4 DAPM 状态机

```
┌─────────────────────────────────────────────────────────────────┐
│                     DAPM Power State                            │
├─────────────────────────────────────────────────────────────────┤
│   ┌──────────┐    path complete    ┌──────────┐                │
│   │  OFF     │ ─────────────────→  │  ON      │                │
│   └──────────┘                      └──────────┘                │
│        ↑                                 │                      │
│        │         path incomplete         │                      │
│        └─────────────────────────────────┘                      │
│                                                                 │
│   Widget 状态变化条件:                                          │
│   - 有完整的音频路径连接到端点 → ON                             │
│   - 音频路径断开 → OFF                                          │
│   - 依赖的 Supply 未开启 → OFF                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Kcontrol 控制接口

### 6.1 Kcontrol 类型

```c
// 简单整数控制
SOC_SINGLE(name, reg, shift, max, invert)

// 立体声整数控制
SOC_DOUBLE(name, reg, shift_left, shift_right, max, invert)

// 立体声 TLV 控制 (带音量信息)
SOC_DOUBLE_TLV(name, reg, shift_left, shift_right, max, invert, tlv)

// 枚举控制
SOC_ENUM(name, enum)

// DAPM 枚举
SOC_DAPM_ENUM(name, enum)
```

### 6.2 Kcontrol 示例

```c
// 音量控制 (带 TLV)
static const DECLARE_TLV_DB_SCALE(playback_tlv, -9600, 50, 0);

static const struct snd_kcontrol_new es8316_controls[] = {
    // 单声道音量
    SOC_SINGLE_TLV("Master Playback Volume",
                   ES8316_DAC_VOLL, 0, 0xff, 0, playback_tlv),
    
    // 立体声音量
    SOC_DOUBLE_TLV("Headphone Volume",
                   ES8316_HPVOL, 0, 4, 0x0f, 0, hp_tlv),
    
    // 开关控制
    SOC_SINGLE("Master Playback Switch",
               ES8316_DAC_MUTE, 0, 1, 0),
    
    // 枚举控制
    SOC_ENUM("Input Mux", &input_mux_enum),
};

// 枚举定义
static const char * const input_mux_text[] = {
    "MIC1", "MIC2", "DMIC"
};

static const struct soc_enum input_mux_enum =
    SOC_ENUM_SINGLE(ES8316_ADC_MUX, 0, 3, input_mux_text);
```

---

## 7. 调试方法

### 7.1 /proc/asound 调试

```bash
# 列出所有声卡
cat /proc/asound/cards

# 列出所有 PCM 设备
cat /proc/asound/pcm

# 查看 PCM 状态
cat /proc/asound/card0/pcm0p/sub0/status

# 查看硬件参数
cat /proc/asound/card0/pcm0p/sub0/hw_params

# 查看 DAPM 状态
cat /sys/kernel/debug/asoc/card0/dapm
```

### 7.2 命令行工具

```bash
# 列出 PCM 设备
aplay -l
arecord -l

# 播放测试
aplay -D hw:0,0 -r 48000 -c 2 -f S16_LE test.wav

# 录音测试
arecord -D hw:0,0 -r 48000 -c 2 -f S16_LE -d 5 test.wav

# 查看所有控件
amixer

# 设置音量
amixer sset 'Master Playback Volume' 50%

# 设置开关
amixer sset 'Master Playback Switch' on

# 设置枚举
amixer sset 'Input Mux' 'MIC1'

# 查看控件列表
amixer controls
amixer contents
```

### 7.3 内核日志

```bash
# 查看 ALSA 日志
dmesg | grep -i alsa

# 查看 ASoC 日志
dmesg | grep -i asoc

# 查看特定声卡日志
dmesg | grep -i "es8316"

# 启用 ALSA 调试
echo 1 > /sys/module/snd/parameters/debug
```

---

## 8. 常见问题诊断

### 8.1 无声问题

```bash
# 1. 检查声卡是否识别
cat /proc/asound/cards

# 2. 检查 PCM 状态
cat /proc/asound/card0/pcm0p/sub0/status

# 3. 检查 DAPM 路径
cat /sys/kernel/debug/asoc/card0/dapm

# 4. 检查音量设置
amixer sget 'Master Playback Volume'
amixer sget 'Master Playback Switch'

# 5. 检查内核日志
dmesg | grep -i "es8316\|i2s\|dma"
```

### 8.2 XRUN 问题

```bash
# 查看 XRUN 统计
cat /proc/asound/card0/pcm0p/sub0/status

# XRUN 类型:
# - underrun: 播放时数据供应不足
# - overrun: 录音时数据处理不及时

# 解决方案:
# 1. 增大缓冲区
# 2. 提高数据供应优先级
# 3. 检查 DMA 配置
```

### 8.3 采样率问题

```bash
# 查看支持的采样率
cat /proc/asound/card0/pcm0p/info

# 查看当前硬件参数
cat /proc/asound/card0/pcm0p/sub0/hw_params
```

---

## 📌 总结

| 类别 | 关键 API/结构 |
|------|--------------|
| **alsa-lib PCM** | snd_pcm_open, snd_pcm_hw_params, snd_pcm_writei/readi |
| **alsa-lib Control** | snd_ctl_open, snd_mixer_open, snd_mixer_selem_* |
| **Machine Driver** | snd_soc_card, snd_soc_dai_link |
| **Platform Driver** | snd_soc_dai_driver, snd_soc_component_driver |
| **Codec Driver** | snd_soc_component_driver, regmap |
| **DAPM Widget** | SND_SOC_DAPM_* 宏 |
| **DAPM Route** | snd_soc_dapm_route |
| **Kcontrol** | SOC_SINGLE, SOC_DOUBLE, SOC_ENUM |
| **调试路径** | /proc/asound/, /sys/kernel/debug/asoc/ |
| **调试命令** | aplay, arecord, amixer, dmesg |
