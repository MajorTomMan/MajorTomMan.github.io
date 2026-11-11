---
title: Android HDMI IN 与本地媒体音量差异：一次 Audio Policy 与 AudioFlinger 排查
date: 2026-11-11 15:30:00
categories:
  - 技术文档
tags:
  - Android
  - AudioFlinger
  - Audio Policy
  - HDMI IN
  - 音频调试
description: 记录 Android 12 定制设备上 HDMI IN 与本地媒体响度不同的调查方法、命令、对照数据和阶段性结论。
---

某 Android 12 定制设备使用相同的内置扬声器，却出现了这样的问题：**HDMI IN 外部输入的声音基本正常，而 Android 本地媒体在相同音量档位下明显偏大**。

最初尝试通过修改 Speaker 的媒体音量曲线降低本地播放音量。测试中发现，HDMI IN 预览应用也使用了 `STREAM_MUSIC`，因此公共音量曲线可能同时影响两种模式。为了避免“把两个音源一起调小，却没有解决差异”的问题，随后从运行时音频路由入手进行对照。

> 本文已脱敏：省略设备型号、厂商项目路径、应用包名和内部业务信息。测试值仅代表本次样本，不是 Android 平台的统一参数。

## 一、明确要调查的三个问题

1. HDMI IN 的声音是否直接走硬件通路，还是经过 Android 音频框架？
2. HDMI IN 与本地媒体是否使用同一套系统音量衰减？
3. 如果音量等级与衰减相同，响度差异可能出现在哪个环节？

排查遵循 **先确认通路、再对照增益、最后定位代码** 的顺序。

```text
HDMI IN / 本地音源
        ↓
应用层采集或媒体解码
        ↓
AudioRecord / AudioTrack / MediaPlayer
        ↓
Audio Policy（路由、音量曲线）
        ↓
AudioFlinger（混音、轨道增益）
        ↓
Audio HAL → 功放 → Speaker
```

这里的示意图是逻辑分层，具体设备可能有硬件直通、HAL 内部重定向等实现，不能仅凭设备列表推断实际路径。

## 二、确认运行时采用的音频策略

先查看设备加载的音频策略及音量配置：

```bash
adb shell dumpsys media.audio_policy | grep -i 'Config source'
adb shell ls /vendor/etc/*audio*policy*.xml
adb shell cat /vendor/etc/audio_policy_configuration.xml
adb shell cat /vendor/etc/audio_policy_volumes.xml
adb shell cat /vendor/etc/default_volume_tables.xml
```

重点关注：

- `AUDIO_DEVICE_IN_HDMI`：HDMI 音频输入设备；
- `AUDIO_DEVICE_OUT_SPEAKER`：内置扬声器；
- `AUDIO_STREAM_MUSIC`：媒体音量流；
- `DEVICE_CATEGORY_SPEAKER`：扬声器音量曲线类别。

**注意：设备已注册 HDMI 输入节点，不代表当前音频一定通过该节点采集。** 需要继续分析活动的 Audio Patch、录音会话和播放轨道。

## 三、抓取 HDMI IN 的运行时链路

让外部设备持续播放有声音的内容，保持 HDMI IN 画面处于工作状态：

```bash
mkdir -p audio_debug/hdmi

adb shell dumpsys audio > audio_debug/hdmi/audio.txt
adb shell dumpsys media.audio_policy > audio_debug/hdmi/policy.txt
adb shell dumpsys media.audio_flinger > audio_debug/hdmi/flinger.txt
```

查看 Audio Policy 中的 Patch 和设备节点：

```bash
grep -inE -A 15 -B 5 \
  'Audio Patches|AUDIO_DEVICE_IN_HDMI|AUDIO_DEVICE_OUT_SPEAKER' \
  audio_debug/hdmi/policy.txt
```

再检查 AudioService 中真正活动的播放器与采集会话：

```bash
grep -inE 'state:started|active\? true|AudioTrack|AudioRecord|pack:' \
  audio_debug/hdmi/audio.txt
```

