FROM quchao/base
MAINTAINER Chao QU <mail@quchao.com>

ENV SS_VERSION='3.0.6' \
    SS_SHA256='7d9b43b0235a57c115bfe160efd54abef96bffcbfff61c5496e7c2800f0734ca' \
    OBFS_VERSION='0.0.3' \
    OBFS_SHA256='350c22e138202868d5726cb55e3d71e9962aad3306988a9f746b80d0e8998a75' \
    LIBCORK_COMMIT='3bcb832' \
    LIBCORK_SHA256='c229dbfe754ec376f906043a35b1fb10725226c6c7136911cf817c92fc170959'

RUN set -ex; \
    mkdir -p /usr/local/src/; \
    cd /usr/local/src/; \
    # build deps
    apk add --no-cache --virtual .build-deps \
        build-base \
        autoconf \
        automake \
        libtool \
        linux-headers \
        curl \
    ; \
    # runtime deps
    apk add --no-cache --virtual .runtime-deps \
        mbedtls-dev \
        libev-dev \
        libsodium-dev \
        udns-dev \
        pcre-dev \
    ; \
    # ss-libev
    curl -LO https://github.com/shadowsocks/shadowsocks-libev/releases/download/v${SS_VERSION}/shadowsocks-libev-${SS_VERSION}.tar.gz; \
    echo "${SS_SHA256}  shadowsocks-libev-${SS_VERSION}.tar.gz" | sha256sum -c; \
    tar xzf shadowsocks-libev-${SS_VERSION}.tar.gz; \
    cd shadowsocks-libev-${SS_VERSION}; \
    ./configure --disable-documentation; \
    make; \
    #make check; \
    make install; \
    #make installcheck; \
    cd ..; \
    rm -f shadowsocks-libev-${SS_VERSION}.tar.gz; \
    rm -rf shadowsocks-libev-${SS_VERSION}; \
    # simple-obfs
    curl -LO https://github.com/shadowsocks/simple-obfs/archive/v${OBFS_VERSION}.tar.gz; \
    echo "${OBFS_SHA256}  v${OBFS_VERSION}.tar.gz" | sha256sum -c -; \
    tar xzf v${OBFS_VERSION}.tar.gz; \
    cd simple-obfs-${OBFS_VERSION}; \
    # libcork
    rm -rf libcork; \
    curl -LO https://github.com/shadowsocks/libcork/archive/${LIBCORK_COMMIT}.tar.gz; \
    echo "${LIBCORK_SHA256}  ${LIBCORK_COMMIT}.tar.gz" | sha256sum -c -; \
    tar xzf ${LIBCORK_COMMIT}.tar.gz; \
    mv libcork-* libcork; \
    ./autogen.sh; \
    ./configure --disable-documentation; \
    make; \
    #make check; \
    make install; \
    #make installcheck; \
    cd ..; \
    rm -f simple-obfs-${OBFS_VERSION}.tar.gz; \
    rm -rf simple-obfs-${OBFS_VERSION}; \
    # cleaning
    apk del .build-deps; \
    rm -rf /tmp/* /var/tmp/* /var/cache/apk/* /usr/local/src/*;

# Set password in server.json
# Set password, server ip and port in client.json
COPY server.json client.json /etc/shadowsocks-libev/
COPY docker-entrypoint.sh /usr/local/bin/

RUN set -ex; \
    chmod +x /usr/local/bin/docker-entrypoint.sh; \
    # User/Group
    addgroup -S shadowsocks; \
    adduser -S -G shadowsocks shadowsocks;

ENTRYPOINT [ "docker-entrypoint.sh" ]
EXPOSE 12345/tcp 12345/udp
CMD [ "ss-server" ]