# niofireflyplay

##  项目简介

利用车机自带浏览器，通过高通底层硬件零缓冲编码与浏览器 `WebCodecs` + `2D Canvas (desynchronized)` 硬件图层直通技术，将一台闲置 Root 安卓手机的画面以 **2560×1116 原生满屏分辨率、60FPS 12M码率、20~30ms 物理极限低延迟** 投射至蔚来萤火虫（Firefly）及各类现代车机中控屏，完美支持 10 指多点触控反控、手机物理屏幕彻底熄屏零发光，以及 `start` / `stop` 一键切换与恢复手机原生竖屏。

说明：

萤火虫屏幕分辨率是2560*1600的16:10屏幕，但是浏览器全屏状态下非常容易触发萤火虫顶层系统设置的“下滑退出全屏”以及车机设置强制退出全屏模式，体验并不好，权衡之后，我还是改为保留上下黑边、车机状态栏、Dock栏，因此分辨率是2560*1116；如果你需要全屏，让ai加一个全屏按钮、分辨率调整为2560*1600即可；再有，为了得到一块接近于安卓平板的体验，系统越新越好；

如果你只是奔着Apple CarPlay而来，这个项目可能不是最好的选择，理论上安装DiPlay一样能用，但是众所周知CarPlay至今令人诟病的延迟就高达70ms+，叠加本项目至少100ms以上，几乎没有体验可言！如果CarPlay刚需，诸如CarPodgo mini之类的linux专业车载屏幕仍是最佳选择。

另外，如果你没有备用机root玩本项目，而手中又有一部主力安卓设备+iPhone，又想体验CarPlay，那强烈推荐你去关注 wheelplay这个项目：github.com/fython/wheelplay，相当于DiPlay的离屏渲染版本，手机安装apk，车机浏览器打开就能使用CarPlay，并且手机锁屏、其余操作完全不影响投屏，非常优雅聪明！

最后，这已经是我找到最好的方案了。中途已经尝试过“安卓副屏”的想法，并且成功实现，但是安卓目前做不到类似macbook那样盒盖关闭主屏、副屏操作系统的逻辑，而是类似于“拓展屏”，只要手机关闭电源键，副屏就关闭了。另外安卓的副屏本身体验就很差，应用适配、逻辑一团糟，故放弃了。

---

##  一、 必读安全警告与风险声明（动手前请务必逐条确认）

1. **必须使用闲置备用机（严禁使用日常主力机）**
   - 本方案需要对手机解锁 Bootloader（BL 锁）、获取 Root 权限，并频繁通过底层指令修改系统显示分辨率（`wm size`）、DPI、屏幕刷新率以及内核背光节点。使用主力机操作会严重干扰日常使用。
2. **数据清空与变砖风险**
   - 解锁手机 Bootloader 会 **100% 彻底清空手机内所有数据**。刷入 Magisk / KernelSU 获取 Root 权限或操作不当可能导致系统无法开机（卡 Fastboot 或 9008 模式），请确保使用无重要数据的闲置设备，并具备基础刷机救砖常识。
3. **车内高温与锂电池起火/鼓包风险（极度重要！）**
   - 备用机若长期放置在车内，夏季暴晒或密闭车厢高温极易引发锂电池鼓包甚至热失控起火。
   - **强制安全规范**：
     - **严禁将手机放在阳光直射的中控台表面或前挡风玻璃下**，请放置在手套箱、扶手箱或有空调出风口散热的阴凉位置。
     - **必须开启充电限制（电量保护）**：通过 Root 充电控制模块（如 ACC - Advanced Charging Controller）或系统自带的“充电保护/旁路充电”功能，将手机电量严格锁定在 **60% ~ 75% 之间**，严禁长期插着充电线维持 100% 满电高电压浮充状态。有动手能力的车友强烈建议对备用机进行“拆电池改直供电”改造。

---

##  二、 技术原理与方案优缺点

### 1. 核心架构原理
- **采集与编码端（手机）**：通过 Root 权限直接拉起 `scrcpy-server` (v3.1) 底层进程，绕过安卓常规录屏弹窗，直接从 `SurfaceFlinger` 抓取屏幕合成图层，并激活高通骁龙硬件编码器私有低延迟通道（`low-latency=1`、`max-bframes=0`、`vendor.qti-ext-enc-low-latency.enable=1`），消除多帧前瞻缓冲（Lookahead Buffer），收到一帧即刻编码吐出 H.264 裸流。
- **零拷贝传输桥（手机 Termux）**：Node.js 服务通过 Linux 抽象命名空间套接字（UNIX Domain Socket）直接读取视频流与写入触控指令，并通过开启了 `TCP_NODELAY` 的 HTTPS/WSS 加密通道向局域网推送。
- **硬件解码与直通渲染（车机浏览器）**：车机 Chromium 浏览器利用 `WebCodecs API (VideoDecoder)` 调用座舱芯片硬解视频流；本教程提供两套经过实测的前端渲染引擎：
  - **主推版（2D Canvas 极速直通版，延迟 20~30ms）**：开启 `desynchronized: true` 绕过浏览器合成器排队，解码完微秒级直接上屏（不等待 `requestAnimationFrame`），并启用 `pointerrawupdate` 硬件级触控中断采样。
  - **备选版（WebGL 垂直同步零撕裂版，延迟 45~50ms）**：采用单槽位信箱模式（Mailbox `latestFrame`）配合 `requestAnimationFrame` 严格对齐车机 60Hz VSync，画面 100% 零撕裂。

### 2. 方案优缺点一览

| 维度 | 核心优势 | 局限性与缺点 |
| :--- | :--- | :--- |
| **延迟表现** | **20~30ms 低延迟**，滑动跟手度媲美车机原生系统 | 2D 极速版在快速横滑桌面时偶有轻微扫描线撕裂（可切 WebGL 版消除） |
| **画质与帧率** | **2560×1116 12M码率点对点满屏无黑边**，手机与车机同频锁定 60FPS | 依赖局域网质量，必须使用 5GHz Wi-Fi 频段 |
| **触控反控** | **原生 10 指多点触控**，支持高德/百度地图双指丝滑缩放与旋转 | 手机物理屏幕熄灭后，手机本体的触摸层依然通电（需放置好防误触） |
| **音频通道** | 视频流极致纯净，不占用额外无线带宽与解码耗时 | 网页端默认仅传画面，音频需通过手机连接车载蓝牙输出 |

---

##  三、 软硬件与网络环境要求（含课前自检）

### 1. 手机硬件要求
- **处理器**：强烈推荐 **高通骁龙 855 / 865 / 870 / 888 / 8 Gen 1 及以上** 机型（作者实测机型：**一加 8 LingeaOS 23.2 / OnePlus 8**）。代码中调用了高通专属硬件低延迟编码扩展，非高通芯片需删除该扩展参数。
- **屏幕材质**：必须为 **AMOLED / OLED 屏幕**（“关闭手机背光零发热推流”功能依赖 OLED 像素自发光特性；LCD 屏幕关闭背光后液晶层仍工作且部分内核会挂起渲染管线）。
- **Root 权限**：必须已刷入 **Magisk、KernelSU 或 APatch**，具备完整 `su` 权限。

### 2. 手机软件要求
- **Android 版本**：Android 11 ~ Android 16 均可（脚本已专门针对 Android 14~16 最严苛的 SELinux 与 Binder 终端句柄限制 `2147483646` 做了穿透适配）。
- **必备 APK**：仅需在手机上安装 **Termux** 一个软件（请从 GitHub Releases 或 F-Droid 下载官方正式版）。

### 3. 网络环境要求
- **必须使用 5GHz Wi-Fi**：萤火虫 2.1.0系统已经是融合模式，近距离自动接手 5GHz 车载热点，备用手机连接车机热点（严禁使用 2.4GHz，2.4GHz 受车载蓝牙同频干扰会导致严重掉帧断流）。
- **首次部署外网要求**：在第五节第 1~2 步下载 Node.js 和依赖包时，请先让手机连接家里有外网的 Wi-Fi（或个人热点）。

