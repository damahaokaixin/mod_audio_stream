
1. 编译 Docker 镜像。

2. 在宿主机创建项目目录 `/www/wwwroot/jw-free-switch/`，并在其下准备：
   - `/www/wwwroot/jw-free-switch/fs-conf/`
   - `/www/wwwroot/jw-free-switch/fs-sounds/`
   然后运行临时容器，先提取 `conf` 和 `sounds`，删除临时容器后再正式启动容器。

3. 正式运行容器时，将以上两个目录分别挂载到：
   - `/usr/local/freeswitch/conf`
   - `/usr/local/freeswitch/sounds`

4. 配置 FreeSWITCH 网关中继或多网关拨号计划。

5. 使用 Golang 示例代码进行 WebSocket 推流测试与联调。
