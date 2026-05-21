# ==============================================================================
# 【一】 镜像编译与推送到阿里云 ACR 指南
# ==============================================================================
# 1. 编译镜像：
#    docker build -t registry.cn-hangzhou.aliyuncs.com/com95jw/jw-freeswitch:v1.0.3 .
# 2. 推送镜像：
#    docker push registry.cn-hangzhou.aliyuncs.com/com95jw/jw-freeswitch:v1.0.3
# ==============================================================================
# 【二】 后续部署与配置修改指南
# ==============================================================================
# 步骤 A：提取默认配置与声音文件
#   docker run --name temp-fs -d registry.cn-hangzhou.aliyuncs.com/com95jw/jw-freeswitch:v1.0.3
#   mkdir -p /www/wwwroot/jw-free-switch/fs-conf /www/wwwroot/jw-free-switch/fs-sounds
#   docker cp temp-fs:/usr/local/freeswitch/conf /www/wwwroot/jw-free-switch/fs-conf/
#   docker cp temp-fs:/usr/local/freeswitch/sounds /www/wwwroot/jw-free-switch/fs-sounds/
#   docker rm -f temp-fs
#
# 步骤 B：激活自定义模块
#   修改宿主机 /www/wwwroot/jw-free-switch/fs-conf/conf/autoload_configs/modules.conf.xml，在 <modules> 标签内加上：
#   <load module="mod_audio_stream"/>
#
# 步骤 C：Host 模式启动并挂载配置（🔥 生产推荐：赋予实时音频内核权限，保障 AI 通话顺畅）
#   docker run -d --name fs-ai-proxy \
#     --network host \
#     --restart always \
#     --privileged \
#     --ulimit rtprio=99 \
#     --ulimit rttime=-1 \
#     --ulimit memlock=-1 \
#     -v /www/wwwroot/jw-free-switch/fs-conf/conf:/usr/local/freeswitch/conf \
#     -v /www/wwwroot/jw-free-switch/fs-sounds/sounds:/usr/local/freeswitch/sounds \
#     registry.cn-hangzhou.aliyuncs.com/com95jw/jw-freeswitch:v1.0.3
# ==============================================================================

# ==========================================
# 阶段 1: 编译阶段 (Builder)
# ==========================================
FROM debian:bookworm AS builder

ENV DEBIAN_FRONTEND=noninteractive

# 安装所有编译期依赖
RUN apt-get update && apt-get install -y \
    git build-essential automake autoconf libtool libtool-bin pkg-config \
    wget cmake uuid-dev libssl-dev libpcre3-dev libspeex-dev libspeexdsp-dev \
    libsqlite3-dev libedit-dev libcurl4-openssl-dev libldns-dev libopus-dev \
    libsndfile1-dev libtiff-dev libjpeg-dev yasm liblua5.2-dev libshout3-dev \
    libmpg123-dev libmp3lame-dev libevent-dev zlib1g-dev \
    libavformat-dev libswscale-dev libavcodec-dev libavutil-dev libswresample-dev \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src

# 1. 编译 libks (🔥【核心加固】：强制将前缀改为 /usr，使其安装到 debian 标准核心共享库目录，避开运行期找不到库的问题)
RUN git clone -b v2.0.6 https://github.com/signalwire/libks.git \
    && cd libks && cmake -DCMAKE_INSTALL_PREFIX=/usr . && make -j$(nproc) && make install

# 2. 编译 signalwire-c (🔥【核心加固】：同样强制指定 /usr 前缀，与 libks 对齐)
RUN git clone -b v2.0.0 https://github.com/signalwire/signalwire-c.git \
    && cd signalwire-c && PKG_CONFIG_PATH=/usr/lib/pkgconfig cmake -DCMAKE_INSTALL_PREFIX=/usr . && make -j$(nproc) && make install

# 3. 独立编译 sofia-sip (SIP 协议栈)
RUN git clone -b v1.13.17 https://github.com/freeswitch/sofia-sip.git \
    && cd sofia-sip && ./bootstrap.sh && ./configure && make -j$(nproc) && make install