### 【硬件与 Root 权限自检】
打开手机上的 **Termux**，复制以下自检命令并回车（首次运行会弹出 Magisk/KernelSU 超级用户授权提示，请勾选“永远记住”并点击 **允许/Grant**）：

```bash
su -c '
  echo "=== 1. Root Check ==="
  id
  echo "=== 2. Backlight Node Check ==="
  ls -l /sys/class/backlight/*/brightness
'
```
> ** 结果验证（必看）**：
> 1. 第一部分必须输出包含 `uid=0(root)` 的字样（若提示 `Permission denied` 说明未授予 Termux Root 权限）。
> 2. 第二部分必须输出类似 `/sys/class/backlight/panel0-backlight/brightness` 的路径。确认无误后方可进入下一步。

---

##  四、 必做的系统设置（含逐步验证）

请严格按顺序完成以下系统设置：

1. **开启开发者选项与充电常亮**
   - 进入手机 `设置` -> `关于本机`，连续点击 7 次“版本号”开启开发者模式。
   - 进入 `设置` -> `系统` -> `开发者选项`：
     - 开启 **“不锁定屏幕（充电时屏幕不会休眠）”**。
     - 开启 **“USB 调试”**。
     - 找到 **“始终开启移动数据网络”** 并将其 **关闭**（极其重要：防止车机热点无外网时手机频繁尝试切网导致局域网断流）。
2. **关闭后台 Wi-Fi 与蓝牙周期性扫描（消除每隔几十秒微卡顿的元凶）**
   - 安卓默认每隔几十秒会让无线网卡切离当前通信信道去扫描周边 Wi-Fi，导致串流周期性卡顿。
   - 进入手机 `设置` -> `位置信息` -> `位置信息服务`：将 **“WLAN 扫描”** 和 **“蓝牙扫描”** 全部 **关闭**。
3. **固定车机热点静态 IP 与关闭随机 MAC**
   - 先去车上打开萤火虫车机热点，用手机连接车机热点后，点开手机上该 Wi-Fi 右侧的齿轮设置图标：
     - **隐私**：改为 **“使用设备 MAC”**（防止重连后 IP 变动）。
     - **按流量计费**：改为 **“视为不按流量计费”**（防止系统限制后台速率）。
     - **IP 设置**：建议将 DHCP 改为 **“静态（Static）”**，将当前分配到的车机局域网 IP（例如 `192.168.121.56`）固定保存，并记下这个 IP。
4. **解除 Termux 电池后台限制**
   - 进入手机 `设置` -> `应用` -> `Termux` -> `电池（应用电池使用情况）`：设置为 **“无限制（Unrestricted / 不优化）”**，并在多任务界面将 Termux 卡片上锁。

### 【系统设置底层加固与结果验证】
在 Termux 中复制运行以下命令，在系统底层永久关闭 Android 14~16 幽灵进程杀手（Phantom Process Killer）与后台 Wi-Fi 扫描：

```bash
su -c '
  /system/bin/device_config set_sync_disabled_for_tests persistent 2>/dev/null
  /system/bin/device_config put activity_manager max_phantom_processes 2147483647 2>/dev/null
  /system/bin/settings put global settings_enable_monitor_phantom_procs false 2>/dev/null
  /system/bin/cmd wifi set-scan-always-available disabled 2>/dev/null
  /system/bin/settings put global wifi_scan_always_enabled 0 2>/dev/null
  echo "Wi-Fi Scan Status (Expected 0): $(/system/bin/settings get global wifi_scan_always_enabled)"
  echo "Phantom Process Monitor (Expected false): $(/system/bin/settings get global settings_enable_monitor_phantom_procs)"
'
```
> ** 结果验证**：终端最后两行必须分别输出 `0` 和 `false`。

---

##  五、 核心环境安装与基础推流部署（含国内网络防坑验证）

> ** 网络提示**：第 1 步和第 2 步需要从网络下载依赖包，请确保此时手机连接的是**能够正常访问互联网的 Wi-Fi 或手机流量**！

### 第 1 步：切换 Termux 国内清华源并安装 Node.js 等基础工具

国内直接运行 `pkg update` 极易卡死，请复制以下脚本自动切换清华镜像源并安装所需环境：

```bash
# Switch Termux repository to Tsinghua University mirror and install dependencies
sed -i 's@^\(deb.*stable main\)$@#\1\ndeb https://mirrors.tuna.tsinghua.edu.cn/termux/apt/termux-main stable main@' $PREFIX/etc/apt/sources.list
pkg update -y && pkg install -y nodejs openssl curl procps psmisc net-tools

echo "=== Step 1 Verification ==="
node -v && npm -v && openssl version
```
> ** 第 1 步结果验证**：终端最后必须正常打印出三行版本号（如 `v20.x.x`、`10.x.x`、`OpenSSL 3.x.x`）。若提示 `command not found`，请检查手机网络连接后重试。

---

### 第 2 步：安装 WebSocket 依赖、下载 `scrcpy-server` 与生成 HTTPS 证书

很多车友因为国内网络无法直连 GitHub，导致下载的 `scrcpy-server.jar` 是 0 字节空文件或报错网页，进而导致车机黑屏。以下脚本内置了国内 npm 镜像与 GitHub 加速自动回退校验机制：

```bash
# 1. Create workspace and install ws module via domestic npm mirror
mkdir -p ~/car-ws && cd ~/car-ws
npm config set registry https://registry.npmmirror.com
npm init -y && npm install ws

# 2. Download scrcpy-server v3.1 (use -fL to fail properly on HTTP errors and fallback)
curl -fL --connect-timeout 10 -o ~/car-ws/scrcpy-server.jar https://ghfast.top/https://github.com/Genymobile/scrcpy/releases/download/v3.1/scrcpy-server-v3.1 \
  || curl -fL -o ~/car-ws/scrcpy-server.jar https://github.com/Genymobile/scrcpy/releases/download/v3.1/scrcpy-server-v3.1

# 3. Deploy scrcpy-server.jar to /data/local/tmp with proper permissions
su -c 'cp /data/data/com.termux/files/home/car-ws/scrcpy-server.jar /data/local/tmp/scrcpy-server.jar && chmod 644 /data/local/tmp/scrcpy-server.jar'

# 4. Generate self-signed SSL certificates (Required by WebCodecs API in car browser)
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout /data/data/com.termux/files/home/car-ws/key.pem \
  -out /data/data/com.termux/files/home/car-ws/cert.pem \
  -subj "/CN=car-mirror"

echo "=== Step 2 Verification ==="
ls -lh /data/local/tmp/scrcpy-server.jar
ls -lh ~/car-ws/key.pem ~/car-ws/cert.pem ~/car-ws/node_modules/ws/package.json
```
> **第 2 步结果验证（极其重要！）**：
> 1. 检查第一行输出的 `/data/local/tmp/scrcpy-server.jar` 文件大小，**必须显示为 `69K` ~ `72K` 左右**！如果显示为 `0`、几百字节或报错不存在，说明 GitHub 下载被阻断，请开启科学上网后重新执行第 2 步。
> 2. 确认 `key.pem`、`cert.pem` 和 `ws/package.json` 均正常列出。

---

### 第 3 步：部署 60帧 2D Canvas 极速推流核心代码（含 `start` / `stop` 自动切换）

本核心脚本将生成完整的服务端与控制命令：
- 输入 **`start`**：手机自动切换为萤火虫满屏分辨率（**2560×1116**，DPI **290**，强制横屏，锁定 60Hz 刷新率），赋予推流进程 `-20` 最高 CPU 优先级并启动后台推流。
- 输入 **`stop`**：停止推流，手机自动唤醒点亮屏幕，并**恢复手机原始默认分辨率、默认出厂 DPI、竖屏模式**。
- *(注：基础状态默认保持手机屏幕点亮以便首次调试；自动熄屏与开机自启已在第七节模块化分离，一行命令即可开启。)*

