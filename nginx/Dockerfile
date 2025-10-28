FROM alpine:3.18 AS build

ARG OPENRESTY_VERSION=1.21.4.3
ENV OPENRESTY_PREFIX=/opt/openresty

# Build dependencies for compiling OpenResty with the requested modules.
RUN apk add --no-cache \
    build-base \
    curl \
    gd-dev \
    geoip-dev \
    gnu-libiconv-dev \
    linux-headers \
    openssl-dev \
    pcre-dev \
    postgresql-dev \
    readline-dev \
    zlib-dev

WORKDIR /tmp

RUN curl -fSL "https://openresty.org/download/openresty-${OPENRESTY_VERSION}.tar.gz" -o openresty.tar.gz \
    && tar -xf openresty.tar.gz \
    && rm openresty.tar.gz

WORKDIR /tmp/openresty-${OPENRESTY_VERSION}

RUN ./configure --prefix="${OPENRESTY_PREFIX}" \
        --with-pcre-jit \
        --with-ipv6 \
        --without-http_redis2_module \
        --with-http_iconv_module \
        --with-http_postgres_module \
        --with-http_stub_status_module \
        -j8 \
    && make -j"$(nproc)" \
    && make install \
    && rm -rf /tmp/openresty-"${OPENRESTY_VERSION}"


FROM alpine:3.18

ENV OPENRESTY_PREFIX=/opt/openresty \
    PATH=/opt/openresty/nginx/sbin:/opt/openresty/bin:$PATH \
    LD_PRELOAD=/usr/lib/preloadable_libiconv.so

# Runtime dependencies needed by OpenResty and compiled modules.
RUN apk add --no-cache \
    gd \
    geoip \
    gnu-libiconv \
    libgcc \
    openssl \
    pcre \
    postgresql-libs \
    zlib

COPY --from=build /opt/openresty /opt/openresty

COPY nginx.conf /opt/openresty/nginx/conf/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
