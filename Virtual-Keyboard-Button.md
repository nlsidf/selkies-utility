# 虚拟键盘按钮 + 扩展功能键面板

## 概述

在 Android 设备上使用 Selkies 远程桌面时，Android 设备本身通常没有物理键盘。本功能提供了两种输入方式：

1. **虚拟键盘按钮** — 点击一次弹出 Android 系统软键盘（IME）
2. **扩展功能键面板** — 双击按钮弹出功能键面板（Esc、Ctrl、Alt、F1-F12 等）

两者都设计为半透明浮动 UI，不干扰视频观看。

---

## 效果

### 虚拟键盘按钮（单击）

| 操作 | 效果 |
|------|------|
| 单击键盘按钮 | Android 软键盘弹出/隐藏 |
| 点击视频画面 | 自动隐藏键盘 |

### 扩展功能键面板（双击）

| 操作 | 效果 |
|------|------|
| 双击键盘按钮 | 功能键面板弹出/隐藏 |
| 点击面板外的区域 | 关闭面板 |
| 点击面板内的按键 | 发送对应的 keysym 到远程桌面 |
| 点击 Ctrl/Alt/Shift/Win | 切换修饰键锁定状态（再次点击释放） |

---

## 技术原理

### 虚拟键盘流程

```
单击按钮
  → 250ms 内无第二次点击 → 执行 toggleKeyboard()
  → 聚焦 <input> (keyboard-input-assist)
  → Android IME 弹出
  → 用户按键 → keydown/keyup 冒泡到 window
  → Guacamole.Keyboard 捕获 → 发送 keysym
```

### 扩展功能键流程

```
双击按钮（250ms 内两次点击）
  → 取消单次点击的键盘切换
  → 生成/显示功能键面板
  → 点击按键 → sendKey(keysym) → "kd,<keysym>" + "ku,<keysym>"
  → 修饰键点击 → 切换锁定状态（视觉高亮）
  → 点击面板外 → 隐藏面板 + 释放所有锁定的修饰键
```

---

## 界面设计

### 键盘按钮

右下角固定（bottom:16px, right:16px），44×44px，半透明黑色背景 + `backdrop-filter: blur(4px)`，Material Design 键盘 SVG 图标。

### 功能键面板

右下角弹出（bottom:68px, right:8px），紧贴键盘按钮上方，深色毛玻璃背景 `rgba(20,20,25,0.88)` + `backdrop-filter: blur(12px)`，圆角 14px，带阴影。

按键为小型圆角矩形，5 列网格布局，共 7 行：

```
 Row 0:  Esc    Tab    Enter  Bsp    Del
 Row 1:  PrtSc  Ins    Pause   ↑      ↓
 Row 2:   ←      →     Home   End    PgUp
 Row 3:  PgDn   Ctrl*  Alt*   Shift* Win*
 Row 4:  F1     F2     F3     F4     F5
 Row 5:  F6     F7     F8     F9     F10
 Row 6:  F11    F12
```

*修饰键（Ctrl/Alt/Shift/Win）支持 toggle 锁定，点击后保持按下状态（蓝色高亮），再次点击释放。

---

## 文件修改

### `addons/gst-web/src/index.html`（Vue.js 前端 — 实际部署版本）

**CSS 新增样式**（`<style>` 块内）：

| 选择器 | 作用 |
|--------|------|
| `.virtual-keyboard-btn.tapped` | 按钮点击动画反馈 |
| `.func-keys-panel` | 功能键面板容器（fixed 定位，毛玻璃背景）|
| `.func-keys-panel.show` | 面板显示状态 |
| `.func-keys-row` | 每行 flex 布局 |
| `.func-key` | 单个功能键样式 |
| `.func-key:active` | 按键按下反馈 |
| `.func-key.mod-active` | 修饰键锁定状态（蓝色边框）|
| `.func-key.arrow` | 方向键（更大字号）|

**HTML 新增元素**：

| 元素 | ID | 作用 |
|------|-----|------|
| `<input>` | `keyboard-input-assist` | 隐藏输入框，触发 Android IME |
| `<button>` | `vkBtn` | 键盘唤起按钮（无 inline onclick） |
| `<div>` | `funcKeysPanel` | 功能键面板容器 |
| `<div>` | `funcOverlay` | 透明遮罩层，点击关闭面板 |

**JavaScript 逻辑**（`app.js` 后加载的内联脚本）：

| 函数/变量 | 作用 |
|-----------|------|
| `sendKey(keysym)` | 发送 kd + ku keysym 到 WebRTC 数据通道 |
| `toggleKeyboard()` | 切换 Android 软键盘显示 |
| `FUNC_KEYS` | 功能键数据定义（7 行 × 5 列） |
| `modState` | 修饰键锁定状态追踪 |
| `buildPanel()` | 动态生成功能键面板 DOM |
| `showPanel()` / `hidePanel()` | 面板显示/隐藏 |
| `togglePanel()` | 切换面板状态 |
| `tapTimer` | 双击检测定时器（250ms 阈值） |
| `btn.click` 事件 | 双击检测 + 单击/双击分发 |

### `addons/gst-web-core/selkies-wr-core.js`（WebRTC 备用前端）

- 仅含虚拟键盘按钮（无功能键面板）

### `addons/gst-web-core/selkies-ws-core.js`（WebSocket 备用前端）

- 仅含虚拟键盘按钮（无功能键面板）

---

## 功能键 Keysym 映射

| 按键 | Keysym | 按键 | Keysym |
|------|--------|------|--------|
| Esc | 0xFF1B | Tab | 0xFF09 |
| Enter | 0xFF0D | Backspace | 0xFF08 |
| Delete | 0xFFFF | PrintScreen | 0xFF61 |
| Insert | 0xFF63 | Pause | 0xFF13 |
| ↑ | 0xFF52 | ↓ | 0xFF54 |
| ← | 0xFF51 | → | 0xFF53 |
| Home | 0xFF50 | End | 0xFF57 |
| PageUp | 0xFF55 | PageDown | 0xFF56 |
| Ctrl | 0xFFE3 | Alt | 0xFFE9 |
| Shift | 0xFFE1 | Super/Win | 0xFFEB |
| F1-F12 | 0xFFBE-0xFFC9 | | |

---

## 用户体验要点

- **无干扰**：面板为半透明毛玻璃效果，不遮挡视频内容
- **双击检测**：250ms 窗口内两次点击视为双击，不延迟单击响应
- **修饰键锁定**：Ctrl/Alt/Shift/Win 点击后保持按下，再次点击释放（类似 sticky keys）
- **自动清理**：面板关闭时自动释放所有锁定的修饰键
- **遮罩关闭**：点击面板外部任意区域即可关闭