复制以下整段代码到 Termux 运行：

```bash
# === 1. Clean up old aliases and unlock backlight permissions ===
unalias stop 2>/dev/null || true
unalias start 2>/dev/null || true
unset -f stop 2>/dev/null || true
unset -f start 2>/dev/null || true
sed -i '/alias stop/d' ~/.bashrc 2>/dev/null || true
sed -i '/alias start/d' ~/.bashrc 2>/dev/null || true
sed -i '/alias stop/d' /data/data/com.termux/files/usr/etc/bash.bashrc 2>/dev/null || true

su -c '
  for f in /sys/class/backlight/*/brightness; do
    chmod 666 "$f" 2>/dev/null
  done
'

# === 2. Write server.js (Verified 60FPS 2D-Canvas Ultra-Low Latency Edition) ===
cat << 'EOF' > /data/data/com.termux/files/home/car-ws/server.js
const https = require('https');
const fs = require('fs');
const net = require('net');
const os = require('os');
const { spawn, execSync } = require('child_process');
const { WebSocketServer } = require('ws');

// Kill any leftover scrcpy server processes before starting
try { execSync('killall -9 app_process 2>/dev/null || true'); } catch (e) {}

let configPacket = null;
let lastKeyframe = null;
let ctrlSocket = null;
const clients = new Set();

// Pre-allocate 32-byte buffer for Scrcpy INJECT_TOUCH_EVENT protocol
const scrcpyTouchMsg = Buffer.alloc(32);
scrcpyTouchMsg[0] = 2; // Message type: INJECT_TOUCH_EVENT
scrcpyTouchMsg.writeUInt16BE(0xffff, 22); // Pressure: 1.0
scrcpyTouchMsg.writeInt32BE(1, 24); // Action button: AMOTION_EVENT_BUTTON_PRIMARY
scrcpyTouchMsg.writeInt32BE(1, 28); // Buttons state

const html = `<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no,viewport-fit=cover">
<meta name="mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-capable" content="yes">
<title>虚拟车机</title>
<style>
  * { margin:0; padding:0; box-sizing:border-box; user-select:none; -webkit-user-select:none; }
  html, body { width:100%; height:100%; background:#000; overflow:hidden; touch-action:none; display:flex; align-items:center; justify-content:center; }
  canvas { max-width:100vw; max-height:100vh; width:100%; height:100%; object-fit:contain; display:block; }
</style>
</head>
<body>
<canvas id="v"></canvas>
<script>
const canvas = document.getElementById('v');

// 2D Canvas origin is top-left; desynchronized: true bypasses compositor queue without coordinate flip bugs
const ctx = canvas.getContext('2d', {
  alpha: false,
  desynchronized: true
});

let decoder = null;
let configured = false;
let waitingForKey = true;
let configData = null;
let ts = 0;

function hex2(b) { return ('0' + b.toString(16)).slice(-2).toUpperCase(); }

function initDecoder() {
  configured = false;
  waitingForKey = true;
  decoder = new VideoDecoder({
    output(frame) {
      // Ultra-low latency: drop current frame if a newer frame is already waiting in decode queue
      if (decoder.decodeQueueSize > 0) {
        frame.close();
        return;
      }
      if (canvas.width !== frame.displayWidth || canvas.height !== frame.displayHeight) {
        canvas.width = frame.displayWidth;
        canvas.height = frame.displayHeight;
      }
      // Draw immediately upon hardware decode completion without waiting for requestAnimationFrame
      ctx.drawImage(frame, 0, 0);
      frame.close();
    },
    error() { waitingForKey = true; }
  });
}

initDecoder();

document.addEventListener('visibilitychange', () => {
  if (!document.hidden) waitingForKey = true;
});

const proto = location.protocol === 'https:' ? 'wss:' : 'ws:';
const ws = new WebSocket(proto + '//' + location.host + '/stream');
ws.binaryType = 'arraybuffer';

ws.onclose = () => { setTimeout(() => location.reload(), 1000); };

ws.onmessage = (ev) => {
  const buf = new Uint8Array(ev.data);
  if (buf.length < 2) return;
  const isKey = buf[0] === 1;
  const isConfig = buf[1] === 1;
  const payload = buf.subarray(2);

  if (isConfig) {
    configData = payload;
    let offset = 0;
    while (offset < payload.length - 4) {
      if (payload[offset] === 0 && payload[offset+1] === 0 && (payload[offset+2] === 1 || (payload[offset+2] === 0 && payload[offset+3] === 1))) {
        const nalOffset = (payload[offset+2] === 1) ? offset + 3 : offset + 4;
        const type = payload[nalOffset] & 0x1f;
        // Parse SPS NAL unit (type 7) to configure WebCodecs hardware decoder dynamically
        if (type === 7) {
          const codecStr = 'avc1.' + hex2(payload[nalOffset+1]) + hex2(payload[nalOffset+2]) + hex2(payload[nalOffset+3]);
          if (!configured) {
            try {
              decoder.configure({ codec: codecStr, optimizeForLatency: true, hardwareAcceleration: 'prefer-hardware' });
              configured = true;
            } catch (e) {}
          }
          break;
        }
      }
      offset++;
    }
    return;
  }

  if (!configured) return;

  if (decoder.decodeQueueSize > 3) {
    waitingForKey = true;
  }

  if (waitingForKey) {
    if (!isKey) return;
    waitingForKey = false;
  }

  try {
    ts += 16666;
    if (isKey && configData) {
      const merged = new Uint8Array(configData.length + payload.length);
      merged.set(configData, 0);
      merged.set(payload, configData.length);
      decoder.decode(new EncodedVideoChunk({ type: 'key', timestamp: ts, data: merged }));
    } else {
      decoder.decode(new EncodedVideoChunk({ type: isKey ? 'key' : 'delta', timestamp: ts, data: payload }));
    }
  } catch (e) {
    waitingForKey = true;
  }
};

function mapCoord(e) {
  const r = canvas.getBoundingClientRect();
  const w = canvas.width || 2560;
  const h = canvas.height || 1116;
  const scale = Math.min(r.width / w, r.height / h);
  const realW = w * scale, realH = h * scale;
  const offX = r.left + (r.width - realW) / 2;
  const offY = r.top + (r.height - realH) / 2;
  const x = Math.max(0, Math.min(w, Math.round((e.clientX - offX) / scale)));
  const y = Math.max(0, Math.min(h, Math.round((e.clientY - offY) / scale)));
  return { x, y, w, h };
}

const activePointers = new Map();
const usedSlots = new Set();

function getSlot(id, isDown) {
  if (activePointers.has(id)) return activePointers.get(id).slot;
  if (!isDown) return -1;
  for (let i = 0; i < 10; i++) {
    if (!usedSlots.has(i)) {
      usedSlots.add(i);
      activePointers.set(id, { slot: i, lastX: -1, lastY: -1 });
      return i;
    }
  }
  return -1;
}

const touchBuf = new Uint8Array(22);
const touchView = new DataView(touchBuf.buffer);
touchBuf[0] = 2;

function sendTouch(action, e) {
  if (ws.readyState !== 1) return;
  const slot = getSlot(e.pointerId, action === 0);
  if (slot === -1) return;

  const { x, y, w, h } = mapCoord(e);
  const info = activePointers.get(e.pointerId);
  if (action === 2 && info && x === info.lastX && y === info.lastY) return;
  if (info) { info.lastX = x; info.lastY = y; }

  touchBuf[1] = action;
  touchView.setBigInt64(2, BigInt(slot));
  touchView.setInt32(10, x);
  touchView.setInt32(14, y);
  touchView.setUint16(18, w);
  touchView.setUint16(20, h);
  ws.send(touchBuf);

  if (action === 1) {
    usedSlots.delete(slot);
    activePointers.delete(e.pointerId);
  }
}

window.addEventListener('pointerdown', (e) => {
  try { canvas.setPointerCapture(e.pointerId); } catch (err) {}
  sendTouch(0, e);
}, { passive: true });

// Prioritize pointerrawupdate to bypass browser frame-aligned input coalescing
const moveEvent = ('onpointerrawupdate' in window) ? 'pointerrawupdate' : 'pointermove';
window.addEventListener(moveEvent, (e) => {
  if (activePointers.has(e.pointerId)) sendTouch(2, e);
}, { passive: true });

window.addEventListener('pointerup',     (e) => { sendTouch(1, e); }, { passive: true });
window.addEventListener('pointercancel', (e) => { sendTouch(1, e); }, { passive: true });
</script>
</body>
</html>`;

console.log('[*] Starting 2K 2D-Canvas zero-wait direct streaming service...');
const scrcpy = spawn('/system/bin/app_process', [
  '/system/bin',
  'com.genymobile.scrcpy.Server',
  '3.1',
  'log_level=info',
  'scid=01234567',
  'max_size=2560',
  'video_bit_rate=12000000',
  'max_fps=60',
  'video_codec_options=i-frame-interval=3,low-latency=1,priority=0,max-bframes=0,vendor.qti-ext-enc-low-latency.enable=1',
  'audio=false',
  'control=true',
  'cleanup=false',
  'tunnel_forward=true',
  'send_device_meta=false',
  'send_codec_meta=false',
  'send_dummy_byte=false',
  'send_frame_meta=true'
], {
  env: { ...process.env, CLASSPATH: '/data/local/tmp/scrcpy-server.jar' }
});

let connected = false;
scrcpy.stdout.on('data', (data) => {
  const text = data.toString().trim();
  console.log('[scrcpy]:', text);
  if (!connected && text.includes('Device:')) {
    connected = true;
    setTimeout(initSockets, 300);
  }
});

scrcpy.stderr.on('data', (data) => {
  console.log('[scrcpy ERR]:', data.toString().trim());
});

function initSockets() {
  const videoSock = net.connect('\0scrcpy_01234567', () => {
    videoSock.setNoDelay(true);
    console.log('[+] Video socket pipeline ready');

    ctrlSocket = net.connect('\0scrcpy_01234567', () => {
      ctrlSocket.setNoDelay(true);
      console.log('[+] Native touch control socket ready');
      // If optional auto-screen-off module is enabled, turn off AMOLED backlight
      if (fs.existsSync('/data/data/com.termux/files/home/car-ws/.screen_off')) {
        try {
          execSync('/system/bin/settings put system screen_brightness_mode 0 </dev/null >/dev/null 2>&1; for f in /sys/class/backlight/*/brightness; do chmod 666 "$f" 2>/dev/null; echo 0 > "$f" 2>/dev/null; done');
          console.log('[+] Phone physical AMOLED backlight turned off');
        } catch (e) {}
      }
    });
    ctrlSocket.on('error', (err) => console.error('[!] Control socket error:', err.message));
  });

  videoSock.on('error', (err) => console.error('[!] Video socket error:', err.message));

  let buf = Buffer.alloc(0);
  videoSock.on('data', (chunk) => {
    buf = buf.length === 0 ? chunk : Buffer.concat([buf, chunk]);
    let offset = 0;
    while (buf.length - offset >= 12) {
      const ptsFlags = buf.readBigUInt64BE(offset);
      const isConfig = (ptsFlags & (1n << 63n)) !== 0n;
      const isKey = (ptsFlags & (1n << 62n)) !== 0n;
      const size = buf.readUInt32BE(offset + 8);
      if (buf.length - offset < 12 + size) break;

      const payload = buf.subarray(offset + 12, offset + 12 + size);
      offset += 12 + size;

      const packet = Buffer.allocUnsafe(2 + size);
      packet[0] = isKey ? 1 : 0;
      packet[1] = isConfig ? 1 : 0;
      payload.copy(packet, 2);

      if (isConfig) configPacket = packet;
      if (isKey) lastKeyframe = packet;

      for (const client of clients) {
        if (client.readyState === 1) {
          if (isConfig || isKey) {
            client.dropping = false;
          } else if (client.bufferedAmount > 524288) {
            client.dropping = true;
          }
          if (client.dropping) continue;

          client.send(packet, { binary: true });
        }
      }
    }
    if (offset > 0) {
      buf = offset === buf.length ? Buffer.alloc(0) : buf.subarray(offset);
    }
  });
}

const server = https.createServer({
  key: fs.readFileSync('/data/data/com.termux/files/home/car-ws/key.pem'),
  cert: fs.readFileSync('/data/data/com.termux/files/home/car-ws/cert.pem')
}, (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/html; charset=utf-8',
    'Cache-Control': 'no-store, no-cache, must-revalidate'
  });
  res.end(html);
});

server.on('connection', (socket) => { socket.setNoDelay(true); });

const wss = new WebSocketServer({ server, perMessageDeflate: false });
wss.on('connection', (ws) => {
  ws.isAlive = true;
  ws.dropping = false;
  ws.on('pong', () => { ws.isAlive = true; });

  clients.add(ws);
  if (configPacket) ws.send(configPacket, { binary: true });
  if (lastKeyframe) ws.send(lastKeyframe, { binary: true });

  ws.on('message', (msg, isBinary) => {
    if (isBinary && ctrlSocket && ctrlSocket.writable && msg.length >= 22) {
      const action = msg[1];
      scrcpyTouchMsg[1] = action;
      msg.copy(scrcpyTouchMsg, 2, 2, 22);
      scrcpyTouchMsg.writeUInt16BE(action === 1 ? 0 : 0xffff, 22);
      scrcpyTouchMsg.writeInt32BE(action === 1 ? 0 : 1, 24);
      scrcpyTouchMsg.writeInt32BE(action === 1 ? 0 : 1, 28);
      ctrlSocket.write(scrcpyTouchMsg);
    }
  });

  const cleanup = () => { clients.delete(ws); };
  ws.on('close', cleanup);
  ws.on('error', cleanup);
});

setInterval(() => {
  for (const ws of clients) {
    if (ws.isAlive === false) {
      clients.delete(ws);
      try { ws.terminate(); } catch (e) {}
      continue;
    }
    ws.isAlive = false;
    try { ws.ping(); } catch (e) {}
  }
}, 5000);

function getLocalIp() {
  try {
    const nets = os.networkInterfaces();
    if (nets['wlan0']) {
      for (const netInfo of nets['wlan0']) {
        if (netInfo.family === 'IPv4' && !netInfo.internal) return netInfo.address;
      }
    }
    for (const name of Object.keys(nets)) {
      for (const netInfo of nets[name]) {
        if (netInfo.family === 'IPv4' && !netInfo.internal) return netInfo.address;
      }
    }
  } catch (e) {}
  return '192.168.121.56';
}

server.listen(8443, '0.0.0.0', () => {
  console.log('[+] Mirroring server started: https://' + getLocalIp() + ':8443');
});
EOF

# === 3. Write start.sh (Switch to 2560x1116 landscape, 290 DPI, lock 60Hz, set highest priority) ===
cat << 'EOF' > /data/data/com.termux/files/home/car-ws/start.sh
#!/data/data/com.termux/files/usr/bin/bash
echo "[*] Switching to 2560x1116 landscape mode and starting streaming service..."
su -c '
  export PATH=/system/bin:/system/xbin:/vendor/bin:$PATH
  unset LD_PRELOAD
  unset LD_LIBRARY_PATH

  for f in /sys/class/backlight/*/brightness; do
    chmod 666 "$f" 2>/dev/null
  done

  killall -9 node app_process 2>/dev/null

  # Disable Wi-Fi power saving and reduce TCP unsent buffer for lower latency
  iw dev wlan0 set power_save off 2>/dev/null || true
  sysctl -w net.ipv4.tcp_notsent_lowat=131072 2>/dev/null || true
  /system/bin/dumpsys deviceidle disable </dev/null >/dev/null 2>&1

  # Lock refresh rate to 60Hz to match car display VSync
  /system/bin/settings put system min_refresh_rate 60.0 </dev/null >/dev/null 2>&1
  /system/bin/settings put system peak_refresh_rate 60.0 </dev/null >/dev/null 2>&1

  # Switch resolution to 1116x2560 (landscape 2560x1116) and DPI to 290
  /system/bin/settings put system screen_brightness_mode 0 </dev/null >/dev/null 2>&1
  /system/bin/settings put system accelerometer_rotation 0 </dev/null >/dev/null 2>&1
  /system/bin/wm size 1116x2560 </dev/null >/dev/null 2>&1
  /system/bin/wm density 290 </dev/null >/dev/null 2>&1
  /system/bin/settings put system user_rotation 1 </dev/null >/dev/null 2>&1
  /system/bin/svc power stayon true </dev/null >/dev/null 2>&1

  cd /data/data/com.termux/files/home/car-ws
  nohup /data/data/com.termux/files/usr/bin/node server.js </dev/null >/dev/null 2>&1 &
  sleep 2

  # Boost CPU and I/O priority of streaming processes to maximum
  for pid in $(pgrep -f "node server.js") $(pgrep -f "com.genymobile.scrcpy.Server"); do
    renice -n -20 -p "$pid" 2>/dev/null || true
    ionice -c 1 -n 0 -p "$pid" 2>/dev/null || true
  done

  # Check optional auto-screen-off flag
  if [ -f /data/data/com.termux/files/home/car-ws/.screen_off ]; then
    for f in /sys/class/backlight/*/brightness; do
      echo 0 > "$f" 2>/dev/null
    done
  fi
' </dev/null >/dev/null 2>&1

PHONE_IP=$(su -c "ip -4 addr show wlan0 | awk '/inet /{print \$2}' | cut -d/ -f1" 2>/dev/null)
echo "[+] Streaming active at: https://${PHONE_IP:-192.168.121.56}:8443 (Type 'stop' to restore default portrait mode)"
EOF
chmod +x /data/data/com.termux/files/home/car-ws/start.sh
cp -f /data/data/com.termux/files/home/car-ws/start.sh /data/data/com.termux/files/usr/bin/start
chmod +x /data/data/com.termux/files/usr/bin/start

# === 4. Write stop.sh (Restore brightness, native resolution, default DPI, and portrait mode) ===
cat << 'EOF' > /data/data/com.termux/files/home/car-ws/stop.sh
#!/data/data/com.termux/files/usr/bin/bash
echo "[*] Stopping stream and restoring native phone resolution, DPI, and portrait mode..."
nohup su -c '
  export PATH=/system/bin:/system/xbin:/vendor/bin:$PATH
  unset LD_PRELOAD
  unset LD_LIBRARY_PATH

  # 1. Restore physical screen backlight and wake display first
  for f in /sys/class/backlight/*/brightness; do
    chmod 666 "$f" 2>/dev/null
    echo 800 > "$f" 2>/dev/null
  done
  /system/bin/settings put system screen_brightness_mode 0 </dev/null >/dev/null 2>&1
  /system/bin/settings put system screen_brightness 160 </dev/null >/dev/null 2>&1
  /system/bin/input keyevent 224 </dev/null >/dev/null 2>&1

  # 2. Reset resolution, default DPI, 90Hz refresh rate, and portrait orientation
  /system/bin/wm size reset </dev/null >/dev/null 2>&1
  /system/bin/wm density reset </dev/null >/dev/null 2>&1
  /system/bin/settings put system min_refresh_rate 90.0 </dev/null >/dev/null 2>&1
  /system/bin/settings put system peak_refresh_rate 90.0 </dev/null >/dev/null 2>&1
  /system/bin/settings put system accelerometer_rotation 0 </dev/null >/dev/null 2>&1
  /system/bin/settings put system user_rotation 0 </dev/null >/dev/null 2>&1
  /system/bin/svc power stayon false </dev/null >/dev/null 2>&1
  /system/bin/dumpsys deviceidle enable </dev/null >/dev/null 2>&1

  # 3. Terminate streaming processes
  killall -9 node app_process 2>/dev/null
' </dev/null >/dev/null 2>&1 &
echo "[+] Restored to default resolution, DPI, and portrait mode! (Type 'start' to resume car streaming)"
EOF
chmod +x /data/data/com.termux/files/home/car-ws/stop.sh
cp -f /data/data/com.termux/files/home/car-ws/stop.sh /data/data/com.termux/files/usr/bin/stop
chmod +x /data/data/com.termux/files/usr/bin/stop

# === 5. Register global aliases in ~/.bashrc ===
cat << 'EOF' > ~/.bashrc
alias start='/data/data/com.termux/files/home/car-ws/start.sh'
alias stop='/data/data/com.termux/files/home/car-ws/stop.sh'
EOF
source ~/.bashrc
echo "[+] Core scripts deployed successfully!"
```

### 【第 3 步结果验证（必测）】
1. 在 Termux 中输入 **`start`** 并回车，随后运行以下状态检查命令：
   ```bash
   su -c 'ps -ef | grep -E "node|scrcpy" | grep -v grep && netstat -tulnp | grep 8443'
   ```
   > **验证标准**：
   > - 手机画面应立即切换为 **2560×1116 横屏模式**，DPI 同步缩放。
   > - 终端必须同时看到 `node server.js` 进程、`com.genymobile.scrcpy.Server` 进程，以及 `0.0.0.0:8443 LISTEN` 监听状态。
2. 紧接着在 Termux 中输入 **`stop`** 并回车：
   > **验证标准**：手机画面应立刻切回原本的 **竖屏模式**，且分辨率与图标大小（DPI）100% 恢复出厂默认状态。验证通过后，再次输入 **`start`** 开启服务，进入车机端连接步骤！

---

##  六、 萤火虫车机端连接步骤与验证

1. **手机连接车机 5GHz 热点**
   - 确认备用手机已连接上萤火虫车机的 5GHz Wi-Fi 热点，并在 Termux 中运行了 `start`，记下终端显示的访问地址（例如 `[https://192.168.121.56:8443](https://192.168.121.56:8443)`）。
2. **车机浏览器首次访问与信任证书**
   - 打开萤火虫车机自带浏览器，在地址栏输入完整网址 **`[https://192.168.121.56:8443](https://192.168.121.56:8443)`**（**注意**：开头必须是 `https://`，不能少 `s`，端口号前是英文半角冒号 `:`）。
   - 首次访问会提示“您的连接不是私密连接”（因为使用了本地自签名证书），依次点击左下角的 **“高级”** -> **“继续前往 192.168.x.x（不安全）”**。
3. **添加为车机桌面独立应用**
   - 此时手机画面已实时显示在车机上。点击车机浏览器右上角的 **`+`（添加应用）** 按钮，保存为桌面快捷方式（默认名称为“虚拟车机”）。
   - 回到车机桌面或应用列表，点开保存的图标，点击全屏即可享受 2560×1116 点对点满屏 60FPS 操控。

---

##  七、 可选进阶模块（按需复制输入 Termux 即可生效）

当你在车机上验证画面与多指触控一切正常后，可根据需要复制以下独立模块到 Termux 中开启进阶功能。

### 可选模块 A：开启“推流时自动彻底熄灭手机 AMOLED 屏幕”（零发光、降发热）
- **作用**：每次运行 `start` 启动推流后，手机会在 2 秒后自动将 AMOLED 物理背光降为 `0`（手机屏幕彻底黑屏不发光、不发热，同时车机推流画面与触控 100% 正常保持）；输入 `stop` 或按一下手机实体电源键即可随时重新点亮屏幕。
- **启用命令**：

```bash
# Enable auto AMOLED screen-off flag and restart streaming
touch /data/data/com.termux/files/home/car-ws/.screen_off
start
echo "[+] Auto screen-off enabled! (To disable anytime, run: rm -f ~/car-ws/.screen_off)"
```
> ** 结果验证**：运行后等待 2 秒，观察手机屏幕是否彻底熄灭变黑，同时在车机浏览器上滑动画面依然丝滑响应；再在终端输入 `stop`，观察手机屏幕是否立即点亮并切回默认竖屏。

---

### 可选模块 B：开启“开机与启动 Termux 自动运行推流”（免装额外 APK）
- **作用**：配置 `.bashrc` 自检逻辑与 Magisk/KernelSU `service.d` 开机自启脚本。手机重启开机后（或首次点开 Termux 时），会自动检测并在后台拉起 `start.sh` 进入车载推流模式，**无需安装 `Termux:Boot` 等任何额外 APK**。
- **启用命令**：

```bash
# 1. Configure ~/.bashrc auto-start check when Termux session initializes
cat << 'EOF' > ~/.bashrc
alias start='/data/data/com.termux/files/home/car-ws/start.sh'
alias stop='/data/data/com.termux/files/home/car-ws/stop.sh'

# Auto-start once per boot if server.js is not running
if ! pgrep -f "node server.js" >/dev/null 2>&1 && [ ! -f /dev/.car_ws_started ]; then
  su -c 'touch /dev/.car_ws_started' 2>/dev/null
  /data/data/com.termux/files/home/car-ws/start.sh
fi
EOF

# 2. Configure Magisk/KernelSU service.d script for background boot auto-start
su -c '
  mkdir -p /data/adb/service.d
  cat << "INNER_EOF" > /data/adb/service.d/car_ws_autostart.sh
#!/system/bin/sh
until [ "$(getprop sys.boot_completed)" = "1" ]; do
  sleep 2
done
sleep 5
touch /dev/.car_ws_started
su - $(stat -c "%U" /data/data/com.termux/files/home) -c "/data/data/com.termux/files/home/car-ws/start.sh"
INNER_EOF
  chmod +x /data/adb/service.d/car_ws_autostart.sh
'
echo "=== Module B Verification ==="
su -c 'ls -l /data/adb/service.d/car_ws_autostart.sh'
```
> ** 结果验证**：终端输出 `-rwxr-xr-x ... /data/adb/service.d/car_ws_autostart.sh` 即代表挂载成功。

---

##  八、 备选方案：60帧 WebGL 垂直同步零撕裂版（Mailbox 模式）

第五节默认部署的是 **2D Canvas 极速直通版**（追求 20~30ms 物理极限低延迟，绕过垂直同步，但在快速左右滑动桌面时可能会有水平扫描线撕裂感）。

如果你对**画面丝滑度、100% 零撕裂**有强迫症级要求，不在乎多出约 10~15ms 的垂直同步等待时间（总延迟约 **40ms ~ 50ms**），可以直接刷入下面这个 **WebGL + Mailbox（单槽位信箱模式）零撕裂稳定版**：
- **实现原理**：硬件解码器解码出的帧放入单槽位变量 `latestFrame`（若有旧帧未画完立即销毁换新帧，绝不排队积压），由 `requestAnimationFrame` 严格踩在车机屏幕每次 60Hz 垂直同步（VSync）的瞬间通过 WebGL 渲染上屏，并保持 `desynchronized: false` 彻底杜绝安卓硬件合成器坐标系翻转闪烁 Bug。
- **一键切换方法**：直接复制下方代码块到 Termux 运行即可无缝替换 `server.js`（你之前设置的 `start` / `stop` 以及可选模块 A 的自动熄屏状态会自动保留，完全不受影响）：

```bash
# === Replace server.js with 60FPS WebGL Mailbox Tear-Free Stable Edition ===
cat << 'EOF' > /data/data/com.termux/files/home/car-ws/server.js
const https = require('https');
const fs = require('fs');
const net = require('net');
const os = require('os');
const { spawn, execSync } = require('child_process');
const { WebSocketServer } = require('ws');

try { execSync('killall -9 app_process 2>/dev/null || true'); } catch (e) {}

let configPacket = null;
let lastKeyframe = null;
let ctrlSocket = null;
const clients = new Set();

const scrcpyTouchMsg = Buffer.alloc(32);
scrcpyTouchMsg[0] = 2;
scrcpyTouchMsg.writeUInt16BE(0xffff, 22);
scrcpyTouchMsg.writeInt32BE(1, 24);
scrcpyTouchMsg.writeInt32BE(1, 28);

const html = `<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no,viewport-fit=cover">
<meta name="mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-capable" content="yes">
<title>虚拟车机</title>
<style>
  * { margin:0; padding:0; box-sizing:border-box; user-select:none; -webkit-user-select:none; }
  html, body { width:100%; height:100%; background:#000; overflow:hidden; touch-action:none; display:flex; align-items:center; justify-content:center; }
  canvas { max-width:100vw; max-height:100vh; width:100%; height:100%; object-fit:contain; display:block; }
</style>
</head>
<body>
<canvas id="v"></canvas>
<script>
const canvas = document.getElementById('v');

canvas.addEventListener('webglcontextlost', (e) => {
  e.preventDefault();
  setTimeout(() => location.reload(), 500);
}, false);

// Keep desynchronized: false in WebGL to prevent Android HWC coordinate flip flickering and screen tearing
const gl = canvas.getContext('webgl', {
  alpha: false,
  depth: false,
  stencil: false,
  antialias: false,
  preserveDrawingBuffer: false,
  desynchronized: false,
  powerPreference: 'high-performance'
});

const vsSource = 'attribute vec2 p; varying vec2 uv; void main(){ uv=vec2(p.x*0.5+0.5, 0.5-p.y*0.5); gl_Position=vec4(p,0.0,1.0); }';
const fsSource = 'precision mediump float; varying vec2 uv; uniform sampler2D t; void main(){ gl_FragColor=texture2D(t,uv); }';

function compileShader(type, src) {
  const s = gl.createShader(type);
  gl.shaderSource(s, src);
  gl.compileShader(s);
  return s;
}
const prog = gl.createProgram();
gl.attachShader(prog, compileShader(gl.VERTEX_SHADER, vsSource));
gl.attachShader(prog, compileShader(gl.FRAGMENT_SHADER, fsSource));
gl.linkProgram(prog);
gl.useProgram(prog);

const vbo = gl.createBuffer();
gl.bindBuffer(gl.ARRAY_BUFFER, vbo);
gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([-1,-1, 1,-1, -1,1, 1,1]), gl.STATIC_DRAW);
const pLoc = gl.getAttribLocation(prog, 'p');
gl.enableVertexAttribArray(pLoc);
gl.vertexAttribPointer(pLoc, 2, gl.FLOAT, false, 0, 0);

const tex = gl.createTexture();
gl.bindTexture(gl.TEXTURE_2D, tex);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);

