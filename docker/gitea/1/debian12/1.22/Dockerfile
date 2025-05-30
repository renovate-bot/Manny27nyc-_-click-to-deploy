FROM marketplace.gcr.io/google/debian12 as ospo

# Download Licenses and restricted source-code
COPY components.csv /components.csv
COPY source_code.txt /source_code.txt

RUN apt update && apt -y install ca-certificates curl

RUN curl -o /download-licenses.sh -L https://raw.githubusercontent.com/GoogleCloudPlatform/click-to-deploy/master/scripts/download-licenses.sh \
		&& curl -o /download-ref-repos.sh -L https://raw.githubusercontent.com/GoogleCloudPlatform/click-to-deploy/master/scripts/download-ref-repos.sh \
		&& chmod +x /download-licenses.sh \
		&& chmod +x /download-ref-repos.sh

RUN mkdir -p /usr/src/licenses \
		&& /download-licenses.sh /components.csv /usr/src/licenses \
		&& /download-ref-repos.sh /source_code.txt /usr/src

#Build stage
FROM golang:1.23-bookworm AS build-env

ARG GOPROXY
ENV GOPROXY ${GOPROXY:-direct}

ARG GITEA_VERSION=release/v1.22
ARG TAGS="sqlite sqlite_unlock_notify"
ENV TAGS "bindata timetzdata $TAGS"
ARG CGO_EXTRA_CFLAGS

# Build deps
RUN curl -fsSL https://deb.nodesource.com/setup_20.x \
    | bash - ; \
    apt update && apt -y install \
        build-essential \
        git \
        nodejs \
        && rm -rf /var/cache/apk/*

#Setup repo
RUN git clone -b ${GITEA_VERSION} https://github.com/go-gitea/gitea.git ${GOPATH}/src/code.gitea.io/gitea
#COPY . /go/src/code.gitea.io/gitea
WORKDIR ${GOPATH}/src/code.gitea.io/gitea

#Checkout version if set
RUN if [ -n "${GITEA_VERSION}" ]; then git checkout "${GITEA_VERSION}"; fi
RUN make clean-all build

# Begin env-to-ini build
RUN go build contrib/environment-to-ini/environment-to-ini.go

FROM marketplace.gcr.io/google/debian12

COPY --from=ospo /usr/src /usr/src

EXPOSE 22 3000

RUN apt-get update && \
    apt-get install -y \
        bash \
        ca-certificates \
        git \
        gettext \
        openssh-server \
        s6 \
        sqlite3 \
        gnupg

RUN ln -s /usr/lib/x86_64-linux-musl/libc.so /lib/libc.musl-x86_64.so.1 \
    &&  ssh-keygen -A \
    && mkdir -p /run/sshd

# Install latest su-exec
RUN  set -ex; \
     \
     curl -o /usr/local/bin/su-exec.c https://raw.githubusercontent.com/ncopa/su-exec/master/su-exec.c; \
     \
     fetch_deps='gcc libc-dev'; \
     apt-get update; \
     apt-get install -y --no-install-recommends $fetch_deps; \
     rm -rf /var/lib/apt/lists/*; \
     gcc -Wall \
         /usr/local/bin/su-exec.c -o/usr/local/bin/su-exec; \
     chown root:root /usr/local/bin/su-exec; \
     chmod 0755 /usr/local/bin/su-exec; \
     rm /usr/local/bin/su-exec.c; \
     \
     apt-get purge -y --auto-remove $fetch_deps

RUN addgroup \
    --system --gid 1000 \
    git && \
  adduser \
    --system --no-create-home --disabled-password \
    --home /data/git \
    --shell /bin/bash \
    --uid 1000 \
    --gid 1000 \
    git && \
  echo "git:*" | chpasswd -e

ENV USER git
ENV GITEA_CUSTOM /data/gitea
ENV C2D_RELEASE 1.22.9

VOLUME ["/data"]

COPY --from=build-env /go/src/code.gitea.io/gitea/gitea /app/gitea/gitea
COPY --from=build-env /go/src/code.gitea.io/gitea/environment-to-ini /usr/local/bin/environment-to-ini
COPY --from=build-env /go/src/code.gitea.io/gitea/docker/root /

RUN chmod 755 /usr/bin/entrypoint /app/gitea/gitea /usr/local/bin/gitea /usr/local/bin/environment-to-ini
RUN chmod 755 /etc/s6/gitea/* /etc/s6/openssh/* /etc/s6/.s6-svscan/*
ENTRYPOINT ["/usr/bin/entrypoint"]
CMD ["/usr/bin/s6-svscan", "/etc/s6"]
