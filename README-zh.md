# mod_audio_stream

**面向生产环境的 FreeSWITCH WebSocket 音频流模块。**

在 FreeSWITCH 与外部系统之间进行实时音频流传输，具备正确的生命周期管理、线程安全与可预测的内存使用。

## 更新（2025/2/22）

### :rocket: 推出「双向流」与「自动播放」

我们已发布新版本 **mod_audio_stream v1.0.3**，支持通过 WebSocket 进行原始二进制音频流传输。
可在 Releases 页面下载，并提供 Debian 12 预编译安装包。

播放功能允许在持续前向推流的同时，播放线程独立运行，从而实现主叫与 WebSocket 端点之间的全双工音频（full-duplex）。

主要特性：

- 全双工音频流（呼叫方 ↔ WebSocket）
- 同时支持 base64 编码音频与原始二进制音频
- 播放可被跟踪、暂停与恢复

🔹 本版本为**商业产品**，可免费使用（包括商业用途），但有 **10 路并发流通道限制**。
如需超过 10 路并发，或需要源码，请通过邮件联系获取许可方案：[amsoftswitch@gmail.com](mailto:amsoftswitch@gmail.com)。

### 为什么会有商业版

社区版 `mod_audio_stream` 提供可用于生产环境的**单向** WebSocket 音频流能力，适用于 ASR 与实时音频处理等场景。
而商业版存在的原因是：真实的电话系统在**高并发与生产负载**下会出现一些只有在实践中才会暴露的复杂工程问题，例如：

- 正确处理 FreeSWITCH 会话生命周期
- 音频注入与关闭流程的线程安全
- 高负载下的安全重连与资源清理
- 有界且可预测的内存使用
- 与 record_session / uuid_record 的正确交互

商业版面向**高并发环境（数千路同时通话，5000+）**设计并测试，强调正确性、稳定性与资源占用可控。

## 关于