let decoder = null;
let configured = false;
let waitingForKey = true;
let configData = null;
let ts = 0;
let latestFrame = null; // Single-slot mailbox: always holds only the newest decoded frame

function hex2(b) { return ('0' + b.toString(16)).slice(-2).toUpperCase(); }

// VSync-aligned render loop for 100% tear-free output
function renderLoop() {
  requestAnimationFrame(renderLoop);
  if (!latestFrame) return;
  const frame = latestFrame;
  latestFrame = null;

  if (canvas.width !== frame.displayWidth || canvas.height !== frame.displayHeight) {
    canvas.width = frame.displayWidth;
    canvas.height = frame.displayHeight;
    gl.viewport(0, 0, canvas.width, canvas.height);
  }
  gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, frame);
  gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
  gl.flush();
  frame.close();
}
requestAnimationFrame(renderLoop);

function initDecoder() {
  configured = false;
  waitingForKey = true;
  decoder = new VideoDecoder({
    output(frame) {
      if (latestFrame) {
        latestFrame.close();
      }
      latestFrame = frame;
    },
    error() { waitingForKey = true; }
  });
}

initDecoder();

document.addEventListener('visibilitychange', () => {
  if (!document.hidden) waitingForKey = true;
});

const proto = location.protocol === 'https:' ? 'wss:' : 'ws:';
const ws = new WebSocket(proto + '//' + location.host + '/stream');
ws.binaryType = 'arraybuffer';

