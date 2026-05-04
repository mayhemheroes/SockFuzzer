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

# Install Nix (single-user mode)
RUN curl -L https://nixos.org/nix/install | sh -s -- --no-daemon

ENV PATH="/root/.nix-profile/bin:/nix/var/nix/profiles/default/bin:${PATH}"
ENV NIX_PATH="nixpkgs=channel:nixos-unstable"

# Copy source
COPY . /src/SockFuzzer
WORKDIR /src/SockFuzzer

# Use the Nix flake to build
RUN . /root/.nix-profile/etc/profile.d/nix.sh && \
    nix --extra-experimental-features "nix-command flakes" develop --command bash -c \
    "bazel build //fuzz/target:xnu_fuzzer_asan_opt 2>&1 | tail -20" && \
    cp bazel-bin/fuzz/target/xnu_fuzzer_asan_opt /xnu_fuzzer

ENTRYPOINT ["/xnu_fuzzer"]
