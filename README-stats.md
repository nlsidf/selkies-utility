# Selkies 系统监控（CPU/内存）实现说明

## 概述

本项目在 Selkies WebRTC 视频流基础上，实现了**系统 CPU 使用率和内存使用量**的实时监控，并将数据从服务器端传递到浏览器前端 UI，以进度环（`v-progress-circular`）形式展示。

## 架构与数据流

```
┌─────────────────────────────────────────────────────────────────┐
│  Termux 服务器 (Android)                                        │
│                                                                 │
│  webrtc_utils.py: SystemMonitor  (每1秒轮询)                     │
│       │                                                         │
│       ├── [Primary]   psutil.cpu_percent() + virtual_memory()   │
│       └── [Fallback]  /proc/self/stat + /proc/meminfo           │
│                         (Termux 无法访问 /proc/stat 时)          │
│       │                                                         │
│       ▼                                                         │
│  webrtc_mode.py: handle_system_monitor()                        │
│       │                                                         │
│       ├── rtc_app.send_system_stats(...)   → WebRTC DataChannel │
│       └── signaling_client.send_json(...)  → Signaling WebSocket│
│                                                                 │
│  signaling_server.py                                            │
│       │  (消息转发: server_uid + JSON)                          │
│       ▼                                                         │
│  WebSocket                                                      │
└───────┼─────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│  浏览器 (Chrome/Firefox)                                         │
│                                                                 │
│  signaling.js: 收到消息 → 解析 JSON → onjsonmessage 回调       │
│       │                                                         │
│       ▼                                                         │
│  app.js: msg.type === 'system_stats' → webrtc.onsystemstats()  │
│       │                                                         │
│       ▼                                                         │
│  UI: v-progress-circular 显示 CPU% / 内存使用                   │
└─────────────────────────────────────────────────────────────────┘
```

**两条路径同时发送**：DataChannel 和 Signaling WebSocket 都会发送同一条 stats 消息。浏览器收到两条相同的消息也无害（后一条覆盖前一条的值）。这确保了在 DataChannel 尚未就绪时，stats 可通过 Signaling WS 可靠到达。

---

## 修改文件清单

### 1. `src/selkies/webrtc_utils.py` — SystemMonitor 改造（核心）

**位置**: 第 731–786 行（`SystemMonitor` 类）

**原代码** (`_get_system_metrics`):
```python
def _get_system_metrics(self):
    try:
        cpu = psutil.cpu_percent()
        mem = psutil.virtual_memory()
        return cpu, mem.total, mem.used
    except Exception:
        return 0.0, 0, 0
```

**问题**: 在 Termux 环境下，`psutil.cpu_percent()` 读取 `/proc/stat` 时触发 `PermissionError`（Android 限制非 root 进程读取系统级 CPU 统计），导致永远返回 `(0.0, 0, 0)`。

**新架构**: 采用**分级回退**策略：

#### 1a. `_read_proc_system_stats()` (第 731 行)

```python
@staticmethod
def _read_proc_system_stats():
    # 1. 尝试 psutil（完整系统 CPU + 内存）
    try:
        mem = psutil.virtual_memory()
        cpu = psutil.cpu_percent()
        return cpu, mem.total, mem.used, True   # True = psutil 成功
    except Exception:
        pass

    # 2. 回退：直接从 /proc/meminfo 读取内存
    try:
        with open('/proc/meminfo') as f:
            for line in f:
                if line.startswith('MemTotal:'):
                    mem_total = int(line.split()[1]) * 1024
                elif line.startswith('MemAvailable:'):
                    mem_avail = int(line.split()[1]) * 1024
            if mem_total > 0:
                mem_used = mem_total - mem_avail
    except Exception:
        pass

    return cpu_pct, mem_total, mem_used, False  # False = psutil 失败
```

- `MemTotal` 和 `MemAvailable` 在 `/proc/meminfo` 中以 kB 为单位，乘以 1024 转为字节。
- 返回值的第四个元素 `psutil_ok` 用于指示 psutil 是否成功。

#### 1b. `_read_proc_cpu_percent()` (第 756 行)

```python
_last_cpu_time = 0.0    # 类级别：上一次累计 CPU 时间（秒）
_last_cpu_wall = 0.0    # 类级别：上一次 wall-clock 时间戳

@classmethod
def _read_proc_cpu_percent(cls):
    try:
        with open('/proc/self/stat') as f:
            parts = f.read().split()
            utime = int(parts[13])        # 用户态 CPU tick 数
            stime = int(parts[14])        # 内核态 CPU tick 数
        now = time.time()
        total_cpu = (utime + stime) / 100.0   # 1 tick = 0.01s (CLK_TCK=100)
        if cls._last_cpu_time > 0:
            dt = now - cls._last_cpu_wall
            if dt > 0:
                cpu = min(100.0, ((total_cpu - cls._last_cpu_time) / dt) * 100.0)
            else:
                cpu = 0.0
        else:
            cpu = 0.0
        cls._last_cpu_time = total_cpu
        cls._last_cpu_wall = now
        return cpu
    except Exception:
        return 0.0
```