ws.onclose = () => { setTimeout(() => location.reload(), 1000); };

ws.onmessage = (ev) => {
  const buf = new Uint8Array(ev.data);
  if (buf.length < 2) return;
  const isKey = buf[0] === 1;
  const isConfig = buf[1] === 1;
  const payload = buf.subarray(2);

  if (isConfig) {
    configData = payload;
    let offset = 0;
    while (offset < payload.length - 4) {
      if (payload[offset] === 0 && payload[offset+1] === 0 && (payload[offset+2] === 1 || (payload[offset+2] === 0 && payload[offset+3] === 1))) {
        const nalOffset = (payload[offset+2] === 1) ? offset + 3 : offset + 4;
        const type = payload[nalOffset] & 0x1f;
        if (type === 7) {
          const codecStr = 'avc1.' + hex2(payload[nalOffset+1]) + hex2(payload[nalOffset+2]) + hex2(payload[nalOffset+3]);
          if (!configured) {
            try {
              decoder.configure({ codec: codecStr, optimizeForLatency: true, hardwareAcceleration: 'prefer-hardware' });
              configured = true;
            } catch (e) {}
          }
          break;
        }
      }
      offset++;
    }
    return;
  }

  if (!configured) return;

  if (decoder.decodeQueueSize > 6) {
    waitingForKey = true;
  }

  if (waitingForKey) {
    if (!isKey) return;
    waitingForKey = false;
  }

  try {
    ts += 16666;
    if (isKey && configData) {
      const merged = new Uint8Array(configData.length + payload.length);
      merged.set(configData, 0);
      merged.set(payload, configData.length);
      decoder.decode(new EncodedVideoChunk({ type: 'key', timestamp: ts, data: merged }));
    } else {
      decoder.decode(new EncodedVideoChunk({ type: isKey ? 'key' : 'delta', timestamp: ts, data: payload }));
    }
  } catch (e) {
    waitingForKey = true;
  }
};

