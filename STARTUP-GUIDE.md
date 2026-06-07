# Selkies Termux 启动指南

## 前置服务

```bash
# 1. 启动桌面 (Xvnc + XFCE4)
vncserver :1 -geometry 1920x1080 -localhost -depth 24
DISPLAY=:1 startxfce4 &

# 2. 启动音频 (PulseAudio)
pulseaudio --start --daemonize
pactl load-module module-native-protocol-tcp auth-ip-acl=127.0.0.1
```

## 启动 Selkies

```bash
# 直接用启动脚本
nohup bash ~/proton11/launch-selkies-new.sh </dev/null >/dev/null 2>&1 & disown
```

启动后访问: `http://<设备IP>:8081`

## 端口

| 服务 | 端口 |
|------|------|
| Selkies 主界面 | 8081 |
| Xvnc (VNC) | 5901 |
| PulseAudio TCP | 4713 |

## 环境变量说明

| 变量 | 值 | 说明 |
|------|------|------|
| `SELKIES_PORT` | 8081 | Web 服务端口 |
| `SELKIES_MODE` | webrtc | 使用 WebRTC 模式 |
| `SELKIES_MEDIA_PIPELINE` | gstreamer | GStreamer 管线 (不用 pixelflux) |
| `SELKIES_ENCODER_RTC` | h264_mediacodec | 硬件编码 (MediaCodec) |
| `SELKIES_FRAMERATE` | 30 | 帧率 |
| `SELKIES_WEB_ROOT` | ~/proton11/selkies/addons/gst-web/src | 前端文件路径 |
| `SELKIES_AUDIO_ENABLED` | true | 启用音频 |
| `SELKIES_ENABLE_BASIC_AUTH` | false | 关闭认证 |
| `DISPLAY` | :1 | X11 显示编号 |
| `PULSE_SERVER` | tcp:127.0.0.1:4713 | PulseAudio 服务器地址 |
| `SELKIES_IGNORE_HOSTS` | 1 | 忽略 host ICE candidate (Termux 兼容) |

## 重启流程

```bash
# 1. 找到进程
ps aux | grep selkies

# 2. 杀掉旧进程
kill <PID>

# 3. 重新启动
nohup bash ~/proton11/launch-selkies-new.sh </dev/null >/dev/null 2>&1 & disown
```

一键重启:
```bash
kill $(pgrep -f 'selkies$') 2>/dev/null; sleep 2; nohup bash ~/proton11/launch-selkies-new.sh </dev/null >/dev/null 2>&1 & disown
```

## 检查运行状态

```bash
# 检查进程
ps aux | grep selkies

# 测试端口
curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8081/
# 返回 200 表示正常
```

## 常见问题

### pixelflux 加载失败 (libandroid_shmget)

如果使用 `SELKIES_MEDIA_PIPELINE=pixelflux`，需要 preload Termux 的共享内存库:

```bash
export LD_PRELOAD=/data/data/com.termux/files/usr/lib/libandroid-shmem.so
```

推荐直接使用 `SELKIES_MEDIA_PIPELINE=gstreamer` 避免此问题(如 `launch-selkies-new.sh` 所示)。

### 前端的系统监控 (CPU/内存) 显示 0

数据通过 WebRTC DataChannel 传输。如果一直显示 0:

1. 检查前台覆盖层 RX# 计数是否增长 → 确认数据是否到达前端
2. 检查后端 SystemMonitor 是否启动 → `logcat | grep system_monitor`
3. Android `/proc/meminfo` 可能缺少 `MemAvailable:` 行，代码已使用 `MemFree+Buffers+Cached` 做 fallback
