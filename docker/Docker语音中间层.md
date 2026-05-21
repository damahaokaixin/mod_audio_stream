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
docker pull registry.cn-hangzhou.aliyuncs.com/com95jw/jw-freeswitch:v1.0.3

```

## 📂 第 2 步：提取默认配置文件

FreeSWITCH 的配置文件和系统音频资源都建议提取到宿主机进行外挂管理。这样后续无论是修改拨号计划，还是替换提示音、彩铃、IVR 音频，都能直接在宿主机维护。

建议先创建一个项目根目录，例如 `/www/wwwroot/jw-free-switch/`，再把 `conf` 和 `sounds` 都放在这个目录下统一管理。

```bash
mkdir -p /www/wwwroot/jw-free-switch/fs-conf
mkdir -p /www/wwwroot/jw-free-switch/fs-sounds

```

启动一个临时容器，将配置“偷”出来后销毁：

```bash
docker run --name temp-fs -d registry.cn-hangzhou.aliyuncs.com/com95jw/jw-freeswitch:v1.0.3
docker cp temp-fs:/usr/local/freeswitch/conf /www/wwwroot/jw-free-switch/fs-conf/
docker cp temp-fs:/usr/local/freeswitch/sounds /www/wwwroot/jw-free-switch/fs-sounds/
docker rm -f temp-fs

```

*执行完毕后，你的 `/www/wwwroot/jw-free-switch/fs-conf/conf` 目录下将拥有完整的 FreeSWITCH 配置树，而 `/www/wwwroot/jw-free-switch/fs-sounds/sounds` 目录下将拥有完整的系统声音资源。*

## ⚙️ 第 3 步：激活 AI 语音推流模块

编辑宿主机上的模块配置文件：
`/www/wwwroot/jw-free-switch/fs-conf/conf/autoload_configs/modules.conf.xml`

在文件底部找到 `<modules>` 闭合标签之前，添加以下代码：

```xml
    <!-- 加载 WebSocket 音频推流模块 -->
    <load module="mod_audio_stream"/>
</modules>

```

## 🚀 第 4 步：正式运行容器

**⚠️ 极其重要：** 必须使用 `--network host`（主机网络模式）。SIP 协议和 RTP 媒体流会使用成千上万个 UDP 随机端口，使用 Docker 的 `-p` 端口映射会导致严重的 NAT 穿透问题和 CPU 性能损耗（单通、无声等）。

如果你的宿主机需要为 FreeSWITCH 提供更稳定的实时调度能力，建议在正式运行时一并开启高优先级相关参数。推荐直接使用下面这条生产版运行命令：

```bash
docker run -d --name fs-ai-proxy \
  --network host \
  --restart always \
  --privileged \
  --ulimit rtprio=99 \
  --ulimit rttime=-1 \
  --ulimit memlock=-1 \
  -v /www/wwwroot/jw-free-switch/fs-conf/conf:/usr/local/freeswitch/conf \
  -v /www/wwwroot/jw-free-switch/fs-sounds/sounds:/usr/local/freeswitch/sounds \
  registry.cn-hangzhou.aliyuncs.com/com95jw/jw-freeswitch:v1.0.3

```

## 🧩 第 4 步补充：生产部署关键说明

在标准容器启动后，若希望该中间层能够稳定用于生产环境，建议继续完成以下三项补充配置：配置外挂完整性确认、WebRTC 的 WSS 证书配置，以及公网 IP / 防火墙端口放行。

### 4.1 同时外挂 `conf` 和 `sounds` 的推荐方式

如果你的部署目标不仅包含 AI 语音中间层，还希望后续方便维护提示音、IVR、彩铃或其他自定义音频资源，那么 **同时外挂 `conf` 和 `sounds` 是更灵活的做法**。

FreeSWITCH 的运行结构本身就是“程序 / 媒体资源 / 配置逻辑”相互解耦的：

- `/usr/local/freeswitch/conf`：包含模块加载规则、SIP Profile、Gateway、Dialplan、Directory 等所有核心业务控制逻辑。你后续对接 AI、修改路由、增加话务员分机、配置外线网关，几乎都只会改这里。
- `/usr/local/freeswitch/sounds`：这是系统提示音、彩铃、IVR 语音和本地媒体资源目录。如果后续有中文化、替换欢迎语、自定义业务提示音等需求，把它外挂到宿主机上会更方便。

因此，推荐使用以下双挂载方式：

```bash
-v /www/wwwroot/jw-free-switch/fs-conf/conf:/usr/local/freeswitch/conf
-v /www/wwwroot/jw-free-switch/fs-sounds/sounds:/usr/local/freeswitch/sounds
```

这样既能保证核心配置持久化，也能让声音文件的维护、备份和替换更加直接，尤其适合实际生产项目中需要频繁调整播报内容的场景。

### 4.2 网页端 WebRTC (WSS) 证书配置

如果你后续需要让网页端通过 WebRTC / Verto 接入 FreeSWITCH，那么 **不能继续使用镜像里的自签名证书**。当你的前端页面是 `https://` 域名时，浏览器通常会直接拦截自签名 WSS 连接，表现为网页无法注册、无法呼叫或连接刚建立就被断开。