function mapCoord(e) {
  const r = canvas.getBoundingClientRect();
  const w = canvas.width || 2560;
  const h = canvas.height || 1116;
  const scale = Math.min(r.width / w, r.height / h);
  const realW = w * scale, realH = h * scale;
  const offX = r.left + (r.width - realW) / 2;
  const offY = r.top + (r.height - realH) / 2;
  const x = Math.max(0, Math.min(w, Math.round((e.clientX - offX) / scale)));
  const y = Math.max(0, Math.min(h, Math.round((e.clientY - offY) / scale)));
  return { x, y, w, h };
}

const activePointers = new Map();
const usedSlots = new Set();

function getSlot(id, isDown) {
  if (activePointers.has(id)) return activePointers.get(id).slot;
  if (!isDown) return -1;
  for (let i = 0; i < 10; i++) {
    if (!usedSlots.has(i)) {
      usedSlots.add(i);
      activePointers.set(id, { slot: i, lastX: -1, lastY: -1 });
      return i;
    }
  }
  return -1;
}

const touchBuf = new Uint8Array(22);
const touchView = new DataView(touchBuf.buffer);
touchBuf[0] = 2;

function sendTouch(action, e) {
  if (ws.readyState !== 1) return;
  const slot = getSlot(e.pointerId, action === 0);
  if (slot === -1) return;

  const { x, y, w, h } = mapCoord(e);
  const info = activePointers.get(e.pointerId);
  if (action === 2 && info && x === info.lastX && y === info.lastY) return;
  if (info) { info.lastX = x; info.lastY = y; }

  touchBuf[1] = action;
  touchView.setBigInt64(2, BigInt(slot));
  touchView.setInt32(10, x);
  touchView.setInt32(14, y);
  touchView.setUint16(18, w);
  touchView.setUint16(20, h);
  ws.send(touchBuf);

  if (action === 1) {
    usedSlots.delete(slot);
    activePointers.delete(e.pointerId);
  }
}

