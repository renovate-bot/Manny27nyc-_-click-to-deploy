FROM gcr.io/cloud-marketplace-tools/testrunner:0.1.8

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    gettext \
    jq \
    uuid-runtime \
    curl \
    wget \
    && rm -rf /var/lib/apt/lists/*

RUN curl -L -o /bin/kubectl \
    https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubectl \
      && chmod 755 /bin/kubectl

COPY tests/basic-suite.yaml /tests/basic-suite.yaml
COPY tester.sh /tester.sh

WORKDIR /
ENTRYPOINT ["/tester.sh"]