#### 第一步：准备正式证书并生成 `wss.pem`

将你的正式证书文件和私钥文件合并为一个供 FreeSWITCH 使用的 `pem` 文件：

```bash
mkdir -p /www/wwwroot/jw-free-switch/fs-conf/conf/certs
cat /你的证书路径/your_domain.crt /你的私钥路径/your_domain.key > /www/wwwroot/jw-free-switch/fs-conf/conf/certs/wss.pem
```

#### 第二步：在 `verto.conf.xml` 中指定证书

编辑配置文件：
`/www/wwwroot/jw-free-switch/fs-conf/conf/autoload_configs/verto.conf.xml`

确认以下参数存在且未被注释，并指向外挂目录中的证书文件：

```xml
<param name="ssl-key-path" value="$${conf_dir}/certs/wss.pem"/>
<param name="ssl-cert-path" value="$${conf_dir}/certs/wss.pem"/>
```

#### 第三步：重启容器使证书生效

证书路径变更通常建议直接重启容器：

```bash
docker restart fs-ai-proxy
```

如果你只做传统 SIP 话机接入，不涉及网页 WebRTC / Verto，则这一部分可以暂时跳过。

### 4.3 公网 IP、云安全组与防火墙端口放行

由于容器使用的是 `--network host`，FreeSWITCH 实际上直接占用了宿主机的网络栈。也就是说：

- 不需要再写 Docker 的 `-p` 端口映射。
- 但必须在云服务器安全组、宿主机防火墙、边界 NAT / 端口映射设备上，确保相关端口真实可达。

建议至少放通以下端口：

| 端口号 / 范围 | 协议 | 用途 | 是否必须 |
| --- | --- | --- | --- |
| `5060` | `UDP` / `TCP` | 标准 SIP 信令端口，用于对接常规话机与线路商中继 | 必须 |
| `8081` 或 `8082` | `TCP` | Verto / WebRTC 的 WSS 通道 | 仅网页通话时必须 |
| `16384-32768` | `UDP` | RTP 音频媒体流端口范围 | 必须 |
| `8021` | `TCP` | ESL 事件控制接口 | 仅建议内网开放 |

**⚠️ 生产环境最常见的坑：**

如果出现“电话能接通、信令正常、AI 也能收到流，但双方听不到声音”这类问题，绝大多数都不是拨号计划写错，而是 **`16384-32768/UDP` 没有在云安全组或宿主机防火墙中完整放行**。这是 FreeSWITCH 部署中最常见的单通 / 无声根因之一。

### 4.4 外网 IP 映射的额外提醒

如果你的服务器位于云主机、公网 NAT 或多网卡环境下，除了放行端口外，还应确认 FreeSWITCH 对外宣告的 SIP / RTP 地址是正确的公网地址。否则常见现象是：

- 注册成功，但通话建立后无声
- 对端只能单向听到声音
- 网页端偶发可呼叫，但媒体协商不稳定

此时建议重点检查以下配置是否填写为公网 IP 或公网域名：

- `sip_profiles/internal.xml`
- `sip_profiles/external.xml`
- `vars.xml`

尤其关注 `ext-sip-ip`、`ext-rtp-ip`、`sip-ip`、`rtp-ip` 等参数。如果部署在 NAT 后面，务必结合你的实际公网出口地址进行配置。

## 📞 第 5 步：配置 B2BUA 业务逻辑 (核心)

容器跑起来后，你需要修改配置让它起到“承上启下”的作用。每次修改完 XML 文件，无需重启容器，直接在宿主机执行 `docker exec fs-ai-proxy fs_cli -x "reloadxml"` 即可热加载生效。

### 5.1 话务员向下注册

默认情况下，话务员的 SIP 话机可以直接注册到你这台 FreeSWITCH 的 `5060` 端口。默认测试账号为 `1000` 到 `1019`，密码均为 `1234`。
*如需修改密码或新增话务员，请编辑 `/www/wwwroot/jw-free-switch/fs-conf/conf/directory/default/` 目录下的 XML 文件。*

### 5.2 向上对接原厂 SIP 服务器

编辑外线配置文件，创建一个指向原厂 SIP 服务器的网关（Gateway）：
`/www/wwwroot/jw-free-switch/fs-conf/conf/sip_profiles/external/upstream.xml`

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
`/www/wwwroot/jw-free-switch/fs-conf/conf/dialplan/default.xml`

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
