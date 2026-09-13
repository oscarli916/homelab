FROM ghcr.io/anomalyco/opencode:1.14.33

RUN apk add --no-cache git openssh-client

ADD --checksum=sha256:6d1fd9253b7fe1cd4789e4951746a6f0e5a7f8075c4a5fde2987e9d88d6fe3bb \
  https://github.com/sursaone/bun-pty/releases/download/v0.4.10/librust_pty_musl.so \
  /usr/local/lib/librust_pty_musl.so

ENV BUN_PTY_LIB=/usr/local/lib/librust_pty_musl.so