window.addEventListener('pointerdown', (e) => {
  try { canvas.setPointerCapture(e.pointerId); } catch (err) {}
  sendTouch(0, e);
});
window.addEventListener('pointermove', (e) => {
  if (activePointers.has(e.pointerId)) sendTouch(2, e);
});
window.addEventListener('pointerup',   (e) => { sendTouch(1, e); });
window.addEventListener('pointercancel',(e) => { sendTouch(1, e); });
</script>
</body>
</html>`;

console.log('[*] Starting 2K 60FPS WebGL Tear-Free Mailbox service...');
const scrcpy = spawn('/system/bin/app_process', [
  '/system/bin',
  'com.genymobile.scrcpy.Server',
  '3.1',
  'log_level=info',
  'scid=01234567',
  'max_size=2560',
  'video_bit_rate=12000000',
  'max_fps=60',
  'video_codec_options=i-frame-interval=2,low-latency=1,priority=0,max-bframes=0,vendor.qti-ext-enc-low-latency.enable=1',
  'audio=false',
  'control=true',
  'cleanup=false',
  'tunnel_forward=true',
  'send_device_meta=false',
  'send_codec_meta=false',
  'send_dummy_byte=false',
  'send_frame_meta=true'
], {
  env: { ...process.env, CLASSPATH: '/data/local/tmp/scrcpy-server.jar' }
});

let connected = false;
scrcpy.stdout.on('data', (data) => {
  const text = data.toString().trim();
  console.log('[scrcpy]:', text);
  if (!connected && text.includes('Device:')) {
    connected = true;
    setTimeout(initSockets, 300);
  }
});

scrcpy.stderr.on('data', (data) => {
  console.log('[scrcpy ERR]:', data.toString().trim());
});

function initSockets() {
  const videoSock = net.connect('\0scrcpy_01234567', () => {
    videoSock.setNoDelay(true);
    console.log('[+] Video socket pipeline ready');

    ctrlSocket = net.connect('\0scrcpy_01234567', () => {
      ctrlSocket.setNoDelay(true);
      console.log('[+] Native touch control socket ready');
      // If optional auto-screen-off module is enabled, turn off AMOLED backlight
      if (fs.existsSync('/data/data/com.termux/files/home/car-ws/.screen_off')) {
        try {
          execSync('/system/bin/settings put system screen_brightness_mode 0 </dev/null >/dev/null 2>&1; for f in /sys/class/backlight/*/brightness; do chmod 666 "$f" 2>/dev/null; echo 0 > "$f" 2>/dev/null; done');
          console.log('[+] Phone physical AMOLED backlight turned off');
        } catch (e) {}
      }
    });
    ctrlSocket.on('error', (err) => console.error('[!] Control socket error:', err.message));
  });

  videoSock.on('error', (err) => console.error('[!] Video socket error:', err.message));

  let buf = Buffer.alloc(0);
  videoSock.on('data', (chunk) => {
    buf = buf.length === 0 ? chunk : Buffer.concat([buf, chunk]);
    let offset = 0;
    while (buf.length - offset >= 12) {
      const ptsFlags = buf.readBigUInt64BE(offset);
      const isConfig = (ptsFlags & (1n << 63n)) !== 0n;
      const isKey = (ptsFlags & (1n << 62n)) !== 0n;
      const size = buf.readUInt32BE(offset + 8);
      if (buf.length - offset < 12 + size) break;

      const payload = buf.subarray(offset + 12, offset + 12 + size);
      offset += 12 + size;

      const packet = Buffer.allocUnsafe(2 + size);
      packet[0] = isKey ? 1 : 0;
      packet[1] = isConfig ? 1 : 0;
      payload.copy(packet, 2);

      if (isConfig) configPacket = packet;
      if (isKey) lastKeyframe = packet;

      for (const client of clients) {
        if (client.readyState === 1) {
          if (isConfig || isKey) {
            client.dropping = false;
          } else if (client.bufferedAmount > 1048576) {
            client.dropping = true;
          }
          if (client.dropping) continue;

          client.send(packet, { binary: true });
        }
      }
    }
    if (offset > 0) {
      buf = offset === buf.length ? Buffer.alloc(0) : buf.subarray(offset);
    }
  });
}

const server = https.createServer({
  key: fs.readFileSync('/data/data/com.termux/files/home/car-ws/key.pem'),
  cert: fs.readFileSync('/data/data/com.termux/files/home/car-ws/cert.pem')
}, (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/html; charset=utf-8',
    'Cache-Control': 'no-store, no-cache, must-revalidate'
  });
  res.end(html);
});

server.on('connection', (socket) => { socket.setNoDelay(true); });

const wss = new WebSocketServer({ server, perMessageDeflate: false });
wss.on('connection', (ws) => {
  ws.isAlive = true;
  ws.dropping = false;
  ws.on('pong', () => { ws.isAlive = true; });

  clients.add(ws);
  if (configPacket) ws.send(configPacket, { binary: true });
  if (lastKeyframe) ws.send(lastKeyframe, { binary: true });

  ws.on('message', (msg, isBinary) => {
    if (isBinary && ctrlSocket && ctrlSocket.writable && msg.length >= 22) {
      const action = msg[1];
      scrcpyTouchMsg[1] = action;
      msg.copy(scrcpyTouchMsg, 2, 2, 22);
      scrcpyTouchMsg.writeUInt16BE(action === 1 ? 0 : 0xffff, 22);
      scrcpyTouchMsg.writeInt32BE(action === 1 ? 0 : 1, 24);
      scrcpyTouchMsg.writeInt32BE(action === 1 ? 0 : 1, 28);
      ctrlSocket.write(scrcpyTouchMsg);
    }
  }
);

  const cleanup = () => { clients.delete(ws); };
  ws.on('close', cleanup);
  ws.on('error', cleanup);
});

setInterval(() => {
  for (const ws of clients) {
    if (ws.isAlive === false) {
      clients.delete(ws);
      try { ws.terminate(); } catch (e) {}
      continue;
    }
    ws.isAlive = false;
    try { ws.ping(); } catch (e) {}
  }
}, 5000);

function getLocalIp() {
  try {
    const nets = os.networkInterfaces();
    if (nets['wlan0']) {
      for (const netInfo of nets['wlan0']) {
        if (netInfo.family === 'IPv4' && !netInfo.internal) return netInfo.address;
      }
    }
    for (const name of Object.keys(nets)) {
      for (const netInfo of nets[name]) {
        if (netInfo.family === 'IPv4' && !netInfo.internal) return netInfo.address;
      }
    }
  } catch (e) {}
  return '192.168.121.56';
}

server.listen(8443, '0.0.0.0', () => {
  console.log('[+] Mirroring server started: https://' + getLocalIp() + ':8443');
});
EOF
start
echo "[+] Switched to 60FPS WebGL Tear-Free Edition and restarted service!"
```

