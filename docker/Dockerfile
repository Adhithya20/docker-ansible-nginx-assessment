FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y \
    openssh-server \
    sudo \
    curl \
    git \
    iputils-ping \
    nginx \
    python3 \
    && \
    rm -rf /var/lib/apt/lists/*

RUN mkdir -p /run/sshd

CMD ["/usr/sbin/sshd", "-D"]
