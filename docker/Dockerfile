# syntax=docker/dockerfile:1

FROM mirror.gcr.io/library/alpine:3.22 AS build-base
# ENV RUNNER_THREADS="4"
RUN apk update && \
    apk add --no-cache \
        coreutils grep sed && \
    apk add --no-cache --virtual .build-tools \
        build-base linux-headers pkgconf \
        cmake samurai \
        mold musl musl-dev musl-libintl musl-utils && \
    apk add --no-cache --virtual .build-deps \
        curl-dev curl-static \
        pcre2-dev pcre2-static \
        rapidjson-dev toml11 yaml-cpp-dev && \
    rm -rf /var/cache/apk/*

FROM build-base AS quickjspp
WORKDIR /quickjspp/
RUN --mount=type=bind,from=src_quickjspp,source=./,target=/quickjspp/ \
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make quickjs -j 2 && \
    install -m644 -pvD \
        quickjs/libquickjs.a /emptydir/usr/lib/quickjs/ && \
    install -m644 -pvD \
        quickjs/quickjs.h quickjs/quickjs-libc.h /emptydir/usr/include/quickjs/ && \
    install -m644 -pvD \
        quickjspp.hpp /emptydir/usr/include/

FROM build-base AS libcron
WORKDIR /libcron/
RUN --mount=type=bind,from=src_libcron,source=./,target=/libcron/ \
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make libcron -j 2 && \
    install -m644 -pvD \
        libcron/out/Release/liblibcron.a /emptydir/usr/lib/ && \
    install -m644 -pvD \
        libcron/include/libcron/* /emptydir/usr/include/libcron/ && \
    install -m644 -pvD \
        libcron/externals/date/include/date/* /emptydir/usr/include/date/

FROM build-base AS subconverter-builder
COPY --link --from=quickjspp /emptydir/ /
COPY --link --from=libcron   /emptydir/ /
WORKDIR /subconverter/
RUN --mount=type=bind,from=src_subconverter,source=./,target=/subconverter/ \
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make -j 4


FROM mirror.gcr.io/library/alpine:3.22 AS final
LABEL maintainer="IceCodeNew"

RUN apk update && \
    apk add --no-cache \
        ca-certificates catatonit tzdata && \
    apk add --no-cache --virtual subconverter-deps \
        libcurl pcre2 yaml-cpp && \
    rm -rf /var/cache/apk/*

COPY --link --from=subconverter-builder --chmod=755 \
    /subconverter/subconverter /usr/bin/
COPY --link --from=src_subconverter \
    ./base/ /base/
WORKDIR /base/
ENTRYPOINT [ "catatonit", "--", "subconverter" ]
EXPOSE 25500/tcp
