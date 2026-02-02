---
summary: "用于 Agent 使用的相机捕获（iOS 节点 + macOS 应用）：照片（jpg）和短视频片段（mp4）"
read_when:
  - 在 iOS 节点或 macOS 上添加或修改相机捕获
  - 扩展 Agent 可访问的 MEDIA 临时文件工作流程
title: "相机捕获"
---

# 相机捕获（Agent）

OpenClaw 支持用于 Agent 工作流程的**相机捕获**：

- **iOS 节点**（通过网关配对）：通过 `node.invoke` 捕获**照片**（`jpg`）或**短视频片段**（`mp4`，可选音频）。
- **Android 节点**（通过网关配对）：通过 `node.invoke` 捕获**照片**（`jpg`）或**短视频片段**（`mp4`，可选音频）。
- **macOS 应用**（通过网关的节点）：通过 `node.invoke` 捕获**照片**（`jpg`）或**短视频片段**（`mp4`，可选音频）。

所有相机访问都受**用户控制设置**的限制。

## iOS 节点

### 用户设置（默认开启）

- iOS 设置标签页 → **相机** → **允许相机** (`camera.enabled`)
  - 默认：**开启**（缺少键被视为启用）。
  - 关闭时：`camera.*` 命令返回 `CAMERA_DISABLED`。

### 命令（通过 Gateway `node.invoke`）

- `camera.list`
  - 响应负载：
    - `devices`：`{ id, name, position, deviceType }` 的数组

- `camera.snap`
  - 参数：
    - `facing`：`front|back`（默认：`front`）
    - `maxWidth`：数字（可选；iOS 节点默认 `1600`）
    - `quality`：`0..1`（可选；默认 `0.9`）
    - `format`：当前为 `jpg`
    - `delayMs`：数字（可选；默认 `0`）
    - `deviceId`：字符串（可选；来自 `camera.list`）
  - 响应负载：
    - `format: "jpg"`
    - `base64: "<...>"`
    - `width`、`height`
  - 负载保护：照片会被重新压缩以保持 base64 负载在 5 MB 以下。

- `camera.clip`
  - 参数：
    - `facing`：`front|back`（默认：`front`）
    - `durationMs`：数字（默认 `3000`，最大限制为 `60000`）
    - `includeAudio`：布尔值（默认 `true`）
    - `format`：当前为 `mp4`
    - `deviceId`：字符串（可选；来自 `camera.list`）
  - 响应负载：
    - `format: "mp4"`
    - `base64: "<...>"`
    - `durationMs`
    - `hasAudio`

### 前台要求

与 `canvas.*` 类似，iOS 节点仅在**前台**允许 `camera.*` 命令。后台调用返回 `NODE_BACKGROUND_UNAVAILABLE`。

### CLI 辅助工具（临时文件 + MEDIA）

获取附件的最简单方式是通过 CLI 辅助工具，它将解码后的媒体写入临时文件并打印 `MEDIA:<path>`。

示例：

```bash
openclaw nodes camera snap --node <id>               # 默认：前后双摄（2 行 MEDIA）
openclaw nodes camera snap --node <id> --facing front
openclaw nodes camera clip --node <id> --duration 3000
openclaw nodes camera clip --node <id> --no-audio
```

注意：

- `nodes camera snap` 默认为**前后双摄**，以便为 Agent 提供两个视角。
- 输出文件是临时的（在 OS 临时目录中），除非您构建自己的包装器。

## Android 节点

### 用户设置（默认开启）

- Android 设置面板 → **相机** → **允许相机** (`camera.enabled`)
  - 默认：**开启**（缺少键被视为启用）。
  - 关闭时：`camera.*` 命令返回 `CAMERA_DISABLED`。

### 权限

- Android 需要运行时权限：
  - `CAMERA` 用于 `camera.snap` 和 `camera.clip`。
  - 当 `includeAudio=true` 时，`camera.clip` 需要 `RECORD_AUDIO`。

如果权限缺失，应用会在可能时提示；如果被拒绝，`camera.*` 请求将失败并返回 `*_PERMISSION_REQUIRED` 错误。

### 前台要求

与 `canvas.*` 类似，Android 节点仅在**前台**允许 `camera.*` 命令。后台调用返回 `NODE_BACKGROUND_UNAVAILABLE`。

### 负载保护

照片会被重新压缩以保持 base64 负载在 5 MB 以下。

## macOS 应用

### 用户设置（默认关闭）

macOS 配套应用暴露一个复选框：

- **设置 → 通用 → 允许相机** (`openclaw.cameraEnabled`)
  - 默认：**关闭**
  - 关闭时：相机请求返回"用户已禁用相机"。

### CLI 辅助工具（节点调用）

使用主 `openclaw` CLI 在 macOS 节点上调用相机命令。

示例：

```bash
openclaw nodes camera list --node <id>            # 列出相机 id
openclaw nodes camera snap --node <id>            # 打印 MEDIA:<path>
openclaw nodes camera snap --node <id> --max-width 1280
openclaw nodes camera snap --node <id> --delay-ms 2000
openclaw nodes camera snap --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --duration 10s          # 打印 MEDIA:<path>
openclaw nodes camera clip --node <id> --duration-ms 3000      # 打印 MEDIA:<path>（旧版标志）
openclaw nodes camera clip --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --no-audio
```

注意：

- `openclaw nodes camera snap` 默认使用 `maxWidth=1600`，除非被覆盖。
- 在 macOS 上，`camera.snap` 在预热/曝光稳定后等待 `delayMs`（默认 2000ms）再捕获。
- 照片负载会被重新压缩以保持 base64 在 5 MB 以下。

## 安全 + 实际限制

- 相机和麦克风访问会触发常规的 OS 权限提示（并需要在 Info.plist 中使用说明字符串）。
- 视频片段有上限（目前 `<= 60s`）以避免过大的节点负载（base64 开销 + 消息限制）。

## macOS 屏幕视频（OS 级别）

对于*屏幕*视频（非相机），使用 macOS 配套应用：

```bash
openclaw nodes screen record --node <id> --duration 10s --fps 15   # 打印 MEDIA:<path>
```

注意：

- 需要 macOS **屏幕录制**权限（TCC）。
