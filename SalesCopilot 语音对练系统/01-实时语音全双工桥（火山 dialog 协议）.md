---
source: /Users/xgw/workspace/Python/SalesCopilot-Coach @5a119d7
type: repo
distilled: 2026-07-27
summary: 浏览器 ↔ 后端 ↔ 火山引擎 dialog 的全双工实时语音桥：自实现的私有二进制帧协议、VolcSession 握手与事件映射、双协程对拷、采样率转换与双声道分轨录音、websockets 双版本兼容。
tags: [distill, realtime-voice, websocket, binary-protocol, audio, volcengine, asyncio]
---

# 实时语音全双工桥（火山 dialog 协议）

> 子笔记 · 回链 [[00-总览]]。姊妹篇：[[02-剧本生成与家长角色建模]] · [[03-图谱锚定评价与规则锚融合]] · [[04-方法论·可迁移模式]]
> 以代码 `@5a119d7` 为准。

## 全链路数据流

一次对练的音频往返穿过三层四跳：

```
浏览器麦克风                          火山 dialog WS
 AudioWorklet 48kHz Float32            (端到端语音大模型)
      │ 二进制帧                            ▲   │
      ▼                                     │   ▼ 24kHz Float32 TTS
 后端 ws_practice ──float32_to_pcm16──► resample 48k→16k ──send_audio──►│
      ▲                                                                 │
      └──────── ws.send_bytes(24k PCM) ◄── float32→pcm16 ◄── BOT_AUDIO ─┘
```

- **上行**（`pump_browser_to_volc`，`ws_practice.py:127-154`）：浏览器 AudioWorklet 给出 48kHz Float32 PCM，二进制帧到后端 → `float32_to_pcm16`（Float32 LE → PCM16 LE）→ `resample_pcm16(48000, 16000)` → 同时**边到边写 `sales.16k.pcm`** → `session.send_audio()` 送火山。
- **下行**（`pump_volc_to_browser`，`ws_practice.py:156-201`）：火山 `BOT_AUDIO` 事件带 24kHz **Float32** PCM → 直接 `ws.send_bytes` 回浏览器播放 + **边到边写 `parent.24k.pcm`**。
- 两个协程由 `asyncio.create_task` 并发，靠一个 `stop_event` 协调收尾。

**关键坑（代码注释点明）**：火山下行是 **Float32 LE**，合成 WAV 前必须先 `float32_to_pcm16` 转 int16（`ws_practice.py:233-234`）；下行回浏览器时不转（浏览器端按 Float32 播）。

## 采样率转换与双声道分轨（`src/voice/audio.py`）

- `resample_pcm16`：整数倍降采样走 `src[::ratio]` **stride 快路径**，非整数倍走 `np.linspace` + `np.interp` 线性插值兜底，最后 `np.clip` 防溢出。48k→16k 是整数倍(3)，走快路径。
- `merge_stereo`：把两路可能不同采样率的单声道（左=老师 16k、右=家长 24k）都重采样到目标 24k，短的补零对齐，交织成立体声 → `audio.wav`（`ws_practice.py:229-246`）。**分轨录音**是为了复盘时能分辨谁说了什么。
- 写完 WAV 后 `unlink` 两个中间 `.pcm` 文件。

**为什么边到边写盘**：`ws_practice.py:107` 注释——"避免长会话把整段音频堆在内存里"。上下行 PCM 直接 `open(...,'wb').write()`，不在内存累积 bytearray。

## 火山私有二进制协议（`src/voice/volc_dialog.py`）

火山 `volc.speech.dialog` 用**自定义二进制帧**（不是 JSON over WS）。本文件从零实现编解码，`tests/test_volc_frame.py` 专测这层。

**帧结构**（4 字节 header v1）：
| 字节 | 位 | 含义 |
|---|---|---|
| byte0 | `[version:4=1][header_size:4=1]` | header_size 单位是 4 字节 |
| byte1 | `[msg_type:4][flag:4]` | 消息类型 + 标志位 |
| byte2 | `[serialization:4][compression:4]` | JSON/RAW + 压缩(无) |
| byte3 | reserved | 0 |

header 之后按 flag/msg_type 依次是：可选 `event_id`(int32 BE，flag 含 `WITH_EVENT=0b0100` 时) → 可选 `session_id`(length-prefixed) 或 `connect_id` → `payload`(length-prefixed)。

**关键路由规则**（`volc_dialog.py:80-85` 的三行注释是解码正确性的命门）：
- `session_id` 在**几乎所有**事件中出现；
- **例外1**：连接级服务端事件 `{50,51,52}` 带的是 `connect_id` 而非 session_id；
- **例外2**：客户端连接事件 `{1,2}` 两者都无。

解码器 `decode()` 据此在 `_CONNECT_ID_EVENTS` / `_NO_SESSION_EVENTS` 间分支取字段。

