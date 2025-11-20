# Copyright Broadcom, Inc. All Rights Reserved.
# SPDX-License-Identifier: APACHE-2.0

FROM docker.io/bitnami/minideb:bookworm

ARG DOWNLOADS_URL="downloads.bitnami.com/files/stacksmith"
ARG EXTRA_LOCALES
ARG TARGETARCH
ARG WITH_ALL_LOCALES="no"
ARG PG_MAJOR=14
ARG PGVECTOR_TAG=0.8.0
ARG VECTORCHORD_TAG=0.4.3
ARG PGVECTORS_TAG=0.2.0

LABEL com.vmware.cp.artifact.flavor="sha256:c50c90cfd9d12b445b011e6ad529f1ad3daea45c26d20b00732fae3cd71f6a83" \
      org.opencontainers.image.base.name="docker.io/bitnami/minideb:bookworm" \
      org.opencontainers.image.created="2025-06-27T06:48:32Z" \
      org.opencontainers.image.description="Application packaged by Broadcom, Inc. with VectorChord extensions" \
      org.opencontainers.image.documentation="https://github.com/bitnami/containers/tree/main/bitnami/postgresql/README.md" \
      org.opencontainers.image.source="https://github.com/bitnami/containers/tree/main/bitnami/postgresql" \
      org.opencontainers.image.title="postgresql" \
      org.opencontainers.image.vendor="Broadcom, Inc." \
      org.opencontainers.image.version="14.18.0-pgvector${PGVECTOR_TAG}-vectorchord${VECTORCHORD_TAG}-pgvectors${PGVECTORS_TAG}"

ENV HOME="/" \
    OS_ARCH="${TARGETARCH:-amd64}" \
    OS_FLAVOUR="debian-12" \
    OS_NAME="linux"

COPY prebuildfs /
SHELL ["/bin/bash", "-o", "errexit", "-o", "nounset", "-o", "pipefail", "-c"]
# Install required system packages and dependencies
RUN install_packages ca-certificates curl libbsd0 libbz2-1.0 libedit2 libffi8 libgcc-s1 libgmp10 libgnutls30 libhogweed6 libicu72 libidn2-0 libldap-2.5-0 liblz4-1 liblzma5 libmd0 libncurses6 libnettle8 libp11-kit0 libpcre3 libreadline8 libsasl2-2 libsqlite3-0 libssl3 libstdc++6 libtasn1-6 libtinfo6 libunistring2 libuuid1 libxml2 libxslt1.1 libzstd1 locales procps zlib1g
RUN --mount=type=secret,id=downloads_url,env=SECRET_DOWNLOADS_URL \
    DOWNLOADS_URL=${SECRET_DOWNLOADS_URL:-${DOWNLOADS_URL}} ; \
    mkdir -p /tmp/bitnami/pkg/cache/ ; cd /tmp/bitnami/pkg/cache/ || exit 1 ; \
    COMPONENTS=( \
      "postgresql-14.18.0-7-linux-${OS_ARCH}-debian-12" \
    ) ; \
    for COMPONENT in "${COMPONENTS[@]}"; do \
      if [ ! -f "${COMPONENT}.tar.gz" ]; then \
        curl -SsLf "https://${DOWNLOADS_URL}/${COMPONENT}.tar.gz" -O ; \
        curl -SsLf "https://${DOWNLOADS_URL}/${COMPONENT}.tar.gz.sha256" -O ; \
      fi ; \
      sha256sum -c "${COMPONENT}.tar.gz.sha256" ; \
      tar -zxf "${COMPONENT}.tar.gz" -C /opt/bitnami --strip-components=2 --no-same-owner ; \
      rm -rf "${COMPONENT}".tar.gz{,.sha256} ; \
    done
RUN apt-get update && apt-get upgrade -y && \
    apt-get clean && rm -rf /var/lib/apt/lists /var/cache/apt/archives
