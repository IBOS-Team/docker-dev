FROM alpine:3.7
MAINTAINER banyan.cheung@gmail.com
# persistent / runtime deps
ENV PHPIZE_DEPS \
        autoconf \
        dpkg-dev dpkg \
        file \
        g++ \
        gcc \
        libc-dev \
        make \
        pcre-dev \
        pkgconf \
        re2c

# 备份原来的repo，切换使用国内阿里云的镜像。
RUN cp /etc/apk/repositories /etc/apk/repositories.bak && echo "http://mirrors.aliyun.com/alpine/v3.7/main/" > /etc/apk/repositories

# 添加phpearth的repo用来添加一些常用但比较难装的php扩展。如swoole、redis、imagick等。详见https://php.earth/docs/linux/alpine
RUN apk add --no-cache wget ca-certificates \
&& wget -O /etc/apk/keys/phpearth.rsa.pub https://repos.php.earth/alpine/phpearth.rsa.pub \
&& echo "https://repos.php.earth/alpine/v3.7" >> /etc/apk/repositories

# 构建环境时时所需的包，这些包会在构建完成时删除
RUN apk add --no-cache --virtual .build-deps \
        ca-certificates \
        curl \
        tar \
        xz \
# https://github.com/docker-library/php/issues/494
        libressl \
        $PHPIZE_DEPS \
        coreutils \
        curl-dev \
        libedit-dev \
        libressl-dev \
        libxml2-dev \
        sqlite-dev \
        freetype \
        libpng \
        libjpeg-turbo \
        freetype-dev \
        libpng-dev \
        libjpeg-turbo-dev \
        gettext-dev \
        imagemagick \
        imap-dev \
        libmcrypt-dev \
        libxslt-dev

# 需要保留的包。
RUN apk add --no-cache --virtual .fetch-deps \
        bash \
        tzdata

# 确保 www-data 用户存在
RUN set -x \
    && addgroup -g 82 -S www-data \
    && adduser -u 82 -D -S -G www-data www-data

ENV PHP_INI_DIR /usr/local/etc/php
RUN mkdir -p $PHP_INI_DIR/conf.d

ENV PHP_EXTRA_CONFIGURE_ARGS --enable-fpm --with-fpm-user=www-data --with-fpm-group=www-data

ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2"
ENV PHP_CPPFLAGS="$PHP_CFLAGS"
ENV PHP_LDFLAGS="-Wl,-O1 -Wl,--hash-style=both -pie"

# 下载并验证php源码包。
ENV PHP_URL="https://oss.ibos.cn/docker/resource/php/php-7.2.0.tar.xz"

RUN set -xe; \
    mkdir -p /ibos/log/php-fpm; \
    mkdir -p /ibos/www; \
    mkdir -p /usr/src; \
    cd /usr/src; \
    wget -O php.tar.xz "$PHP_URL"; 

RUN set -xe; \
    mkdir -p /usr/src/php; \
    tar -Jxf /usr/src/php.tar.xz -C /usr/src/php --strip-components=1;

RUN set -xe \
    && export CFLAGS="$PHP_CFLAGS" \
        CPPFLAGS="$PHP_CPPFLAGS" \
        LDFLAGS="$PHP_LDFLAGS" \
    && cd /usr/src/php \
    && gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
    && ./configure \
        --build="$gnuArch" \
        --with-config-file-path="$PHP_INI_DIR" \
        --with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
        \
        --disable-cgi \
        \
# --enable-ftp is included here because ftp_ssl_connect() needs ftp to be compiled statically (see https://github.com/docker-library/php/issues/236)
        --enable-ftp \
# --enable-mbstring is included here because otherwise there's no way to get pecl to use it properly (see https://github.com/docker-library/php/issues/195)
        --enable-mbstring \
# --enable-mysqlnd is included here because it's harder to compile after the fact than extensions are (since it's a plugin for several extensions, not an extension in itself)
        --enable-mysqlnd \
        \
        --with-curl \
        --with-libedit \
        --with-openssl \
        --with-zlib \
        --with-imap \
        --with-imap-ssl \
        --with-mhash \
        --with-mcrypt \
        --with-pdo-mysql \
        --with-mysqli \
        --with-gettext \
        --with-xmlrpc \
        --with-xsl \
        --with-gd \
        --with-freetype-dir=/usr/include/ \
        --with-png-dir=/usr/include/ \
        --with-jpeg-dir=/usr/include/ \
        --enable-zip \
        --enable-exif \
        --enable-bcmath \
        --enable-calendar \
        --enable-soap \
        --enable-sockets \
        --enable-sysvsem \
        --enable-sysvshm \
        --enable-pcntl \
        --enable-opcache \
        --enable-inline-optimization \
        --disable-rpath \ 
        \
# bundled pcre is too old for s390x (which isn't exactly a good sign)
# /usr/src/php/ext/pcre/pcrelib/pcre_jit_compile.c:65:2: error: #error Unsupported architecture
        --with-pcre-regex=/usr \
        \
        $PHP_EXTRA_CONFIGURE_ARGS \
    && make -j "$(nproc)" \
    && make install \
    && { find /usr/local/bin /usr/local/sbin -type f -perm +0111 -exec strip --strip-all '{}' + || true; } \
    && make clean \
    && cd / \
    && rm -rf /usr/src/php \
    \
    && runDeps="$( \
        scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
            | tr ',' '\n' \
            | sort -u \
            | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
    )" \
    && apk add --no-cache --virtual .php-rundeps $runDeps \
# https://github.com/docker-library/php/issues/443
    && pecl update-channels 

WORKDIR /ibos/www/

RUN set -ex \
    && cd /usr/local/etc \
    && if [ -d php-fpm.d ]; then \
        # for some reason, upstream's php-fpm.conf.default has "include=NONE/etc/php-fpm.d/*.conf"
        sed 's!=NONE/!=!g' php-fpm.conf.default | tee php-fpm.conf > /dev/null; \
    else \
        # PHP 5.x doesn't use "include=" by default, so we'll create our own simple config that mimics PHP 7+ for consistency
        mkdir php-fpm.d; \
        cp php-fpm.conf.default php-fpm.d/www.conf; \
        { \
            echo '[global]'; \
            echo 'include=etc/php-fpm.d/*.conf'; \
        } | tee php-fpm.conf; \
    fi

# 安装其他额外的扩展包。
RUN apk add --no-cache php7.2-swoole php7.2-redis php7.2-memcached php7.2-imagick php7.2-xdebug

# 清理安装环境
RUN apk del .build-deps && rm -rf /tmp/pear ~/.pearrc

# 安装 tzdata 包,解决TimeZone问题
ENV TIME_ZONE Asia/Shanghai
RUN echo "${TIME_ZONE}" > /etc/timezone \ 
&& ln -sf /usr/share/zoneinfo/${TIME_ZONE} /etc/localtime

# for swoole
EXPOSE 9501

CMD ["php-fpm"]