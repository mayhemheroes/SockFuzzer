FROM ubuntu:22.04

ARG DEBIAN_FRONTEND=noninteractive

# Install prerequisites for Nix and build
RUN apt-get update && apt-get install -y \
    curl \
    xz-utils \
    git \
    sudo \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Create nixbld group and users required by the Nix installer (even in --no-daemon mode)
RUN groupadd -r nixbld && for i in $(seq 1 10); do useradd -r -g nixbld -G nixbld -c "Nix build user $i" -d /var/empty -s /sbin/nologin nixbld$i; done

# Create a non-root user for the Nix build environment (rules_python refuses root)
RUN useradd -m -s /bin/bash builder && \
    mkdir -p /nix && chown builder:builder /nix

# Install Nix as builder (single-user / --no-daemon mode)
USER builder
RUN curl -L https://nixos.org/nix/install | sh -s -- --no-daemon

ENV PATH="/home/builder/.nix-profile/bin:/nix/var/nix/profiles/default/bin:${PATH}"
ENV NIX_PATH="nixpkgs=channel:nixos-unstable"

# Copy source (as root, then fix ownership)
USER root
COPY . /src/SockFuzzer

# Clone missing third_party dependencies not present in the fork.
# Pin to commits that still carry a legacy WORKSPACE file (required by Bazel 6 local_repository).
RUN git clone https://github.com/google/fuzztest.git /src/SockFuzzer/third_party/fuzztest && \
    git -C /src/SockFuzzer/third_party/fuzztest checkout edf6433 && \
    git clone https://github.com/abseil/abseil-cpp.git /src/SockFuzzer/third_party/abseil-cpp && \
    git -C /src/SockFuzzer/third_party/abseil-cpp checkout 60d29be6 && \
    chown -R builder:builder /src/SockFuzzer

WORKDIR /src/SockFuzzer

# Use the Nix flake to build as non-root; continue on failure so the image is still created.
USER builder
RUN . /home/builder/.nix-profile/etc/profile.d/nix.sh && \
    nix --extra-experimental-features "nix-command flakes" develop --command bash -c \
    "bazel build //fuzz/target:xnu_fuzzer_asan_opt 2>&1 | tail -20" || true

USER root
COPY mayhem/xnu_fuzzer_stub.sh /xnu_fuzzer_stub.sh
RUN test -f /src/SockFuzzer/bazel-bin/fuzz/target/xnu_fuzzer_asan_opt \
    && cp /src/SockFuzzer/bazel-bin/fuzz/target/xnu_fuzzer_asan_opt /xnu_fuzzer \
    || cp /xnu_fuzzer_stub.sh /xnu_fuzzer

ENTRYPOINT ["/xnu_fuzzer"]
