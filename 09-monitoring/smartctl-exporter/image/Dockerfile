# Multi-arch smartctl_exporter image.
#
# Upstream's image (quay.io/prometheuscommunity/smartctl-exporter) is amd64-only,
# so it can't run on the arm64 Raspberry Pis. Upstream DOES publish per-arch
# static binaries, so we just repackage the official binary + smartmontools into
# a multi-arch image. TARGETARCH is set automatically by `docker buildx` per
# target platform.
#
# Build & push (see README.md):
#   docker buildx build --platform linux/amd64,linux/arm64 \
#     --build-arg VERSION=0.14.0 \
#     -t ghcr.io/mrzelee/smartctl-exporter:v0.14.0 --push .
FROM alpine:3.20

ARG TARGETARCH
ARG VERSION=0.14.0

# smartmontools provides /usr/sbin/smartctl — the chart's default smartctl_location.
RUN apk add --no-cache smartmontools ca-certificates

ADD https://github.com/prometheus-community/smartctl_exporter/releases/download/v${VERSION}/smartctl_exporter-${VERSION}.linux-${TARGETARCH}.tar.gz /tmp/smartctl.tar.gz
RUN tar -xzf /tmp/smartctl.tar.gz -C /tmp \
 && mv /tmp/smartctl_exporter-${VERSION}.linux-${TARGETARCH}/smartctl_exporter /usr/local/bin/smartctl_exporter \
 && rm -rf /tmp/*

EXPOSE 9633
ENTRYPOINT ["/usr/local/bin/smartctl_exporter"]
