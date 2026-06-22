# 小智语音通信端到端执行流程图

本文用流程图展示一次典型语音对话从“用户对开发板讲话”到“开发板播报最终回复”的端到端执行过程，重点说明语音数据与控制信令在开发板、`xiaozhi-server` 以及 ASR、LLM、TTS 等第三方服务之间的流转顺序。

> 说明：下图以 WebSocket 直连为主线；若使用 MQTT 网关，设备侧音频与控制消息会先经 MQTT/UDP 网关转发，再进入相同的服务端处理链路。

```mermaid
sequenceDiagram
    autonumber
    actor User as 用户
    participant Dev as ESP32 开发板
    participant Server as xiaozhi-server<br/>核心语音服务
    participant VAD as VAD<br/>语音活动检测
    participant ASR as ASR 服务<br/>语音识别
    participant Intent as 意图/插件/MCP<br/>可选工具调用
    participant LLM as LLM 服务<br/>大语言模型
    participant TTS as TTS 服务<br/>语音合成

    User->>Dev: 对开发板讲话
    Dev->>Server: 控制信令 hello / listen:start<br/>建立会话、进入录音状态
    Dev->>Server: 上行 Opus 音频流<br/>WebSocket 二进制帧

    loop 持续接收语音帧
        Server->>VAD: 检测当前音频帧是否有人声
        VAD-->>Server: have_voice / silence
        Server->>ASR: 转交音频帧与 VAD 状态
    end

    alt 设备发送 listen:stop 或服务端检测到语音结束
        Server->>ASR: 提交完整语音片段 / 发送流式结束请求
        ASR-->>Server: 识别文本 STT
    else 设备端已完成唤醒词/文本检测
        Dev->>Server: 控制信令 listen:detect + text
    end

    Server-->>Dev: 控制信令 stt<br/>回传识别文本用于展示
    Server->>Intent: 意图识别 / 插件匹配 / MCP 工具判断

    alt 命中本地动作或工具可直接回复
        Intent-->>Server: 动作结果或直接回复文本
    else 需要大模型生成回复
        Server->>LLM: 发送用户文本、上下文、记忆、可用函数定义
        LLM-->>Server: 流式文本片段 / function call
        opt LLM 请求工具调用
            Server->>Intent: 执行插件、IoT、MCP 或外部工具
            Intent-->>Server: 工具结果
            Server->>LLM: 携带工具结果继续生成最终回复
            LLM-->>Server: 最终回复文本片段
        end
    end

    Server-->>Dev: 控制信令 tts:start / sentence_start<br/>通知设备即将播放
    loop LLM 文本可边生成边合成
        Server->>TTS: 发送回复文本片段
        TTS-->>Server: 返回合成音频<br/>Opus/PCM/WAV 等，服务端转为设备可播格式
        Server-->>Dev: 下行 Opus 音频帧<br/>WebSocket 二进制帧
        Dev-->>User: 扬声器播放语音片段
    end

    Server-->>Dev: 控制信令 tts:stop<br/>本轮回复播放结束
    Dev-->>User: 播报最终回复完成
```

## 数据与信令分层

| 阶段 | 主要数据 | 方向 | 说明 |
| --- | --- | --- | --- |
| 建连与会话初始化 | `hello`、会话 ID、设备信息 | 开发板 → 服务器 | 建立实时双向通信，服务器初始化 VAD、ASR、LLM、TTS 等模块。 |
| 开始拾音 | `listen:start` 控制信令 | 开发板 → 服务器 | 服务器清理上一轮音频状态，准备接收新一轮语音。 |
| 上行语音 | Opus 音频二进制帧 | 开发板 → 服务器 | 开发板持续推送麦克风音频，服务器边收边进行 VAD/ASR 处理。 |
| 语音结束 | `listen:stop` 或 VAD 结束判断 | 开发板/服务器内部 | 非流式 ASR 在语音结束后提交完整片段；流式 ASR 则发送结束请求。 |
| 识别结果 | STT 文本 | ASR → 服务器 → 开发板 | ASR 返回用户文本，服务器通过 `stt` 信令回传给设备用于显示。 |
| 语义处理 | 用户文本、上下文、函数定义、工具结果 | 服务器 ↔ LLM/插件/MCP | 服务器先做意图与工具判断；需要时调用 LLM，LLM 可进一步触发插件、IoT 或 MCP 工具。 |
| 语音合成 | 回复文本片段、合成音频 | 服务器 ↔ TTS | LLM 流式输出的文本会进入 TTS 队列，支持边生成边合成。 |
| 下行播报 | `tts:start`、`sentence_start`、Opus 音频、`tts:stop` | 服务器 → 开发板 | 服务器先发送播放状态信令，再发送音频帧，结束时发送 `tts:stop`。 |

## 关键时序说明

1. **控制信令与音频数据分离**：`hello`、`listen`、`stt`、`tts` 等 JSON 文本消息负责驱动状态机；音频则以二进制帧传输。
2. **VAD 与 ASR 协同**：服务器收到每个音频帧后先做 VAD，再把音频与人声状态交给 ASR；流式 ASR 可持续识别，非流式 ASR 通常在 `listen:stop` 后统一识别。
3. **LLM 与工具调用可递归**：如果 LLM 返回 function call，服务器会执行插件、IoT、MCP 或外部工具，并把工具结果补回对话上下文后继续请求 LLM，直到得到可播报的回复。
4. **TTS 可流式播报**：LLM 生成的文本片段会被持续推入 TTS 队列，TTS 音频生成后立即下发给开发板，从而降低首句响应延迟。
5. **打断与状态同步**：若用户在设备播报时再次说话，非手动模式下服务器会发送中止/停止播放相关信令，清理当前 TTS 状态并进入新一轮拾音。