**MsgType**（高 4 位）：`FULL_CLIENT=1`(JSON上行) / `AUDIO_ONLY_CLIENT=2`(上行音频) / `FULL_SERVER=9`(服务端JSON) / `AUDIO_ONLY_SERVER=11`(TTS音频) / `ERROR_INFO=15`。

**编码器**：`encode_json`（命令/配置）、`encode_audio`（PCM，msg_type=2, event=TASK_REQUEST=200, serialization=RAW）。

## 会话握手与事件映射（`src/voice/volc_session.py`）

一个 `VolcSession` = 一条对外 WS + 一个会话（`session_id`/`connect_id` 各自 uuid4）。

**连接握手序列**（`connect()`）：
1. `websockets.connect` 带鉴权 header（`X-Api-App-ID/App-Key/Access-Key/Resource-Id/Connect-Id`）；
2. 发 `START_CONNECTION(1)` → 等 `CONNECTION_STARTED(50)`；
3. 发 `START_SESSION(100)` 带配置 payload → 等 `SESSION_STARTED(150)`；
4. 若有 `hello_text`，发 `SAY_HELLO(300)` 让**家长先开口**。

`_expect_event` 是个带超时的阻塞读循环：读到目标 event 返回，中途遇 `SESSION_FAILED/CONNECTION_FAILED` 或 `ERROR_INFO` 直接 raise，其他事件忽略。

**START_SESSION 配置 payload**（`_build_start_session_payload`）注入了对练的全部灵魂：
- `asr.extra.end_smooth_window_ms=1500`（断句平滑窗）；
- `tts.speaker` + 24kHz PCM 输出；
- `dialog.system_role`（[[02-剧本生成与家长角色建模]] 拼出的家长设定）、`bot_name`、`speaking_style`、`model`(火山版本)、`recv_timeout`，还有 `strict_audit=False` + 兜底审核话术。

**事件映射** `_map_event`：把火山原始 event id 归一成上层 `ServerEvent` 常量——
- `ASR_RESPONSE(451)` → 按 `is_interim` 分 `ASR_PARTIAL`/`ASR_FINAL`；
- `CHAT_RESPONSE(550)` → `BOT_TEXT`（家长说的话文本）；
- `AUDIO_ONLY_SERVER` msg_type → `BOT_AUDIO`（bytes）；
- TTS 分句/结束等事件**静默忽略**（返回 None）。

ASR/Chat 文本抽取用 `_extract_asr_text` / `_extract_asr_is_final` / `_extract_chat_text` 做**多层结构兜底**（`results[0].alternatives[0].text` → `results[0].text` → `body.text`），因为火山返回结构不完全稳定。

**收尾** `finish()`：发 `FINISH_SESSION(102)` → sleep 0.1 → `FINISH_CONNECTION(2)`，不等 ack，最多给主循环 2 秒，然后强制 close。

## websockets 双版本兼容（一个反复踩的坑）

`connect()` 里（`volc_session.py:82-92`）先试 `additional_headers=`（websockets 14+ 新 asyncio 实现），捕获 `TypeError` 且错误信息含 `additional_headers` 时回退 `extra_headers=`（13.x legacy）。

git log 记录了这个来回：`5b7e969` 为修部署机硬切 `extra_headers`，反把本地 16.0 弄挂；`d57c225` 才改成"先新后旧带 TypeError 兜底"的双版本兼容。**教训**：跨环境依赖版本差异要用能力探测（try/except 参数名），不能二选一硬编码。

## 上层编排（`src/api/ws_practice.py`）

`/ws/practice/{session_id}` 的完整生命周期：
1. `accept` → 读 meta/script（不存在则报错关闭）；
2. `build_system_role` + `build_hello_text` → 构造 `VolcSession` → 发 `stage: preparing`；
3. `session.connect()`（失败发 error 关闭）→ 发 `stage: practicing`；
4. 起双协程对拷，`await stop_event.wait()`（用户 stop / 断连 / 火山 finished 都会 set）；
5. finally：`session.finish()` + cancel 协程 + 关文件；
6. 发 `stage: evaluating` → 合成 `audio.wav` → 写 transcript / 更新 meta（finished_at, duration）→ `asyncio.to_thread` 跑 [[03-图谱锚定评价与规则锚融合|评价]] → 发 `result` → 发 `stage: done` → close。

逐字稿在对拷过程中实时累积：`ASR_FINAL` 追加为 `role=sales` 的 Turn，`BOT_TEXT` 追加为 `role=parent`，都带相对起始的秒数时间戳（`Turn.ts`）。

## 待确认与边界

- 前端 AudioWorklet 的具体实现（`web/js/audio.js` 61 行、`playback.js`）未逐行读，48kHz/Float32 的约定来自后端注释与 README。
- 火山 Event 枚举只列了代码处理的子集，`volc_dialog.py:68` 注释"id 见文档"。
- `recv_timeout`/`ping_interval` 等超时参数的调优依据未在代码中说明。
