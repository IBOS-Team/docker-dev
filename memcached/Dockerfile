FROM alpine:3.7

# 备份原来的repo，切换使用国内阿里云的镜像。
RUN cp /etc/apk/repositories /etc/apk/repositories.bak && echo "http://mirrors.aliyun.com/alpine/v3.7/main/" > /etc/apk/repositories
# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN adduser -D memcache

ENV MEMCACHED_VERSION 1.5.3
ENV MEMCACHED_SHA1 9331ef99f1b7fedfe476062faed8c0ffb5528d8f

RUN set -x \
    \
    && apk add --no-cache --virtual .build-deps \
        ca-certificates \
        coreutils \
        cyrus-sasl-dev \
        dpkg-dev dpkg \
        gcc \
        libc-dev \
        libevent-dev \
        libressl \
        linux-headers \
        make \
        perl \
        perl-utils \
        tar \
    \
    && wget -O memcached.tar.gz "https://memcached.org/files/memcached-$MEMCACHED_VERSION.tar.gz" \
    && echo "$MEMCACHED_SHA1  memcached.tar.gz" | sha1sum -c - \
    && mkdir -p /usr/src/memcached \
    && tar -xzf memcached.tar.gz -C /usr/src/memcached --strip-components=1 \
    && rm memcached.tar.gz \
    \
    && cd /usr/src/memcached \
    \
    && ./configure \
        --build="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
        --enable-sasl \
    && make -j "$(nproc)" \
    \
    && make test \
    && make install \
    \
    && cd / && rm -rf /usr/src/memcached \
    \
    && runDeps="$( \
        scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
            | tr ',' '\n' \
            | sort -u \
            | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
    )" \
    && apk add --virtual .memcached-rundeps $runDeps \
    && apk del .build-deps \
    \
    && memcached -V

COPY docker-entrypoint.sh /usr/local/bin/
RUN ln -s usr/local/bin/docker-entrypoint.sh /entrypoint.sh # backwards compat
ENTRYPOINT ["docker-entrypoint.sh"]

USER memcache
EXPOSE 11211
CMD ["memcached"]