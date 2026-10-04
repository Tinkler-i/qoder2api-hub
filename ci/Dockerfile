# ============================================================================
#  ci/Dockerfile —— qoder2api-hub 的 glibc 变体（fork 本地补丁）
#
#  为什么需要它：
#    上游 Dockerfile 的最终镜像是 python:3.11-alpine（musl）。而 v1.2.3 引入的
#    UMID 原生组件（从 @qoder-ai/qodercli 提取）是**动态链接的 glibc C++ 二进制**：
#        PT_INTERP = /lib64/ld-linux-x86-64.so.2
#        DT_NEEDED = libstdc++.so.6 libm.so.6 libpthread.so.0 libgcc_s.so.1
#                    librt.so.1 libdl.so.2 libc.so.6
#        GLIBC 符号 = 最高 GLIBC_2.28
#    Alpine 里没有 /lib64/ld-linux-x86-64.so.2 → execve ENOENT → 网关回退
#    derived 身份 → 国际版活动列表被服务端过滤、签不到账。
#    （上游作者在 Release 里注明 Docker 构建未端到端验证，此处补上。）
#
#  基准：上游 v1.2.3 的 Dockerfile，仅替换基础镜像与包管理器，其余逐字保留。
#        上游以后改了 Dockerfile（新增 COPY 文件等），请对照同步本文件。
#        若上游自己换成 glibc 基础镜像，可直接删掉本文件。
# ============================================================================

FROM python:3.11-slim AS umid-builder
ARG TARGETARCH
WORKDIR /build
COPY _install_umid.py ./
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates \
 && rm -rf /var/lib/apt/lists/* \
 && arch="x64"; [ "$TARGETARCH" = "arm64" ] && arch="arm64"; \
    python _install_umid.py --platform linux --arch "$arch" --dest /build/umid \
    || echo "WARN: UMID extraction failed (gateway will fall back to derived identity)"; \
    mkdir -p /build/umid

FROM python:3.11-slim

ENV PYTHONUNBUFFERED=1 \
    HOST=0.0.0.0 \
    PORT=8790 \
    API_KEY= \
    TZ=Asia/Shanghai

WORKDIR /app

# Debian：时区 + 证书；libstdc++6 是 UMID 组件的运行时依赖，必须显式安装
RUN apt-get update \
 && apt-get install -y --no-install-recommends tzdata ca-certificates libstdc++6 \
 && rm -rf /var/lib/apt/lists/* \
 && ln -snf /usr/share/zoneinfo/${TZ} /etc/localtime \
 && echo "${TZ}" > /etc/timezone

# Copy application files (Zero external pip dependencies)
COPY qoder_proxy.py qoder_accounts.py qoder_catalog.py qoder_fingerprint.py \
     qoder_scheduler.py qoder_settings.py qoder_sign.py qoder_tasks.py \
     dashboard.html baseprompt.json ./

# 官方模型目录快照
COPY qoder_catalog_intl.json qoder_catalog_cn.json ./

# UMID 原生组件（构建期提取；运行时 qoder_accounts.runtime_info_exe() 按
# <repo>/umid/runtime-info 查找）
COPY --from=umid-builder /build/umid /app/umid

# 验证 / 诊断脚本
COPY _test_qoder.py _diag_gateway.py _diag_campaign.py _verify_models.py \
     _refresh_catalog.py _install_umid.py ./

RUN mkdir -p /app/accounts /app/usage

VOLUME ["/app/accounts", "/app/usage"]

EXPOSE 8790

CMD ["python", "qoder_proxy.py"]
