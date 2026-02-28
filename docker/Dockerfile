FROM debian:13

ARG DEBIAN_FRONTEND=noninteractive

ENTRYPOINT ["/lib/systemd/systemd"]

STOPSIGNAL SIGRTMIN+3

RUN apt-get update \
    && apt-get -y install wget curl vim openssl git systemd openssh openvpn apparmor apparmor-easyprof apparmor-profiles apparmor-profiles-extra \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