**本次观察：**

- 系统注册了 HDMI 输入节点，但活动 Patch 中未看到直接的 `HDMI IN → Speaker` 连接。
- 一个预览应用同时存在活跃的录音会话与 `AudioTrack` 播放轨道。
- 播放属性为 `USAGE_MEDIA / CONTENT_TYPE_MUSIC`，最终路由到 Speaker。
- Framework 将录音端显示为 `CAMCORDER / BACK_MIC`。

最后一点值得注意：**这只是 Framework 所报告的逻辑输入设备**，不能据此认定 HDMI 音频来自真实麦克风，也不能排除 HAL 做了底层重定向。

## 四、确认媒体音量曲线是否生效

Android 的音量策略可以按流类型与设备类别指定曲线。例如：

```xml
<volume stream="AUDIO_STREAM_MUSIC"
        deviceCategory="DEVICE_CATEGORY_SPEAKER">
    <point>1,-5400</point>
    <point>20,-4000</point>
    <point>40,-3500</point>
    <point>60,-3000</point>
    <point>80,-2500</point>
    <point>100,-2000</point>
</volume>
```

这里的点值单位是 **mB（毫贝）**，`100 mB = 1 dB`，因此 `-4000` 对应 `-40 dB`。上述数值是本次用于验证的定制曲线，不应不经测试直接应用到其他设备。

在当前设备上设置媒体音量 20：

```bash
adb shell cmd media_session volume --stream 3 --set 20
adb shell dumpsys media.audio_flinger > flinger.txt
grep -n -A 4 'Tracks of which' flinger.txt
grep -n 'Stream volumes in dB' flinger.txt
```

检查对应活动轨道的 `ST`（Stream Type）、`G db`（轨道总增益）、左右声道增益以及输出设备。

此次 HDMI IN 播放轨道在音量 20 时显示约 **-40 dB**，与配置曲线一致。因此，这种 HDMI 预览实现**同样会受到 Speaker/MUSIC 曲线影响**，并非完全独立的音量通道。

## 五、获取真正的本地播放对照数据

第一次对照失败的原因很简单：虽然返回了桌面，HDMI 预览应用仍在后台维持音频采集和播放，抓到的始终是同一个会话。

因此，采样前先确认 HDMI 预览音频会话已经结束。必要时可终止测试应用（注意会中断其当前功能）：

```bash
adb shell am force-stop <HDMI_PREVIEW_PACKAGE>
```

随后启动本地播放器，让测试音频**持续出声**，保持同样的音量档位：

```bash
mkdir -p audio_debug/local

adb shell cmd media_session volume --stream 3 --set 20
adb shell dumpsys audio > audio_debug/local/audio.txt
adb shell dumpsys media.audio_policy > audio_debug/local/policy.txt
adb shell dumpsys media.audio_flinger > audio_debug/local/flinger.txt
```

核对播放应用、PID、Session、活动轨道及录音状态：

```bash
grep -inE 'state:started|active\? true|rec release' audio_debug/local/audio.txt
grep -n -A 4 'Tracks of which' audio_debug/local/flinger.txt
```

**不能只看到文件成功生成就算完成对照**：应确保本地播放器产生了新的活跃播放会话，且旧的 HDMI 录音和播放会话已结束。

## 六、两种模式的对照结果

| 指标 | HDMI IN 模式 | Android 本地媒体 |
| --- | --- | --- |
| 播放接口 | AudioTrack | MediaPlayer |
| 音频流 | MUSIC | MUSIC |
| 系统音量等级 | 20 | 20 |
| 策略衰减 | -40 dB | -40 dB |
| 输出设备 | Speaker | Speaker |
| AudioFlinger 输出线程 | Primary Mixer | Deep Buffer Mixer |
| 活动效果链 | 未发现 | 未发现 |
| 轨道左右声道附加增益 | 0 dB / 0 dB | 0 dB / 0 dB |

