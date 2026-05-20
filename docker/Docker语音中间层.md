# FreeSWITCH AI 语音辅助中间层 (Docker 版)

本项目提供了一个集成了 `mod_audio_stream` 模块的完整 FreeSWITCH Docker 镜像。主要用于在“话务员 SIP 话机”与“原厂 SIP 服务器”之间搭建一个透明的代理中间层，实现对双向通话的实时无感监听，并通过 WebSocket 将底层 PCM 语音流实时推送到 Golang 后端，以进行 STT 识别和 LLM 话术辅助。

## 🎯 系统架构拓扑

```text
               (SIP 注册 & 呼叫)                        (SIP 桥接 Bridge)
[SIP 话机(话务员)] --------> [ 本地 FreeSWITCH 中间层 ] --------> [ 原厂 SIP 服务器 ] --------> [ 客户 ]
                                     |
                                     | (WebSocket 实时推送 L16 PCM 音频流 - Stereo 左右声道分离)
                                     v
                           [ Golang AI 辅助引擎 ]
                                     |
                           (STT 语音转文本 + LLM 大模型提示)

```

## 📦 第 1 步：拉取镜像

在你的目标服务器（如宝塔面板所在宿主机）上，登录阿里云私有镜像仓库并拉取镜像：

```bash
docker login --username=玖伍陈海天 registry.cn-hangzhou.aliyuncs.com
docker pull registry.cn-hangzhou.aliyuncs.com/com95jw/jw-freeswitch:v1.0.0

```

## 📂 第 2 步：提取默认配置文件

FreeSWITCH 极其依赖 XML 配置文件。我们不能把配置锁死在容器内，必须将其提取到宿主机上进行外挂管理。

创建一个用于存放配置的目录（以 `/www/wwwroot/fs-conf` 为例）：

```bash
mkdir -p /www/wwwroot/fs-conf

```

启动一个临时容器，将配置“偷”出来后销毁：

```bash
docker run --name temp-fs -d registry.cn-hangzhou.aliyuncs.com/com95jw/jw-freeswitch:v1.0.0
docker cp temp-fs:/usr/local/freeswitch/conf /www/wwwroot/fs-conf/
docker rm -f temp-fs

```

*执行完毕后，你的 `/www/wwwroot/fs-conf/conf` 目录下将拥有完整的 FreeSWITCH 配置树。*

## ⚙️ 第 3 步：激活 AI 语音推流模块

编辑宿主机上的模块配置文件：
`/www/wwwroot/fs-conf/conf/autoload_configs/modules.conf.xml`

在文件底部找到 `<modules>` 闭合标签之前，添加以下代码：

```xml
    <!-- 加载 WebSocket 音频推流模块 -->
    <load module="mod_audio_stream"/>
</modules>

```

## 🚀 第 4 步：正式运行容器

**⚠️ 极其重要：** 必须使用 `--network host`（主机网络模式）。SIP 协议和 RTP 媒体流会使用成千上万个 UDP 随机端口，使用 Docker 的 `-p` 端口映射会导致严重的 NAT 穿透问题和 CPU 性能损耗（单通、无声等）。

```bash
docker run -d --name fs-ai-proxy \
  --network host \
  --restart always \
  -v /www/wwwroot/fs-conf/conf:/usr/local/freeswitch/conf \
  registry.cn-hangzhou.aliyuncs.com/com95jw/jw-freeswitch:v1.0.0

```

## 📞 第 5 步：配置 B2BUA 业务逻辑 (核心)

容器跑起来后，你需要修改配置让它起到“承上启下”的作用。每次修改完 XML 文件，无需重启容器，直接在宿主机执行 `docker exec fs-ai-proxy fs_cli -x "reloadxml"` 即可热加载生效。

### 5.1 话务员向下注册