---

##  九、小白专属：AI 调试助手提示词（复制给任意 AI 帮你排错与定制）

如果你在部署过程中遇到任何报错，或者想让 AI 帮你在 **2D Canvas 极速版** 与 **WebGL 无撕裂版** 之间切换、调整分辨率/DPI 或适配其他品牌手机，只需把 **下方代码块内的英文提示词** 连同 **整篇教程（及你的终端报错信息）** 一起复制发送给任意 AI（ChatGPT / Claude / Gemini / DeepSeek 等），AI 就会直接用中文手把手带你排错：

```text
### SYSTEM PROMPT FOR AI TROUBLESHOOTING ASSISTANT ###

You are an elite Android Systems Engineer, Linux Kernel/Root Specialist, and WebCodecs Low-Latency Streaming Architect.

The user is a NIO Firefly (蔚来萤火虫) car owner replicating the "ZeroCast" custom ultra-low-latency wireless screen mirroring system from a rooted Android phone to the vehicle's built-in Chromium browser, based on the attached tutorial.

#### Technical Architecture & Dual Rendering Editions:
1. **Host Environment**: Rooted Android phone (typically Qualcomm Snapdragon with AMOLED display, Android 11-16) running ONLY the `Termux` app (no `Termux:Boot` APK required).
2. **Capture & Encode**: `/system/bin/app_process` launches `/data/local/tmp/scrcpy-server.jar` (v3.1) with raw H.264 stream (`send_device_meta=false`, `send_codec_meta=false`, `send_dummy_byte=false`, `send_frame_meta=true`) and Qualcomm low-latency flags (`low-latency=1,priority=0,max-bframes=0,vendor.qti-ext-enc-low-latency.enable=1`).
3. **Two Frontend Rendering Editions in `server.js`**:
   - **Edition 1 (Section 5 - 2D Canvas Direct Edition, 20-30ms)**: Uses `canvas.getContext('2d', { alpha: false, desynchronized: true })` and calls `ctx.drawImage(frame, 0, 0)` immediately inside `VideoDecoder.output(frame)` without `requestAnimationFrame`, plus `pointerrawupdate` for raw touch polling. Lowest possible latency, safe from WebGL vertical flip bugs, though minor tearing can occur.
   - **Edition 2 (Section 8 - WebGL Mailbox Tear-Free Edition, 35-50ms)**: Uses `canvas.getContext('webgl', { desynchronized: false, ... })` with a single-slot mailbox (`latestFrame`) and `requestAnimationFrame(renderLoop)` + `gl.flush()`. Note: `desynchronized` MUST be `false` in WebGL on Android Chromium to prevent upside-down coordinate flipping/flickering.
4. **System Control (`start.sh` / `stop.sh`) & Screen-Off Flag**:
   - Uses `su -c` with `unset LD_PRELOAD` and I/O redirection `</dev/null >/dev/null 2>&1` (mandatory to bypass Android 14-16 SELinux Binder transaction error `2147483646` when calling `/system/bin/wm` and `/system/bin/settings`).
   - `start` sets `wm size 1116x2560`, `wm density 290`, `user_rotation 1`, locks 60Hz, and sets `renice -n -20`.
   - `stop` wakes the screen first, runs `wm size reset`, `wm density reset`, `user_rotation 0`, restores 90Hz, and kills `node`/`app_process`.
   - Auto AMOLED screen-off is controlled by a flag file `/data/data/com.termux/files/home/car-ws/.screen_off` (`touch` to enable, `rm -f` to disable), so switching `server.js` between Edition 1 and Edition 2 never loses the screen-off configuration.

#### Your Task & Rules:
1. **Language Requirement**: You MUST communicate with the user strictly in **Simplified Chinese (简体中文)**, using a clear, patient, and step-by-step beginner-friendly tone.
2. **Code Comments Requirement**: Keep all comments inside any bash/JS code blocks in **English**.
3. **Key Diagnostic & Customization Scenarios**:
   - **Switching Editions**: Help the user seamlessly switch between the Section 5 (2D Canvas) and Section 8 (WebGL Tear-Free) editions with a single copy-paste command.
   - **Network/Download Failures in China**: If `/data/local/tmp/scrcpy-server.jar` is 0 bytes or missing, guide them to check file size (`ls -lh /data/local/tmp/scrcpy-server.jar`, must be ~71KB) and provide working download commands.
   - **Binder Error `2147483646`**: Ensure all `/system/bin/wm` and `/system/bin/settings` calls run inside `su -c` with `unset LD_PRELOAD` and `</dev/null >/dev/null 2>&1`.
   - **Non-Snapdragon Phones**: If `scrcpy` fails on MediaTek/Kirin/Exynos, instruct them to remove `vendor.qti-ext-enc-low-latency.enable=1` from `video_codec_options`.
   - **Backlight Stuck Off**: Provide the emergency wake-up command: `su -c 'for f in /sys/class/backlight/*/brightness; do chmod 666 "$f"; echo 800 > "$f"; done; /system/bin/input keyevent 224'`.
4. Always provide ready-to-copy-and-paste one-liner Termux commands and explain how to verify the output.

---
Below is the complete tutorial the user is following, followed by the user's specific question or terminal log:
```