从 `AudioFlinger` 的输出线程 `Signal power history` 进行初步抽样，HDMI IN 与本地媒体的信号电平中位数分别约为 **-71 dB**、**-58 dB**，相差约 **13 dB**。

但这组结果必须附带两个限制：

1. 两次播放的**不是同一段音源**，不能据此断定系统链路具有固定 13 dB 的增益差。
2. `Signal power history` 反映的是输出链路中的数字信号电平，**不是扬声器的实际声压级（SPL）**，也不能独立代表主观响度。

因此，以上数据说明“本次采样中本地媒体输出电平较高”，**尚不能证明高出的部分由 Audio HAL 或应用缺陷造成**。

## 七、如何缩小根因范围

当两种模式使用相同的 MUSIC 曲线且具有相同的 -40 dB 策略衰减时，继续降低全局曲线无法解释或消除原有的相对差异。剩余排查重点如下：

| 排查对象 | 可能影响 | 验证方法 |
| --- | --- | --- |
| 音源自身 | 文件电平、母带响度不同 | 两种模式播放同一测试音频 |
| HDMI 采集侧 | 采集增益、PCM 缩放、格式转换 | 检查录音信号电平与 HAL 输入逻辑 |
| HDMI 预览应用 | AudioTrack 增益或软件处理 | 检索 `setVolume`、`setStereoVolume`、PCM `read/write` |
| HAL 输出路径 | Primary / Deep Buffer 处理可能不同 | 对照不同输出流的 HAL 参数和实现 |

如需定位应用安装位置：

```bash
adb shell pm list packages | grep -iE 'hdmi|preview|camera'
adb shell pm path <PACKAGE_NAME>
adb shell ps -A | grep <PROCESS_NAME>
```

在有源码的情况下，优先阅读音频采集和回放代码；没有源码时，再考虑反编译 APK，必要时继续排查 JNI 与相关 native 库。**是否存在额外软件增益，目前尚无直接证据。**

## 八、继续验证的推荐实验

为消除音源差异，应做一次控制变量测试：

1. 准备一段固定幅度的测试 WAV，例如 **1 kHz、-20 dBFS** 的正弦波；
2. HDMI 输入端与 Android 本地播放器使用**相同文件**；
3. 固定系统音量档位、扬声器、音效状态，并确保没有其他音频同时播放；
4. 分别抓取 AudioService、Audio Policy、AudioFlinger 日志；
5. 比较对应时间段的信号电平、轨道增益和输出线程；
6. 如仍存在稳定差异，再追查 HDMI 采集及应用层 PCM 处理，而不是继续猜测音量曲线。

如需在 Android 源码树中确认最终打包的音量 XML 来源：

```bash
source build/envsetup.sh
lunch
get_build_var PRODUCT_COPY_FILES | tr ' ' '\n' | grep 'audio_policy_volumes.xml'
```

这个方法比仅查找同名 XML 更可靠：某些平台通过 `wildcard` 和变量批量复制配置文件，无法通过文件名全文搜索直接找到明确的复制规则。

## 九、结论

这次排查得到四个可以复用的经验：

1. **HDMI IN 不等于一定绕过 Android 音频框架。** 本次设备的预览应用通过媒体音频轨道播放声音，受到 MUSIC/Speaker 曲线控制。
2. **相同音量档位、相同衰减，不等于相同输出电平。** 音源和上游处理仍会影响最终结果。
3. **先验证活动会话，再比较日志。** 否则即使切换界面，也可能一直在比较同一条后台播放通路。
4. **全局音量曲线优化与多音源响度归一化是两个问题。** 前者解决音量整体过高或档位不合理，后者需要进一步识别来源专属增益。

**阶段性结论：** 已确认两种模式都应用了媒体音量策略，并使用不同的 AudioFlinger 输出线程；初次采样显示本地媒体信号电平更高。但尚未完成相同音源条件下的对照，不能认定固定的增益差来自 HAL 或预览应用。

后续应以统一音源实验为分界点，确定是否需要调整采集、应用或 HAL，而不是贸然进一步压低公共 Speaker 音量。
