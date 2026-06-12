FROM almalinux:9

RUN dnf install -y --allowerasing \
    curl \
    wget \
    bind-utils \
    iputils \
    iproute \
    net-tools \
    nmap-ncat \
    traceroute \
    procps-ng \
    jq \
    vim \
    less \
 && dnf clean all

CMD ["/bin/bash"]