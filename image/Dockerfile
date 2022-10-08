FROM alpine:3.16 as build

ARG BOOST_VERSION="1.80.0"
ARG LIBTORRENT_VERSION="2.0.7"
ARG QBITTORRENT_VERSION="4.4.5"
ARG LIBTORRENT_RELEASE_TYPE="Release"
ARG QBITTORRENT_RELEASE_TYPE="release"

# Build dependencies
RUN apk add autoconf automake build-base cmake curl git libtool linux-headers perl pkgconf python3 python3-dev re2c tar
# Application dependencies
RUN apk add icu-dev libexecinfo-dev gnutls-dev qt6-qtbase-dev qt6-qttools-dev zlib-dev qt6-qtsvg-dev

# Add dedicated build user
#RUN adduser -S -D -h /build -s /sbin/nologin build-user

#USER build-user

# Build Ninja
RUN git clone --shallow-submodules --recurse-submodules https://github.com/ninja-build/ninja.git "$HOME/ninja" \
    && cd "$HOME/ninja" \
    && git checkout "$(git tag -l --sort=-v:refname "v*" | head -n 1)" \
    && cmake -Wno-dev -B build \
	-D CMAKE_CXX_STANDARD=17 \
	-D CMAKE_INSTALL_PREFIX="/usr/local" \
    && cmake --build build \
    && cmake --install build

# Get Boost build files
RUN curl -L https://boostorg.jfrog.io/artifactory/main/release/${BOOST_VERSION}/source/boost_1_80_0.tar.gz -o "$HOME/boost_${BOOST_VERSION//./_}.tar.gz" \
    && tar xvf "$HOME/boost_${BOOST_VERSION//./_}.tar.gz" -C "$HOME"

# Build libtorrent
RUN git clone --shallow-submodules --recurse-submodules https://github.com/arvidn/libtorrent.git "$HOME/libtorrent" \
    && cd "$HOME/libtorrent" \
    && git checkout "v${LIBTORRENT_VERSION}" \
    && cmake -Wno-dev -G Ninja -B build \
    -D CMAKE_BUILD_TYPE="${LIBTORRENT_RELEASE_TYPE}" \
    -D CMAKE_CXX_STANDARD=17 \
    -D BOOST_INCLUDEDIR="$HOME/boost_${BOOST_VERSION//./_}/" \
    -D CMAKE_INSTALL_LIBDIR="lib" \
    -D CMAKE_INSTALL_PREFIX="/usr/local" \
    -D BUILD_SHARED_LIBS=OFF \
    -D static_runtime=ON \
    -D gnutls=ON \
    && cmake --build build \
    && cmake --install build
    
# Build qBittorrent
RUN git clone --shallow-submodules --recurse-submodules https://github.com/qbittorrent/qBittorrent.git "$HOME/qbittorrent" \
    && cd "$HOME/qbittorrent" \
    && git checkout "release-${QBITTORRENT_VERSION}" \
    && cmake -Wno-dev -G Ninja -B build \
    -D CMAKE_BUILD_TYPE="${QBITTORRENT_RELEASE_TYPE}" \
    -D CMAKE_CXX_STANDARD=17 \
    -D OPENSSL_ROOT_DIR="/lib" \
    -D BOOST_INCLUDEDIR="$HOME/boost_${BOOST_VERSION//./_}/" \
    -D CMAKE_CXX_STANDARD_LIBRARIES="/usr/lib/libexecinfo.so" \
    -D CMAKE_INSTALL_PREFIX="/usr/local" \
    -D QT6=ON \
    -D GUI=OFF \
    -D DBUS=OFF \
    -D SYSTEMD=OFF \
    && cmake --build build \
    && mv "$HOME/qbittorrent/build/qbittorrent-nox" /usr/local/bin/

FROM alpine:3.16

ARG S6_OVERLAY_VERSION="3.1.2.1"

ENV S6_BEHAVIOUR_IF_STAGE2_FAILS="2"
ENV QBITTORRENT_FOLDER="/var/lib/qbittorrent"
ENV HOME="${QBITTORRENT_FOLDER}"

COPY --from=build /usr/local/bin/qbittorrent-nox /usr/local/bin/qbittorrent-nox

RUN apk add --no-cache bash curl gettext qt6-qtbase gnutls libexecinfo python3 tinyssh tar netcat-openbsd openssh-sftp-server ucspi-tcp6 xz \
    && addgroup -S qbittorrent \
    && adduser -S -D -h /var/lib/qbittorrent -s /bin/bash -G qbittorrent -g qbittorrent qbittorrent \
    && curl -OL --output-dir /tmp https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-noarch.tar.xz \
    && tar -C / -Jxpf /tmp/s6-overlay-noarch.tar.xz \
    && rm /tmp/s6-overlay-noarch.tar.xz \
    && curl -OL --output-dir /tmp https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-x86_64.tar.xz \
    && tar -C / -Jxpf /tmp/s6-overlay-x86_64.tar.xz \
    && rm /tmp/s6-overlay-x86_64.tar.xz \
    && apk del curl tar xz

COPY ./root /

WORKDIR /var/lib/qbittorrent
ENTRYPOINT ["/init"]
HEALTHCHECK --interval=30s --timeout=30s CMD nc -z 127.0.0.1 8080 || exit 1