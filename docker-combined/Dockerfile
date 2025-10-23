FROM busybox:stable

ARG S6_OVERLAY_VERSION=3.2.0.0
ARG S6_ARCH=x86_64
ADD https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-noarch.tar.xz /tmp
ADD https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-${S6_ARCH}.tar.xz /tmp
RUN \
  tar -C / -Jxpf /tmp/s6-overlay-noarch.tar.xz && \
  tar -C / -Jxpf /tmp/s6-overlay-${S6_ARCH}.tar.xz && \
  rm /tmp/s6-overlay*.tar.xz && \
  ln -s /run /var/run

COPY rootfs /

# Default environment variables
# RELAY: Single relay server (backward compatible, takes precedence over RELAY_SERVERS)
#        Set to "relay.example.com" as placeholder - will be ignored if not changed
# RELAY_SERVERS: Multiple relay servers (comma-separated, supports load balancing & health checks)
#        Example: RELAY_SERVERS=relay1.com:21117,relay2.com:21117
ENV RELAY=
ENV RELAY_SERVERS=
ENV ENCRYPTED_ONLY=0

EXPOSE 21115 21116 21116/udp 21117 21118 21119

HEALTHCHECK --interval=10s --timeout=5s CMD /usr/bin/healthcheck.sh

WORKDIR /data

VOLUME /data

ENTRYPOINT ["/init"]
