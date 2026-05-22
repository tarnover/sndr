FROM alpine:latest
LABEL org.opencontainers.image.source="https://github.com/tarnover/sndr"
LABEL org.opencontainers.image.description="sndr - command-line Send client"
LABEL org.opencontainers.image.licenses="GPL-3.0-only"

# Expects a statically-linked sndr binary to be present alongside this
# Dockerfile at build time. Build with:
#   cargo build --release -j2
#   cp target/release/sndr pkg/docker/sndr
#   docker build -t sndr pkg/docker
COPY ./sndr /

WORKDIR /data/
ENTRYPOINT ["/sndr"]
