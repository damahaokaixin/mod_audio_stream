
# FreeSWITCH AI 中间层配置指南 —— 多网关模拟模式 (Multi-Gateway Mode)

**适用场景：** 原厂 SIP 服务器不提供“中继(Trunk)”对接权限，只允许一个个单独的账号（如 1001, 1002）进行注册。
**架构逻辑：** 你的中间层作为“背靠背代理”。话机向中间层注册，中间层代替话机向原厂注册。

## 1. 话机端的配置修改
你需要登录每个话务员办公桌上的物理 SIP 话机（或软电话）的后台网页。
*   **SIP 服务器/代理 (SIP Server/Proxy)：** 修改为你这台 FreeSWITCH 中间层的 IP 地址。
*   **SIP 端口：** `5060`
*   **分机号 (Username/Extension)：** 保持原样（如 `1001`）。
*   **密码 (Password)：** 设置为你在 FreeSWITCH 中设定的密码（默认是 `1234`，可在第 2 步修改）。

## 2. 配置中间层本地分机 (让话机能连上)
容器内路径：`conf/directory/default/`
宿主机路径：`/www/wwwroot/jw-free-switch/fs-conf/conf/directory/default/`

FreeSWITCH 默认已经创建了 1000-1019 的分机。如果你的话务员账号在这些范围内，直接使用即可。
如果需要新建（例如 8001），复制模板并修改：
```bash
cp 1000.xml 8001.xml

```

编辑 `8001.xml`，将里面的 `id` 修改为 `8001`，修改 `password` 字段即可。

## 3. 建立向上游（原厂）注册的网关

容器内路径：`conf/sip_profiles/external/`
宿主机路径：`/www/wwwroot/jw-free-switch/fs-conf/conf/sip_profiles/external/`

为你拥有的每一个原厂账号，创建一个独立的网关配置文件。例如 `gw_1001.xml` 和 `gw_1002.xml`。

**`gw_1001.xml` 示例：**

```xml
<include>
  <!-- 网关名字必须以 gw_ 开头加上分机号，这是后续动态路由匹配的关键 -->
  <gateway name="gw_1001">
    <param name="realm" value="原厂SIP服务器IP"/>
    <param name="username" value="1001"/> <!-- 原厂账号 -->
    <param name="password" value="原厂密码"/>
    <param name="register" value="true"/> <!-- 开启自动注册 -->
  </gateway>
</include>

```

## 4. 配置呼出路由 (Outbound Dialplan)

容器内路径：`conf/dialplan/default.xml`
宿主机路径：`/www/wwwroot/jw-free-switch/fs-conf/conf/dialplan/default.xml`

当话务员打电话给客户时，拦截、推流 AI、并使用对应的网关呼出。将此规则加在 `Local_Extension` 之前：

```xml
<extension name="ai_outbound_multi">
  <!-- 匹配呼出号码（假设加拨 0 呼出，正则去除 0） -->
  <condition field="destination_number" expression="^0(\d+)$">
    <!-- 1. 启动 AI 双向音频推流 -->
    <action application="set" data="STREAM_MESSAGE_DEFLATE=1"/>
    <action application="set" data="STREAM_BUFFER_SIZE=100"/>
    <action application="audio_stream" data="ws://你的GolangIP:8080/stream?agent=${caller_id_number} stereo 8k" />
    
    <!-- 2. 动态桥接：根据拨号的话务员分机，找到它专属的网关送出 -->
    <!-- 如果是 1001 拨打，这里解析为 sofia/gateway/gw_1001/$1 -->
    <action application="bridge" data="sofia/gateway/gw_${caller_id_number}/$1" />
  </condition>
</extension>

```

## 5. 配置呼入路由 (Inbound Dialplan)

容器内路径：`conf/dialplan/public.xml`
宿主机路径：`/www/wwwroot/jw-free-switch/fs-conf/conf/dialplan/public.xml`

当客户拨打原厂号码，原厂将电话送到中间层时，进行拦截、推流，并分发给对应的本地话机。

```xml
<extension name="ai_inbound_multi">
  <!-- 假设你的话务员分机是 1001 到 1005 -->
  <condition field="destination_number" expression="^(1001|1002|1003|1004|1005)$">
    <!-- 1. 启动 AI 推流。注意 inbound 时，agent 是被叫 $1 -->
    <action application="set" data="STREAM_MESSAGE_DEFLATE=1"/>
    <action application="set" data="STREAM_BUFFER_SIZE=100"/>
    <action application="audio_stream" data="ws://你的GolangIP:8080/stream?agent=$1&amp;direction=inbound stereo 8k" />
    
    <!-- 2. 桥接到注册在中间层上的本地话务员话机 -->
    <action application="bridge" data="user/$1" />
  </condition>
</extension>

```

## 6. 生效配置

在宿主机执行以下命令，无需重启容器：

```bash
docker exec fs-ai-proxy fs_cli -x "reloadxml"
docker exec fs-ai-proxy fs_cli -x "sofia profile external restart"

```
