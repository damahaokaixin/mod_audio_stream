# ==============================================================================
# 【一】 镜像编译与推送到阿里云 ACR 指南
# ==============================================================================
# 1. 编译镜像：
#    docker build -t registry.cn-hangzhou.aliyuncs.com/com95jw/jw-freeswitch:v1.0.0 .
# 2. 推送镜像：
#    docker push registry.cn-hangzhou.aliyuncs.com/com95jw/jw-freeswitch:v1.0.0
# ==============================================================================
# 【二】 后续部署与配置修改指南
# ==============================================================================
# 步骤 A：提取默认配置
#   docker run --name temp-fs -d registry.cn-hangzhou.aliyuncs.com/com95jw/jw-freeswitch:v1.0.0
#   docker cp temp-fs:/usr/local/freeswitch/conf /你的宿主机绝对路径/fs-conf
#   docker rm -f temp-fs
#
# 步骤 B：激活自定义模块
#   修改宿主机 fs-conf/autoload_configs/modules.conf.xml，在 <modules> 标签内加上：
#   <load module="mod_audio_stream"/>
#
# 步骤 C：Host 模式启动并挂载配置
#   docker run -d --name fs-ai-proxy --network host \
#     -v /你的宿主机绝对路径/fs-conf:/usr/local/freeswitch/conf \
#     registry.cn-hangzhou.aliyuncs.com/com95jw/jw-freeswitch:v1.0.0
# ==============================================================================

# ==========================================
# 阶段 1: 编译阶段 (Builder)
# ==========================================


FROM debian:bookworm AS builder

ENV DEBIAN_FRONTEND=noninteractive

# 安装所有依赖（包含 FFmpeg 和 数据库开发库，满足完整版需求）
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

# 1. 编译 libks (WebRTC 核心依赖)
RUN git clone https://github.com/signalwire/libks.git \
    && cd libks && cmake . && make -j$(nproc) && make install

# 2. 编译 signalwire-c (WebRTC 通信依赖)
RUN git clone https://github.com/signalwire/signalwire-c.git \
    && cd signalwire-c && PKG_CONFIG_PATH=/usr/local/lib/pkgconfig cmake . && make -j$(nproc) && make install

# 3. 独立编译 sofia-sip (SIP 协议栈)
RUN git clone https://github.com/freeswitch/sofia-sip.git \
    && cd sofia-sip && ./bootstrap.sh && ./configure && make -j$(nproc) && make install

# 4. 独立编译 spandsp (⚠️ 关键修复：锁死历史 commit，解决 v18_init API 冲突)
RUN git clone https://github.com/freeswitch/spandsp.git \
    && cd spandsp \
    && git checkout 0d2e6ac65e0e8f53d652665a743015a88bf048d4 \
    && ./bootstrap.sh && ./configure && make -j$(nproc) && make install

# 5. 编译 FreeSWITCH 主体 (完整默认版本)
RUN git clone -b v1.10.11 https://github.com/signalwire/freeswitch.git freeswitch-src
WORKDIR /usr/src/freeswitch-src
RUN ./bootstrap.sh -j

# 确保开启基础的 SIP (mod_sofia)，不关闭任何其他模块
RUN sed -i 's/#endpoints\/mod_sofia/endpoints\/mod_sofia/g' modules.conf

# 修复变量警告，并指引编译器寻找 libks
ENV PKG_CONFIG_PATH="/usr/local/lib/pkgconfig:${PKG_CONFIG_PATH:-}"

RUN ./configure --enable-core-odbc-support=no
RUN make -j$(nproc) && make install

# 6. 采用 CMake 独立编译 mod_audio_stream 模块
WORKDIR /usr/src
RUN git clone --recursive https://github.com/damahaokaixin/mod_audio_stream.git mod_audio_stream
WORKDIR /usr/src/mod_audio_stream
# 导出所有的 pkgconfig 路径，确保找到 FS 和 libevent
RUN export PKG_CONFIG_PATH=/usr/local/freeswitch/lib/pkgconfig:/usr/local/lib/pkgconfig \
    && mkdir build && cd build \
    && cmake -DCMAKE_BUILD_TYPE=Release -DUSE_TLS=ON .. \
    && make -j$(nproc) \
    && make install



# ==========================================
# 阶段 2: 最终运行阶段 (Runner)
# ==========================================
FROM debian:bookworm-slim

ENV DEBIAN_FRONTEND=noninteractive

# 补齐所有运行时的动态库（新增了 libevent-openssl-2.1-7 支持 WSS 加密推流）
RUN apt-get update && apt-get install -y \
    libsqlite3-0 libcurl4 libpcre3 libedit2 libldns3 libspeex1 libspeexdsp1 \
    libopus0 libsndfile1 libjpeg62-turbo uuid-runtime libssl3 \
    libevent-2.1-7 libevent-pthreads-2.1-7 libevent-openssl-2.1-7 zlib1g libpq5 ffmpeg \
    libavformat-dev libswscale-dev libavcodec-dev libavutil-dev libswresample-dev \
    liblua5.2-0 \
    && rm -rf /var/lib/apt/lists/*

# 从 builder 引入编译好的 FS 主程序和底层 C 共享库
COPY --from=builder /usr/local/freeswitch /usr/local/freeswitch
COPY --from=builder /usr/local/lib /usr/local/lib

# 设置环境变量，指引系统去读取拷贝的动态库 (修复警告)
ENV LD_LIBRARY_PATH="/usr/local/lib:${LD_LIBRARY_PATH:-}"

# 创建全局快捷指令
RUN ln -s /usr/local/freeswitch/bin/freeswitch /usr/bin/freeswitch
RUN ln -s /usr/local/freeswitch/bin/fs_cli /usr/bin/fs_cli

# 暴露必要端口
EXPOSE 5060/udp 5060/tcp 5080/udp 5080/tcp
EXPOSE 8021/tcp
EXPOSE 16384-32768/udp

# 以前台控制台模式启动
CMD ["/usr/bin/freeswitch", "-c", "-nonat", "-nf"]