# 4. 独立编译 spandsp (锁死历史 commit，解决 v18_init API 冲突)
RUN git clone https://github.com/freeswitch/spandsp.git \
    && cd spandsp \
    && git checkout 0d2e6ac65e0e8f53d652665a743015a88bf048d4 \
    && ./bootstrap.sh && ./configure && make -j$(nproc) && make install

# 显式刷新 builder 阶段动态库缓存
RUN ldconfig

# 5. 编译 FreeSWITCH 主体
RUN git clone -b v1.10.11 https://github.com/signalwire/freeswitch.git freeswitch-src
WORKDIR /usr/src/freeswitch-src
RUN ./bootstrap.sh -j

# 确保开启基础的 SIP (mod_sofia)
RUN sed -i 's/#endpoints\/mod_sofia/endpoints\/mod_sofia/g' modules.conf

# 显式指明依赖扫描链，由于引入了 /usr 前缀，追加对应检索路径
ENV PKG_CONFIG_PATH="/usr/local/lib/pkgconfig:/usr/lib/pkgconfig:/usr/local/freeswitch/lib/pkgconfig"

RUN ./configure --enable-core-odbc-support=no
RUN make -j$(nproc) && make install

# 编译期拉取原生高保真提示音与彩铃包
RUN make cd-sounds-install cd-moh-install

# 6. 采用 CMake 独立编译 mod_audio_stream 模块
WORKDIR /usr/src
RUN git clone --recursive https://github.com/damahaokaixin/mod_audio_stream.git mod_audio_stream
WORKDIR /usr/src/mod_audio_stream

RUN rm -rf build && mkdir build && cd build \
    && PKG_CONFIG_PATH=/usr/local/freeswitch/lib/pkgconfig:/usr/local/lib/pkgconfig:/usr/lib/pkgconfig cmake -DCMAKE_BUILD_TYPE=Release -DUSE_TLS=ON .. \
    && make -j$(nproc) \
    && make install


# ==========================================
# 阶段 2: 最终运行阶段 (Runner)
# ==========================================
FROM debian:bookworm-slim

ENV DEBIAN_FRONTEND=noninteractive

# 精简运行时依赖库
RUN apt-get update && apt-get install -y \
    libsqlite3-0 libcurl4 libpcre3 libedit2 libldns3 libspeex1 libspeexdsp1 \
    libopus0 libsndfile1 libjpeg62-turbo uuid-runtime libssl3 \
    libevent-2.1-7 libevent-pthreads-2.1-7 libevent-openssl-2.1-7 zlib1g libpq5 \
    liblua5.2-0 ffmpeg \
    && rm -rf /var/lib/apt/lists/*

# 从 builder 引入编译好的 FS 主程序和底层 C 共享库
COPY --from=builder /usr/local/freeswitch /usr/local/freeswitch
COPY --from=builder /usr/local/lib /usr/local/lib
# 🔥【核心修复】：同步把安装在标准 /usr 目录下的 libks 和 signalwire 共享库原封不动拷贝进运行环境
COPY --from=builder /usr/lib/libks* /usr/lib/
COPY --from=builder /usr/lib/libsignalwire* /usr/lib/

# 设置系统运行时动态库扫描路径
ENV LD_LIBRARY_PATH="/usr/local/lib:/usr/lib:/lib"

# 强制刷新动态链接全局缓存
RUN ldconfig

# 创建全局快捷指令
RUN ln -s /usr/local/freeswitch/bin/freeswitch /usr/bin/freeswitch
RUN ln -s /usr/local/freeswitch/bin/fs_cli /usr/bin/fs_cli

# 放开声音及 MOH 等目录权限
RUN chmod -R 777 /usr/local/freeswitch/sounds

# 暴露通信端口
EXPOSE 5060/udp 5060/tcp 5080/udp 5080/tcp
EXPOSE 8021/tcp
EXPOSE 16384-32768/udp

# 以前台标准生产模式运行
CMD ["/usr/bin/freeswitch", "-c", "-nonat", "-nf"]
