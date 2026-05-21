
# FreeSWITCH AI 中间层配置指南 —— SIP 中继对接模式 (SIP Trunk Mode)

**适用场景：** 原厂 SIP 服务器允许你使用 IP 对接或账号对接建立一条“SIP Trunk（中继线）”。
**架构逻辑：** 极大简化配置。中间层和原厂之间只有“一根粗管道”。不需要为几十个话务员建几十个网关。主被叫号码在 Trunk 中透明传递。

## 1. 话机端的配置修改
与多网关模式完全相同：
*   登录所有话机后台，将 SIP Server 指向你这台中间层的 IP。端口 `5060`。账号密码使用中间层 `conf/directory/` 中配置的信息。
*   如果你在宿主机上直接维护分机文件，对应目录为 `/www/wwwroot/jw-free-switch/fs-conf/conf/directory/`。

## 2. 建立统一的 SIP Trunk 网关
容器内路径：`conf/sip_profiles/external/`
宿主机路径：`/www/wwwroot/jw-free-switch/fs-conf/conf/sip_profiles/external/`

如果你有 100 个话务员，在 Trunk 模式下，你也**只需要创建一个**网关文件。
新建 `trunk_upstream.xml`：

**基于账号密码注册的 Trunk (Registration Trunk)：**
```xml
<include>
  <gateway name="main_trunk">
    <param name="realm" value="原厂SIP服务器IP"/>
    <param name="username" value="中继账号"/>
    <param name="password" value="中继密码"/>
    <param name="register" value="true"/>
    <!-- 关键参数：允许主叫号码透传 -->
    <param name="caller-id-in-from" value="true"/> 
  </gateway>
</include>

```

**基于 IP 互信的 Trunk (IP Auth Trunk / 对接不需要密码)：**

```xml
<include>
  <gateway name="main_trunk">
    <param name="realm" value="原厂SIP服务器IP"/>
    <param name="register" value="false"/> <!-- 不需要注册，IP 直连 -->
    <param name="caller-id-in-from" value="true"/>
  </gateway>
</include>

```

## 3. 配置呼出路由 (Outbound Dialplan)

容器内路径：`conf/dialplan/default.xml`
宿主机路径：`/www/wwwroot/jw-free-switch/fs-conf/conf/dialplan/default.xml`

所有话务员的呼出，全部打包塞进这一个 Trunk 中送走。原厂服务器会根据头域中的 Caller ID 自动识别是哪个话务员打的。

```xml
<extension name="ai_outbound_trunk">
  <!-- 匹配呼出外线 -->
  <condition field="destination_number" expression="^0(\d+)$">
    <!-- 1. 启动 AI 双向音频推流 -->
    <action application="set" data="STREAM_MESSAGE_DEFLATE=1"/>
    <action application="set" data="STREAM_BUFFER_SIZE=100"/>
    <action application="audio_stream" data="ws://你的GolangIP:8080/stream?agent=${caller_id_number} stereo 8k" />
    
    <!-- 2. 确保透传主叫号码给原厂 (非常重要，否则原厂不知道是谁打的) -->
    <action application="set" data="effective_caller_id_number=${caller_id_number}"/>
    
    <!-- 3. 固定通过 main_trunk 送出这通电话 -->
    <action application="bridge" data="sofia/gateway/main_trunk/$1" />
  </condition>
</extension>

```

## 4. 配置呼入路由 (Inbound Dialplan)

容器内路径：`conf/dialplan/public.xml`
宿主机路径：`/www/wwwroot/jw-free-switch/fs-conf/conf/dialplan/public.xml`

所有打入 Trunk 的电话，都会携带被叫号码 (Destination Number)。我们直接提取这个号码，桥接给对应的本地分机。

```xml
<extension name="ai_inbound_trunk">
  <!-- 拦截所有通过 Trunk 打进来的电话，分配给 1000-1999 的分机 -->
  <condition field="destination_number" expression="^(1\d{3})$">
    
    <!-- 1. 启动 AI 推流 -->
    <action application="set" data="STREAM_MESSAGE_DEFLATE=1"/>
    <action application="set" data="STREAM_BUFFER_SIZE=100"/>
    <action application="audio_stream" data="ws://你的GolangIP:8080/stream?agent=$1&amp;direction=inbound stereo 8k" />
    
    <!-- 2. 直接根据被叫号码桥接本地话机 -->
    <action application="bridge" data="user/$1" />
  </condition>
</extension>

```

## 5. 生效配置

```bash
docker exec fs-ai-proxy fs_cli -x "reloadxml"
docker exec fs-ai-proxy fs_cli -x "sofia profile external restart"

```
