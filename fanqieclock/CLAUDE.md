# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

番茄钟 (Pomodoro Timer) — 纯前端计时器，单文件应用，零依赖。

## 运行方式

```bash
# 直接在浏览器中打开即可
start index.html

# 或使用静态服务器
python -m http.server 8000
```

## 架构概要

`index.html` 包含全部代码（HTML 结构 + CSS 样式 + JS 逻辑），约 1200 行。JavaScript 使用 IIFE 包裹，不污染全局作用域。

### 核心状态对象 `state`

所有逻辑围绕一个 `state` 对象展开（单一数据源），字段包括：
- `mode`: `'work'` | `'shortBreak'` | `'longBreak'`
- `timeLeft` / `totalTime`: 剩余秒数和阶段总秒数
- `isRunning`: 计时器是否运行中
- `completedPomodoros`: 今日已完成番茄总数
- `settings`: 时长配置（秒），含 `work` / `shortBreak` / `longBreak`
- `timerId`: `setInterval` 句柄

### 状态机规则

- 专注(work) → 短休息(shortBreak)，每 4 个番茄后 → 长休息(longBreak)
- 任意休息 → 专注
- 状态转换发生在 `handleTimerEnd()` → `switchMode()`

### SVG 进度环原理

- 圆半径 136px，周长 `2 × π × 136 ≈ 854.51`
- `stroke-dasharray="854.51"` 设为圆周长
- 通过动态修改 `stroke-dashoffset` 控制进度：`0` = 满圆，`854.51` = 空圆
- SVG 默认从 3 点钟方向开始，通过 `transform: rotate(-90deg)` 旋转到 12 点钟

### 计时器精度策略

- 不使用每秒递减 1 的方式（会累积误差）
- 记录启动时时间戳 `startTime`，每次 tick 计算 `已过秒数 = (Date.now() - startTime) / 1000`
- `timeLeft = max(0, 原始剩余 - 已过秒数)`

### 页面不可见时的修正

- 浏览器会降频后台标签页的 `setInterval`
- `visibilitychange` 事件：隐藏时记录快照（`hiddenAt`, `hiddenTimeLeft`），恢复时计算实际经过时间进行修正

### 持久化

- 键名：`pomodoro_state_v2`（localStorage）
- 存储内容：`{ settings, completedPomodoros, date }`
- 跨天自动重置番茄计数（通过比对 `date` 字段与 `today()` 返回值）

### 音频（Web Audio API）

- `AudioContext` 延迟初始化，在用户首次交互（click/keydown）时才创建（浏览器自动播放策略要求）
- 提示音为三段上行音阶：880Hz → 1100Hz → 1320Hz
- 使用 `OscillatorNode` 合成，无外部音频文件