- 读取当前进程（`/proc/self/stat`）的 `utime`（第14个字段）和 `stime`（第15个字段）。
- 除以 100（`CLK_TCK`，Linux 默认 100 ticks/秒）得到累计 CPU 秒数。
- 两次采样之间的差值 / 时间间隔 × 100% = 当前进程 CPU 使用率。
- 使用 `classmethod` + 类变量来跨实例共享上一次采样值。

**注意**: 这是**进程级别**的 CPU 使用率（selkies 自身的 CPU 消耗），而非系统级 CPU。在 Termux 无法访问 `/proc/stat` 的前提下，这是最准确的可用数据。

#### 1c. `_get_system_metrics()` (第 782 行)

```python
def _get_system_metrics(self):
    cpu, mem_total, mem_used, psutil_ok = self._read_proc_system_stats()
    if not psutil_ok:
        cpu = self._read_proc_cpu_percent()   # 只有 psutil 失败时才回退
    return cpu, mem_total, mem_used
```

---

### 2. `src/selkies/webrtc_mode.py` — 添加 Stats 的 Signaling WebSocket 回退路径

**位置**: 第 459–475 行（`handle_system_monitor` 方法）

**新增**: 第 88–90 行（`__init__` 初始化 `_client_peer_id`）

```python
self._client_peer_id: Optional[str] = None
```

**新增**: 第 223–225 行（`handle_session_start` 存储客户端 peer ID）

```python
async def handle_session_start(self, session_peer_id: str, client_type: str) -> None:
    self._client_peer_id = session_peer_id
    ...
```

**修改**: 第 459–475 行

```python
async def handle_system_monitor(self, t: float) -> None:
    if self.input_handler and self.rtc_app and self.system_monitor:
        self.input_handler.ping_start = t
        # 路径1: DataChannel (WebRTC)
        self.rtc_app.send_system_stats(
            self.system_monitor.cpu_percent,
            self.system_monitor.mem_total,
            self.system_monitor.mem_used
        )
        self.rtc_app.send_ping(t)
    # 路径2: Signaling WebSocket (更可靠)
    if self.signaling_client and self._client_peer_id and self.system_monitor:
        await self.signaling_client.send_json("system_stats", {
            "cpu_percent": self.system_monitor.cpu_percent,
            "mem_total": self.system_monitor.mem_total,
            "mem_used": self.system_monitor.mem_used,
        }, self._client_peer_id)
```

两条路径**同时发送**，不存在"主/备"切换逻辑。DataChannel 在 WebRTC 连接建立后可用但时间不确定；Signaling WS 全程可用。

---

### 3. `src/selkies/webrtc_signaling.py` — 新增 `send_json` 方法

**位置**: 第 115–118 行

```python
async def send_json(self, msg_type: str, data: dict, client_peer_id: str):
    """Sends an arbitrary JSON message to the peer."""
    msg = json.dumps({"type": msg_type, "data": data})
    await self.conn.send(str(client_peer_id) + ' ' + msg)
```

发送格式: `<目标 peer ID> <JSON>`，遵循 Signaling 协议的"前缀路由"机制。Signaling Server 收到后提取 `target_uid` 并转发，同时将前缀替换为发送方 uid。

---

### 4. `src/selkies/signaling_server.py` — 抑制 Stats 日志噪音

**位置**: 第 390 行

```python
if 'system_stats' not in msg_string:
    logger.info("{} -> {}: {}".format(uid, other_id, msg))
```

每秒一次的 stats 转发如果不加过滤会产生大量日志。过滤后保留 SDP/ICE/重连等关键消息的日志。

---

### 5. `addons/gst-web/src/signaling.js` — 前端 Signaling 客户端

#### 5a. `onjsonmessage` 回调接口（第 119–123 行）

```javascript
this.onjsonmessage = null;
```

#### 5b. HELLO 超时自动断开（第 212–219 行）

```javascript
this._hello_received = false;
if (this._hello_timeout) clearTimeout(this._hello_timeout);
this._hello_timeout = setTimeout(() => {
    if (!this._hello_received && this.state === 'connected') {
        this._setError("No HELLO response from server, reconnecting...");
        this._ws_conn.close();
    }
}, 8000);
```

