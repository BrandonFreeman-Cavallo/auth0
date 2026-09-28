FROM mcr.microsoft.com/devcontainers/go:1-1.24-bookworm

ARG TARGETARCH
ARG JUST_VERSION=1.58.0
ARG KIND_VERSION=0.27.0
ARG TILT_VERSION=0.37.7
ARG MKCERT_VERSION=1.4.4
ARG GOLANGCI_LINT_VERSION=2.4.0

USER root

RUN rm -f /etc/apt/sources.list.d/yarn.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        jq \
        libnss3-tools \
        openssl \
        tar \
    && rm -rf /var/lib/apt/lists/*

RUN set -eux; \
    case "${TARGETARCH}" in \
        amd64) \
            just_arch='x86_64'; \
            kind_arch='amd64'; \
            tilt_arch='x86_64'; \
            mkcert_arch='amd64'; \
            ;; \
        arm64) \
            just_arch='aarch64'; \
            kind_arch='arm64'; \
            tilt_arch='arm64'; \
            mkcert_arch='arm64'; \
            ;; \
        *) echo "Unsupported architecture: ${TARGETARCH}" >&2; exit 1 ;; \
    esac; \
    curl -fsSL "https://github.com/casey/just/releases/download/${JUST_VERSION}/just-${JUST_VERSION}-${just_arch}-unknown-linux-musl.tar.gz" \
        | tar -xz -C /usr/local/bin just; \
    curl -fsSL "https://github.com/kubernetes-sigs/kind/releases/download/v${KIND_VERSION}/kind-linux-${kind_arch}" \
        -o /usr/local/bin/kind; \
    curl -fsSL "https://github.com/tilt-dev/tilt/releases/download/v${TILT_VERSION}/tilt.${TILT_VERSION}.linux.${tilt_arch}.tar.gz" \
        | tar -xz -C /usr/local/bin tilt; \
    curl -fsSL "https://github.com/FiloSottile/mkcert/releases/download/v${MKCERT_VERSION}/mkcert-v${MKCERT_VERSION}-linux-${mkcert_arch}" \
        -o /usr/local/bin/mkcert; \
    rm -f /go/bin/golangci-lint; \
    curl -fsSL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh \
        | sh -s -- -b /usr/local/bin "v${GOLANGCI_LINT_VERSION}"; \
    chmod +x /usr/local/bin/kind /usr/local/bin/mkcert

RUN mkdir -p /home/vscode/go/pkg/mod /home/vscode/.cache/go-build \
    && chown -R vscode:vscode /home/vscode/go /home/vscode/.cache

USER vscode