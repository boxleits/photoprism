FROM photoprism/development:20210602 as build

# Set up project directory
WORKDIR "/go/src/github.com/photoprism/photoprism"
COPY . .

# Build PhotoPrism
RUN make dep build-js install

# Same base image as photoprism/development
FROM ubuntu:21.04

# Set environment variables
# see https://docs.photoprism.org/getting-started/config-options/
ENV DEBIAN_FRONTEND="noninteractive" \
    TF_CPP_MIN_LOG_LEVEL="2" \
    PATH="/photoprism/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" \
    TMPDIR="/tmp" \
    PHOTOPRISM_ASSETS_PATH="/photoprism/assets" \
    PHOTOPRISM_STORAGE_PATH="/photoprism/storage" \
    PHOTOPRISM_BACKUP_PATH="/var/lib/photoprism" \
    PHOTOPRISM_ORIGINALS_PATH="/photoprism/originals" \
    PHOTOPRISM_IMPORT_PATH="/photoprism/import" \
    PHOTOPRISM_LOG_FILENAME="/photoprism/photoprism.log" \
    PHOTOPRISM_PID_FILENAME="/photoprism/photoprism.pid" \
    PHOTOPRISM_DEBUG="false" \
    PHOTOPRISM_PUBLIC="false" \
    PHOTOPRISM_READONLY="false" \
    PHOTOPRISM_UPLOAD_NSFW="true" \
    PHOTOPRISM_DETECT_NSFW="false" \
    PHOTOPRISM_EXPERIMENTAL="false" \
    PHOTOPRISM_SITE_URL="http://localhost:2342/" \
    PHOTOPRISM_SITE_TITLE="PhotoPrism" \
    PHOTOPRISM_SITE_CAPTION="Browse Your Life" \
    PHOTOPRISM_SITE_DESCRIPTION="" \
    PHOTOPRISM_SITE_AUTHOR="" \
    PHOTOPRISM_HTTP_HOST="0.0.0.0" \
    PHOTOPRISM_HTTP_PORT=2342 \
    PHOTOPRISM_DATABASE_DRIVER="sqlite" \
    PHOTOPRISM_DATABASE_SERVER="" \
    PHOTOPRISM_DATABASE_NAME="photoprism" \
    PHOTOPRISM_DATABASE_USER="photoprism" \
    PHOTOPRISM_DATABASE_PASSWORD="" \
    PHOTOPRISM_DISABLE_WEBDAV="false" \
    PHOTOPRISM_DISABLE_SETTINGS="false" \
    PHOTOPRISM_DISABLE_BACKUPS="false" \
    PHOTOPRISM_DISABLE_EXIFTOOL="false" \
    PHOTOPRISM_DISABLE_PLACES="false" \
    PHOTOPRISM_DISABLE_TENSORFLOW="false" \
    PHOTOPRISM_DARKTABLE_PRESETS="false" \
    PHOTOPRISM_THUMB_FILTER="lanczos" \
    PHOTOPRISM_THUMB_UNCACHED="false" \
    PHOTOPRISM_THUMB_SIZE=2048 \
    PHOTOPRISM_THUMB_SIZE_UNCACHED=7680 \
    PHOTOPRISM_JPEG_SIZE=7680 \
    PHOTOPRISM_JPEG_QUALITY=92 \
    PHOTOPRISM_WORKERS=0 \
    PHOTOPRISM_WAKEUP_INTERVAL=900 \
    PHOTOPRISM_AUTO_INDEX=300 \
    PHOTOPRISM_AUTO_IMPORT=300

# Configure apt-get
RUN echo 'Acquire::Retries "10";' > /etc/apt/apt.conf.d/80retry && \
    echo 'APT::Install-Recommends "false";' > /etc/apt/apt.conf.d/80recommends && \
    echo 'APT::Install-Suggests "false";' > /etc/apt/apt.conf.d/80suggests && \
    echo 'APT::Get::Assume-Yes "true";' > /etc/apt/apt.conf.d/80forceyes && \
    echo 'APT::Get::Fix-Missing "true";' > /etc/apt/apt.conf.d/80fixmissin

# Install additional distribution packages
RUN apt-get update && apt-get dist-upgrade && apt-get install -y --no-install-recommends \
        gpgv \
        wget \
        curl \
        davfs2 \
        ca-certificates \
        mariadb-client \
        sqlite3 \
        tzdata \
        libheif-examples \
        gnupg \
        gpg-agent \
        apt-utils \
        add-apt-key \
        exiftool \
        rawtherapee \
        ffmpeg \
        ffmpegthumbnailer \
        libavcodec-extra  && \
        [ "$TARGETARCH" = "arm" ] || apt-get install darktable

# Remove files to reduce image size
RUN apt-get autoremove && apt-get clean && rm -rf /var/lib/apt/lists/*

# Update Exiftool, see https://ubuntu.com/security/CVE-2021-22204
RUN wget "http://launchpadlibrarian.net/536122424/libimage-exiftool-perl_12.16+dfsg-2_all.deb" -O /tmp/libimage-exiftool-perl.deb && \
    dpkg -i /tmp/libimage-exiftool-perl.deb && \
    rm /tmp/libimage-exiftool-perl.deb

# Copy dependencies
COPY --from=build /go/bin/gosu /bin/gosu
COPY --from=build /usr/lib/libtensorflow.so /usr/lib/libtensorflow.so
COPY --from=build /usr/lib/libtensorflow_framework.so /usr/lib/libtensorflow_framework.so
RUN ldconfig

# Set default umask and create photoprism user
RUN umask 0000 && useradd photoprism -m -d /photoprism && chmod a+rwx /photoprism
WORKDIR /photoprism

# Copy files to /photoprism
COPY --from=build /root/.local/bin/photoprism /photoprism/bin/photoprism
COPY --from=build /root/.photoprism/assets /photoprism/assets
COPY --chown=root:root --chmod=755 /docker/scripts/heif-convert.sh /usr/local/bin/heif-convert

# Create directories
RUN mkdir -m 777 -p \
    /var/lib/photoprism \
    /tmp/photoprism \
    /photoprism/originals \
    /photoprism/import \
    /photoprism/storage/config \
    /photoprism/storage/cache && \
    chown -Rf photoprism:photoprism /photoprism /var/lib/photoprism /tmp/photoprism && \
    chmod -Rf a+rw /photoprism /var/lib/photoprism /tmp/photoprism && \
    find /photoprism -type d -print0 | xargs -0 chmod 777

# Show photoprism version
RUN photoprism -v

# Expose http port
EXPOSE 2342

# Configure entrypoint
COPY --chown=root:root --chmod=755 /docker/scripts/entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
VOLUME /var/lib/photoprism

# Run server
CMD ["photoprism", "start"]