- `mod_audio_stream` 旨在提供一个简单、低依赖、但足够有效的模块，用于将音频推送到 WebSocket 服务端并接收响应。
- 引入 [libwsc](https://github.com/amigniter/libwsc)：我们自研的、**兼容 RFC-6455** 的 WebSocket 客户端，专为 `mod_audio_stream` 设计。
  - 替换了此前使用多年的 [ixwebsocket](https://machinezone.github.io/IXWebSocket/)；`libwsc` 基于 libevent，极轻量，并针对低延迟音频流做了优化。
- 本模块受 mod_audio_fork 启发。

## 安装

### 依赖

在 Debian/Ubuntu 上需要安装 `libfreeswitch-dev`、`libssl-dev`、`zlib1g-dev`、`libevent-dev`、`libspeexdsp-dev`（这些通常也是安装 FreeSWITCH 时的常规依赖）。

### 编译

克隆仓库后请执行：**git submodule init** 与 **git submodule update** 初始化子模块。

#### 自定义路径

如果你从源码编译并安装 FreeSWITCH，例如安装目录是 `/usr/local/freeswitch`，需要把 pkgconfig 路径加入环境变量：

```bash
export PKG_CONFIG_PATH=/usr/local/freeswitch/lib/pkgconfig
```

在仓库目录中执行构建：

```bash
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make
sudo make install
```

**TLS** 默认是 `OFF`。若要启用 TLS 支持，请在 cmake 中加入 `-DUSE_TLS=ON`。

#### DEB 包

完成模块构建后可生成 DEB 包：

```bash
cpack -G DEB
```

生成的 Debian 包位于仓库根目录的 `_packages` 文件夹中。

## 脚本化构建与安装

```bash
sudo apt-get -y install git \
    && cd /usr/src/ \
    && git clone https://github.com/amigniter/mod_audio_stream.git \
    && cd mod_audio_stream \
    && sudo bash ./build-mod-audio-stream.sh
```

## 通道变量（Channel variables）

以下通道变量可用于微调 WebSocket 连接行为，并可配置 mod_audio_stream 的日志输出：

| 变量                                   | 说明                                                    | 默认值 |
| -------------------------------------- | ------------------------------------------------------- | ------ |
| STREAM_MESSAGE_DEFLATE                 | true 或 1，禁用 per message deflate                     | off    |
| STREAM_HEART_BEAT                      | 心跳间隔（秒），无流量时发送心跳                         | off    |
| STREAM_SUPPRESS_LOG                    | true 或 1，抑制打印到日志                                | off    |
| STREAM_BUFFER_SIZE                     | 缓冲时长（毫秒），需能被 20 整除                         | 20     |
| STREAM_EXTRA_HEADERS                   | 额外请求头，JSON 对象的字符串格式                        | none   |
| ~~STREAM_NO_RECONNECT~~                | true 或 1，禁用自动重连（已废弃）                        | off    |
| STREAM_TLS_CA_FILE                     | CA 证书/证书包，或特殊值 SYSTEM / NONE                  | SYSTEM |
| STREAM_TLS_KEY_FILE                    | WSS 可选客户端私钥文件                                    | none   |
| STREAM_TLS_CERT_FILE                   | WSS 可选客户端证书文件                                    | none   |
| STREAM_TLS_DISABLE_HOSTNAME_VALIDATION | true 或 1，禁用 WSS 主机名校验                            | false  |

- per message deflate 压缩默认开启，可显著节省带宽；若要关闭请将变量设为 `true|1`。
- 心跳：当连接空闲时每隔 N 秒发送一次，避免负载均衡器清理空闲连接。
- 抑制日志：默认不抑制（false）；WebSocket 服务端返回的响应默认会打印到日志。若担心日志刷屏可设为 `true|1`。事件仍会照常触发，仅影响日志打印。
- `STREAM_BUFFER_SIZE` 表示每次发送给 WebSocket 的音频块对应的时长。例如想每次发送 100ms 的音频数据，则设置为 100。若省略，默认发送 20ms（即 FreeSWITCH 默认帧大小）。
- `STREAM_EXTRA_HEADERS` 必须是 JSON 对象的字符串，键为 HTTP Header 名，值为字符串。例如：

  ```json
  {
      "Header1": "Value1",
      "Header2": "Value2",
      "Header3": "Value3"
  }
  ```

- ~~WebSocket 自动重连默认开启，设为 true 或 1 可禁用。~~
  - libwsc 不支持自动重连。
- WSS（TLS）相关参数通过 `STREAM_TLS_*` 调整：
  - `STREAM_TLS_CA_FILE`：CA 证书（或证书包）文件路径。默认 `SYSTEM` 表示使用系统默认。也可设为 `NONE`，表示不校验对端证书。
  - `STREAM_TLS_CERT_FILE`：可选的客户端 TLS 证书文件，会发送给服务端。
  - `STREAM_TLS_KEY_FILE`：上述证书对应的可选客户端私钥文件。
  - `STREAM_TLS_DISABLE_HOSTNAME_VALIDATION`：设为 `true` 时禁用对端证书的主机名匹配校验；默认 `false`，会强制主机名匹配。

## API

### 命令

模块暴露以下 FreeSWITCH API 命令：

```text
uuid_audio_stream <uuid> start <wss-url> <mix-type> <sampling-rate> <metadata>
```

为通话挂载 media bug，并开始把音频（L16 格式）推送到 WebSocket 服务端。FreeSWITCH 默认采样率为 8k。若指定非 8k，模块会进行重采样。

- `uuid`：FreeSWITCH 通道唯一 ID
- `wss-url`：WebSocket URL，`ws://` 或 `wss://`
- `mix-type`：可选值
  - `mono`：单声道，仅包含主叫音频
  - `mixed`：单声道，混合主叫与被叫音频
  - `stereo`：双声道，一路为主叫，一路为被叫
- `sampling-rate`：可选值
  - `8k`：生成 8000 Hz 采样率
  - `16k`：生成 16000 Hz 采样率
- `metadata`：（可选）合法的 `utf-8` 文本，会在音频推流开始前先发送一次

```text
uuid_audio_stream <uuid> send_text <metadata>
```

向 WebSocket 服务端发送文本消息（需要合法 `utf-8` 文本）。

```text
uuid_audio_stream <uuid> stop <metadata>
```

停止音频流并关闭 WebSocket 连接。若提供 `metadata`，将会在关闭连接前发送。

```text
uuid_audio_stream <uuid> pause
```

暂停音频流。

```text
uuid_audio_stream <uuid> resume
```

恢复音频流。

## 事件

模块会产生以下事件类型：

- `mod_audio_stream::json`
- `mod_audio_stream::connect`
- `mod_audio_stream::disconnect`
- `mod_audio_stream::error`
- `mod_audio_stream::play`

### response

从 WebSocket 端点收到的消息。预期是 JSON，但实际上包含服务端返回的任意内容。

#### 触发的 FreeSWITCH 事件

- **Name**：`mod_audio_stream::json`
- **Body**：WebSocket 服务端响应内容

### connect

成功连接到 WebSocket 服务端。

#### 触发的 FreeSWITCH 事件

- **Name**：`mod_audio_stream::connect`
- **Body**：JSON

```json
{
	"status": "connected"
}
```

### disconnect

与 WebSocket 服务端断开连接。

#### 触发的 FreeSWITCH 事件

- **Name**：`mod_audio_stream::disconnect`
- **Body**：JSON

```json
{
	"status": "disconnected",
	"message": {
		"code": 1000,
		"reason": "Normal closure"
	}
}
```

- code：`<int>`
- reason：`<string>`

### error

连接发生错误。事件中会包含多个字段用于描述错误。

#### 触发的 FreeSWITCH 事件

- **Name**：`mod_audio_stream::error`
- **Body**：JSON

```json
{
	"status": "error",
	"message": {
		"code": 1,
		"error": "String explaining the error"
	}
}
```

- code：`<int>`
- error：`<string>`

#### 可能的 `code` 值

| Code | 枚举名                | 含义                                                   |
|:----:|:----------------------|:-------------------------------------------------------|
| 1    | `IO`                  | 读写 socket 的 I/O 错误                                 |
| 2    | `INVALID_HEADER`      | 服务端发送了格式错误的 WebSocket Header                 |
| 3    | `SERVER_MASKED`       | 服务端帧被 mask（规范不允许）                           |
| 4    | `NOT_SUPPORTED`       | 请求的特性不被支持（例如扩展）                           |
| 5    | `PING_TIMEOUT`        | 超时未收到 PONG                                         |
| 6    | `CONNECT_FAILED`      | TCP 连接失败或 DNS 解析失败                              |
| 7    | `TLS_INIT_FAILED`     | SSL/TLS 上下文初始化失败                                 |
| 8    | `SSL_HANDSHAKE_FAILED`| 与服务端进行 SSL/TLS 握手失败                             |
| 9    | `SSL_ERROR`           | OpenSSL 通用错误（证书、加密套件等）                     |
| 10   | `TIMEOUT`             | 超时                                                   |
| 11   | `PROTOCOL`            | WebSocket 协议错误                                      |

### play

- **Name**：`mod_audio_stream::play`
- **Body**：JSON

WebSocket 服务端可能返回包含 base64 编码音频的 JSON 对象，模块将其播放给用户。要启用此特性，服务端响应必须符合如下格式：

```json
{
  "type": "streamAudio",
  "data": {
    "audioDataType": "raw",
    "sampleRate": 8000,
    "audioData": "base64 encoded audio"
  }
}
```

- audioDataType：`<raw|wav|mp3|ogg>`

模块触发的事件（子类：`mod_audio_stream::play`）内容与 `data` 元素一致，并额外加入 `file` 字段表示生成的文件路径（filePath）：

```json
{
  "audioDataType": "raw",
  "sampleRate": 8000,
  "file": "/path/to/the/file"
}
```

若未抑制日志输出，控制台中打印的 `response` 与事件内容一致。原始响应中包含的 base64 音频可能非常大，因此模块会用上述结构替代打印，以避免日志过大。

该特性生成的文件均位于临时目录，并会在会话关闭时被删除。