默认情况下，话务员的 SIP 话机可以直接注册到你这台 FreeSWITCH 的 `5060` 端口。默认测试账号为 `1000` 到 `1019`，密码均为 `1234`。
*如需修改密码或新增话务员，请编辑 `/www/wwwroot/fs-conf/conf/directory/default/` 目录下的 XML 文件。*

### 5.2 向上对接原厂 SIP 服务器

编辑外线配置文件，创建一个指向原厂 SIP 服务器的网关（Gateway）：
`/www/wwwroot/fs-conf/conf/sip_profiles/external/upstream.xml`

```xml
<include>
  <gateway name="original_sip_server">
    <!-- 原厂 SIP 服务器的 IP 或域名 -->
    <param name="realm" value="192.168.1.100"/>
    <!-- 向原厂服务器注册的账号和密码 -->
    <param name="username" value="your_agent_account"/>
    <param name="password" value="your_password"/>
    <param name="register" value="true"/>
  </gateway>
</include>

```

### 5.3 拨号计划：拦截呼叫、启动推流并桥接

编辑拨号计划文件，拦截话务员的呼叫，触发推流，然后送给原厂网关：
`/www/wwwroot/fs-conf/conf/dialplan/default.xml`

在 `<extension name="Local_Extension">` 之前，插入你的 AI 拦截规则：

```xml
<extension name="ai_assist_outbound">
  <!-- 匹配话务员拨打的任意外部号码，假设以 0 开头 -->
  <condition field="destination_number" expression="^0(\d+)$">
    
    <!-- 1. 设置推流参数：禁用压缩、设置缓冲帧大小(100ms降低Golang解包压力) -->
    <action application="set" data="STREAM_MESSAGE_DEFLATE=1"/>
    <action application="set" data="STREAM_BUFFER_SIZE=100"/>
    
    <!-- 2. 启动音频推流：使用 stereo 模式分离左右声道，8k 采样率 -->
    <!-- 假设你的 Golang 服务在 127.0.0.1:8080 -->
    <action application="audio_stream" data="ws://127.0.0.1:8080/stream?agent=${caller_id_number} stereo 8k" />
    
    <!-- 3. 将这通电话桥接(Bridge)到原厂 SIP 服务器 -->
    <action application="bridge" data="sofia/gateway/original_sip_server/$1" />
  </condition>
</extension>

```

## 💻 第 6 步：Golang 侧开发对接指南

配置完成后，当话务员拨打电话，你的 Golang 服务将在 WebSocket 接口（`/stream`）收到连接请求。

**通信协议说明：**

1. **连接建立**：WebSocket 握手成功后，FreeSWITCH 首先会发送一段 JSON 格式的 metadata（如果你在拨号计划中配置了的话），或者直接开始发二进制流。
2. **音频数据**：随后不断收到 **Binary (二进制)** 帧。数据格式为 `L16 PCM`（16-bit 线性 PCM）。
3. **声道分离 (Stereo)**：由于你在拨号计划中指定了 `stereo` 模式，收到的 PCM 数据是**交错存取的双声道数据**（即：2字节左声道，2字节右声道，依次交替）。
* **左声道**：通常是打进来的那一方（话务员）。
* **右声道**：通常是被叫方（客户）。
* *在 Golang 中解包时，请按字节拆分数组，分别送入两路独立的 STT（语音识别）引擎通道中，即可完美区分是谁在说话。*



## 🛠️ 常用运维命令

* **进入 FreeSWITCH 控制台 (FS_CLI)**：
可以实时查看 SIP 交互日志和通话事件，排错必备。
```bash
docker exec -it fs-ai-proxy fs_cli

```


* **热加载 XML 配置 (无需重启通话不断)**：

```bash
    docker exec fs-ai-proxy fs_cli -x "reloadxml"
    ```
*   **查看当前活动通话**：
    
```bash
    docker exec fs-ai-proxy fs_cli -x "show calls"
    ```
*   **查看原厂网关注册状态**：
    ```bash
    docker exec fs-ai-proxy fs_cli -x "sofia status"
    ```

```