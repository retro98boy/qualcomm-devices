# docker buildx build -t retro98boy/bkerler-edl:latest .

FROM ubuntu:jammy

ARG DEBIAN_FRONTEND=noninteractive
SHELL ["/usr/bin/bash", "-c"]

# 安装语系、时区等基础包
RUN sed -i 's/archive.ubuntu.com/mirrors.ustc.edu.cn/g' /etc/apt/sources.list \
 && apt update \
 && apt upgrade -y \
 && apt install -y locales tzdata

# 设置时区和语系
RUN if [ -e /etc/localtime ]; then rm /etc/localtime; fi; ln -svf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
 && echo -e 'en_US.UTF-8 UTF-8\nzh_CN.UTF-8 UTF-8' >> /etc/locale.gen \
 && locale-gen \
 && echo 'LANG=en_US.UTF-8' > /etc/locale.conf
ENV LANG=en_US.UTF-8

# 安装bkerler/edl
RUN apt install -y usbutils adb fastboot python3-dev python3-pip liblzma-dev git \
 && cd /usr/src \
 && git clone https://github.com/bkerler/edl.git \
 && cd edl \
 && git reset --hard 334028cd887a2960867750c403dbcad9cbe8c46b \
 && git submodule update --init --recursive \
 && python3 setup.py build \
 && python3 setup.py install

# 清理
RUN apt autopurge -y && apt autoclean

RUN mkdir -p /workspace

WORKDIR /workspace
USER root
