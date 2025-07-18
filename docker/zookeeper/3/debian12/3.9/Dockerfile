FROM marketplace.gcr.io/google/debian12 as ospo

# Download Licenses and restricted source-code
COPY components.csv /components.csv
COPY source_code.txt /source_code.txt

RUN apt update && apt -y install curl ca-certificates

RUN curl -o /download-licenses.sh -L https://raw.githubusercontent.com/GoogleCloudPlatform/click-to-deploy/master/scripts/download-licenses.sh \
        && curl -o /download-ref-repos.sh -L https://raw.githubusercontent.com/GoogleCloudPlatform/click-to-deploy/master/scripts/download-ref-repos.sh \
        && chmod +x /download-licenses.sh \
        && chmod +x /download-ref-repos.sh

RUN mkdir -p /usr/src/licenses \
        && /download-licenses.sh /components.csv /usr/src/licenses \
        && /download-ref-repos.sh /source_code.txt /usr/src

# Build Gosu
FROM golang:1.22-bookworm as build-gosu

RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        arch-test \
        file \
    ; \
    rm -rf /var/lib/apt/lists/*

# note: we cannot add "-s" here because then "govulncheck" does not work (see SECURITY.md); the ~0.2MiB increase (as of 2022-12-16, Go 1.18) is worth it
ENV BUILD_FLAGS="-v -trimpath -ldflags '-d -w'"

RUN set -eux; \
    { \
        echo '#!/usr/bin/env bash'; \
        echo 'set -Eeuo pipefail -x'; \
        echo 'eval "go build $BUILD_FLAGS -o /go/bin/gosu-$ARCH"'; \
        echo 'file "/go/bin/gosu-$ARCH"'; \
        echo 'if arch-test "$ARCH"; then'; \
# there's a fun QEMU + Go 1.18+ bug that causes our binaries (especially on ARM arches) to hang indefinitely *sometimes*, hence the "timeout" and looping here
        echo '  try() { for (( i = 0; i < 30; i++ )); do if timeout 1s "$@"; then return 0; fi; done; return 1; }'; \
        echo '  try "/go/bin/gosu-$ARCH" --version'; \
        echo '  try "/go/bin/gosu-$ARCH" nobody id'; \
        echo '  try "/go/bin/gosu-$ARCH" nobody ls -l /proc/self/fd'; \
        echo 'fi'; \
    } > /usr/local/bin/gosu-build-and-test.sh; \
    chmod +x /usr/local/bin/gosu-build-and-test.sh

# disable CGO for ALL THE THINGS (to help ensure no libc)
ENV CGO_ENABLED 0

WORKDIR /go/src/github.com/tianon/gosu

RUN mkdir -p /usr/src \
      && git clone https://github.com/tianon/gosu.git --branch="1.17" /usr/src/gosu \
      && cp /usr/src/gosu/go.mod ./ \
      && cp /usr/src/gosu/go.sum ./ \
      && go mod download \
      && cp /usr/src/gosu/*.go ./

RUN ARCH=amd64  GOARCH=amd64   gosu-build-and-test.sh

RUN set -eux; ls -lAFh /go/bin/gosu-*; file /go/bin/gosu-*


FROM marketplace.gcr.io/google/debian12

COPY --from=ospo /usr/src /usr/src

ARG ZK_VERSION=3.9.3

ENV ZOO_CONF_DIR=/conf \
    ZOO_DATA_DIR=/data \
    ZOO_DATA_LOG_DIR=/datalog \
    ZOO_LOG_DIR=/logs \
    ZOO_TICK_TIME=2000 \
    ZOO_INIT_LIMIT=5 \
    ZOO_SYNC_LIMIT=2 \
    ZOO_AUTOPURGE_PURGEINTERVAL=0 \
    ZOO_AUTOPURGE_SNAPRETAINCOUNT=3 \
    ZOO_MAX_CLIENT_CNXNS=60 \
    ZOO_STANDALONE_ENABLED=true \
    ZOO_ADMINSERVER_ENABLED=true

# Add a user with an explicit UID/GID and create necessary directories.
RUN set -eux; \
    groupadd -r zookeeper --gid=1000; \
    useradd -r -g zookeeper --uid=1000 zookeeper; \
    mkdir -p "${ZOO_DATA_LOG_DIR}" "${ZOO_DATA_DIR}" "${ZOO_CONF_DIR}" "${ZOO_LOG_DIR}"; \
    chown zookeeper:zookeeper "${ZOO_DATA_LOG_DIR}" "${ZOO_DATA_DIR}" "${ZOO_CONF_DIR}" "${ZOO_LOG_DIR}"

COPY --from=build-gosu /go/bin/gosu-amd64 /usr/sbin/gosu
RUN chmod +x /usr/sbin/gosu \
        && gosu --version \
        && gosu nobody true

# Install required packages.
RUN set -eux; \
    apt-get update; \
    DEBIAN_FRONTEND=noninteractive \
    apt-get install -y --no-install-recommends \
        default-jdk \
        ca-certificates \
        dirmngr \
        gnupg \
        netcat-traditional \
        openssl \
        wget; \
    rm -rf /var/lib/apt/lists/*

    # Verify that gosu binary works
    # gosu nobody true

ENV GPG_KEYS 3F7A1D16FA4217B1DC75E1C9FFE35B7F15DFA1BA

ARG SHORT_DISTRO_NAME="zookeeper-${ZK_VERSION}"
ARG DISTRO_NAME="apache-zookeeper-${ZK_VERSION}-bin"

ENV C2D_RELEASE="${ZK_VERSION}"

# Download Apache Zookeeper, verify its PGP signature, untar and clean up.
RUN set -eux; \
    ddist() { \
        local f="${1}"; shift; \
        local distFile="${1}"; shift; \
        local success=; \
        local distUrl=; \
        for distUrl in \
            'https://www.apache.org/dyn/closer.cgi?action=download&filename=' \
            https://www-us.apache.org/dist/ \
            https://www.apache.org/dist/ \
            https://archive.apache.org/dist/ \
        ; do \
            if wget -q -O "${f}" "$distUrl$distFile" && [ -s "${f}" ]; then \
                success=1; \
                break; \
            fi; \
        done; \
        [ -n "${success}" ]; \
    }; \
    ddist "${DISTRO_NAME}.tar.gz" "zookeeper/${SHORT_DISTRO_NAME}/${DISTRO_NAME}.tar.gz"; \
    ddist "${DISTRO_NAME}.tar.gz.asc" "zookeeper/${SHORT_DISTRO_NAME}/${DISTRO_NAME}.tar.gz.asc"; \
    export GNUPGHOME="$(mktemp -d)"; \
    gpg --keyserver ha.pool.sks-keyservers.net --recv-key ${GPG_KEYS} || \
    gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys ${GPG_KEYS} || \
    gpg --keyserver keyserver.pgp.com --recv-keys ${GPG_KEYS}; \
    gpg --batch --verify "${DISTRO_NAME}.tar.gz.asc" "${DISTRO_NAME}.tar.gz"; \
    tar -zxf "${DISTRO_NAME}.tar.gz"; \
    mv "${DISTRO_NAME}/conf/"* "${ZOO_CONF_DIR}"; \
    rm -rf "${GNUPGHOME}" "${DISTRO_NAME}.tar.gz" "${DISTRO_NAME}.tar.gz.asc"; \
    chown -R zookeeper:zookeeper "/${DISTRO_NAME}"

WORKDIR "${DISTRO_NAME}"
VOLUME ["${ZOO_DATA_DIR}", "${ZOO_DATA_LOG_DIR}", "${ZOO_LOG_DIR}"]

EXPOSE 2181 2888 3888 8080

ENV PATH="${PATH}:/${DISTRO_NAME}/bin" \
    ZOOCFGDIR="${ZOO_CONF_DIR}"

COPY docker-entrypoint.sh /

RUN chmod 775 /docker-entrypoint.sh

ENTRYPOINT ["/docker-entrypoint.sh"]

CMD ["zkServer.sh", "start-foreground"]
