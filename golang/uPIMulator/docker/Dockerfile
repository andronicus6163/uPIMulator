FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt update
RUN apt install -y git
RUN apt install -y wget
RUN apt install -y cmake
RUN apt install -y ninja-build
RUN apt install -y pkg-config
RUN apt install -y python3
RUN apt install -y python3-pip

# UPMEM SDK
# NOTE: sdk-releases.upmem.com is defunct (UPMEM domain no longer resolves).
# The DPU-target clang is instead built from https://github.com/upmem/llvm-project
# branch rel_2023.2 and copied in here. See ../../../.toolchains/.
WORKDIR /root
COPY upmem-2023.2.0-Linux-x86_64 /root/upmem-2023.2.0-Linux-x86_64

WORKDIR /root/uPIMulator