RUN chmod g+rwX /opt/bitnami
RUN find / -perm /6000 -type f -exec chmod a-s {} \; || true
RUN update-locale LANG=C.UTF-8 LC_MESSAGES=POSIX && \
    DEBIAN_FRONTEND=noninteractive dpkg-reconfigure locales && \
    echo 'en_US.UTF-8 UTF-8' >> /etc/locale.gen
# Build and install pgvector from source
RUN install_packages wget make gcc libc6-dev && \
    export PG_CONFIG=/opt/bitnami/postgresql/bin/pg_config && \
    wget -nv -O /tmp/pgvector.tar.gz "https://github.com/pgvector/pgvector/archive/refs/tags/v${PGVECTOR_TAG}.tar.gz" && \
    tar -xzf /tmp/pgvector.tar.gz -C /tmp && \
    cd /tmp/pgvector-${PGVECTOR_TAG} && \
    make && make install && \
    rm -rf /tmp/pgvector* && \
    apt-get purge -y --auto-remove make gcc libc6-dev && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
# Install VectorChord and pgvectors extensions from pre-built .deb packages
RUN install_packages wget && \
    ARCH=${TARGETARCH:-amd64} && \
    wget -nv -O /tmp/vchord.deb "https://github.com/tensorchord/VectorChord/releases/download/${VECTORCHORD_TAG}/postgresql-${PG_MAJOR}-vchord_${VECTORCHORD_TAG}-1_${ARCH}.deb" && \
    if [ -n "$PGVECTORS_TAG" ]; then \
        wget -nv -O /tmp/pgvectors.deb "https://github.com/tensorchord/pgvecto.rs/releases/download/v${PGVECTORS_TAG}/vectors-pg${PG_MAJOR}_${PGVECTORS_TAG}_${ARCH}.deb"; \
    fi && \
    mkdir -p /tmp/deb-extract && \
    cd /tmp/deb-extract && \
    dpkg-deb -x /tmp/vchord.deb . && \
    if [ -n "$PGVECTORS_TAG" ]; then \
        dpkg-deb -x /tmp/pgvectors.deb .; \
    fi && \
    find . -name "*.so" -exec cp {} /opt/bitnami/postgresql/lib/ \; && \
    find . -name "*.control" -exec cp {} /opt/bitnami/postgresql/share/extension/ \; && \
    find . -name "*.sql" -exec cp {} /opt/bitnami/postgresql/share/extension/ \; && \
    cd / && \
    rm -rf /tmp/deb-extract /tmp/vchord.deb /tmp/pgvectors.deb && \
    uninstall_packages wget

RUN uninstall_packages curl

COPY rootfs /
RUN /opt/bitnami/scripts/postgresql/postunpack.sh
RUN update-locale LANG=C.UTF-8 LC_MESSAGES=POSIX && \
    DEBIAN_FRONTEND=noninteractive dpkg-reconfigure locales && \
    echo 'en_US.UTF-8 UTF-8' >> /etc/locale.gen
RUN /opt/bitnami/scripts/locales/generate-locales.sh
# Configure PostgreSQL to preload vector extensions
RUN mkdir -p /opt/bitnami/postgresql/conf && \
    echo "shared_preload_libraries = 'vchord.so, vectors.so'" >> /opt/bitnami/postgresql/conf/postgresql.conf.sample && \
    echo "search_path = '\"\$user\", public, vectors'" >> /opt/bitnami/postgresql/conf/postgresql.conf.sample

ENV APP_VERSION="14.18.0" \
    BITNAMI_APP_NAME="postgresql" \
    LANG="en_US.UTF-8" \
    LANGUAGE="en_US:en" \
    NSS_WRAPPER_LIB="/opt/bitnami/common/lib/libnss_wrapper.so" \
    PATH="/opt/bitnami/postgresql/bin:$PATH"

VOLUME [ "/bitnami/postgresql", "/docker-entrypoint-initdb.d", "/docker-entrypoint-preinitdb.d" ]

EXPOSE 5432

USER 1001
ENTRYPOINT [ "/opt/bitnami/scripts/postgresql/entrypoint.sh" ]
CMD [ "/opt/bitnami/scripts/postgresql/run.sh" ]