- 连接 WebSocket 后启动 8 秒定时器。
- 如果 8 秒内未收到 `HELLO` 响应，认为服务器异常，主动断开并触发重连。
- `_onServerMessage()` 收到 `HELLO` 后设置 `_hello_received = true` 并清除定时器。
- 断开时（`_onServerClose`）清除定时器，防止内存泄漏。

#### 5c. JSON 消息分发（第 296–305 行）

```javascript
if (msg.sdp != null) {
    this._setSDP(new RTCSessionDescription(msg.sdp));
} else if (msg.ice != null) {
    var icecandidate = new RTCIceCandidate(msg.ice);
    this._setICE(icecandidate);
} else if (msg.type !== undefined && this.onjsonmessage !== null) {
    this.onjsonmessage(msg);
} else {
    this._setError("unhandled JSON message: " + msg);
}
```

收到消息时先剥去前端的 `<uid> <JSON>` 前缀（第 285 行），然后按照 SDP → ICE → 通用 JSON 的顺序分发。`system_stats` 这类自定义 JSON 走 `onjsonmessage` 路由。

---

### 6. `addons/gst-web/src/app.js` — 前端应用逻辑

**位置**: 第 394–398 行

```javascript
signaling.onjsonmessage = async (msg) => {
    if (msg.type === 'system_stats' && msg.data) {
        webrtc.onsystemstats(msg.data);
    }
};
```

- 过滤 `type === 'system_stats'` 的消息。
- 将 `msg.data`（即 `{cpu_percent, mem_total, mem_used}`）传递给 `webrtc.onsystemstats()`。
- `webrtc.onsystemstats()` 最终更新 UI 中的 `v-progress-circular` 组件。

---

### 7. `launch-selkies-new.sh` — 启动脚本

**位置**: 第 8–9 行

```bash
export PULSE_SERVER=tcp:127.0.0.1:4713
export SELKIES_AUDIO_ENABLED=true
```

- 启用音频（默认 Selkies 禁用音频，需显式设置 `SELKIES_AUDIO_ENABLED=true`）。
- 指定 PulseAudio TCP 连接地址（AAudio sink）。

---

## Termux / Android 兼容性详解

### 受限资源

| 文件/接口 | 可用性 | 原因 |
|-----------|--------|------|
| `/proc/stat` | ❌ Permission denied | Android SELinux 阻止非 root 读取系统 CPU 统计 |
| `/proc/loadavg` | ❌ Permission denied | 同上 |
| `/proc/meminfo` | ✅ 可读 | 内存信息不被视为敏感 |
| `/proc/self/stat` | ✅ 可读 | 进程自身统计 |
| `/proc/self/status` | ✅ 可读 | 进程自身状态 |
| `user namespaces` (`unshare`) | ❌ 不可用 | Android 内核默认禁用 |
| `vmstat` / `dumpsys` | ❌ 不存在 | Termux 中不可用 |

### 解决方案

- **内存**: 直接解析 `/proc/meminfo` 的 `MemTotal` 和 `MemAvailable`。
- **CPU**: 利用 `/proc/self/stat` 获取进程自身的 `utime` + `stime`，通过两次采样间隔计算 CPU 使用率。**不是系统 CPU 使用率**，但对于监控 selkies 自身负载已足够。
- **分层设计**: 优先尝试 `psutil`（标准 Linux 环境），异常时静默回退到 `/proc/*` 读取。

---

## 测试与验证

1. 启动 selkies: `bash launch-selkies-new.sh`
2. 浏览器打开 `http://<device-ip>:8081`
3. WebRTC 连接建立后，UI 中的 CPU 和 Memory 进度环显示实时数值
4. 日志中可以通过 `grep system_stats selkies.log` 查看每 1 秒的 stats 推送

## 关键文件索引

| 文件 | 作用 |
|------|------|
| `src/selkies/webrtc_utils.py` | `SystemMonitor` 类，CPU/内存数据采集（第 712 行） |
| `src/selkies/webrtc_mode.py` | orchestrator，`handle_system_monitor` 发送数据（第 459 行） |
| `src/selkies/webrtc_signaling.py` | `send_json` 方法通过 WS 发送任意 JSON（第 115 行） |
| `src/selkies/signaling_server.py` | WS 转发 + stats 日志静默（第 390 行） |
| `addons/gst-web/src/signaling.js` | 前端 WS 客户端，`onjsonmessage` + HELLO 超时 |
| `addons/gst-web/src/app.js` | 前端消息路由，`system_stats` → `webrtc.onsystemstats` |
| `addons/gst-web/src/index.html` | UI 定义，包含 CPU/内存 `v-progress-circular` |
| `launch-selkies-new.sh` | 启动脚本，配置音频和编码器 |
