FROM adoptopenjdk/openjdk11:ubi-slim
MAINTAINER Raymond Ploski <https://github.com/rayploski/docker-consul-openjdk-11>

# This is the release of Consul to pull in.
ENV CONSUL_VERSION=1.7.2

# This is the location of the releases.
ENV HASHICORP_RELEASES=https://releases.hashicorp.com

# Create a consul user and group first so the IDs get set the same way, even as
# the rest of this may change over time.
RUN groupadd consul && \
    useradd -r -g consul consul

# Set up certificates, base tools, and Consul.
# libc6-compat is needed to symlink the shared libraries for ARM builds
RUN dnf update && \
    dnf install curl ca-certificates wget iputils unzip -y && \
    dnf clean all &&\
    mkdir -p /tmp/build && \
    cd /tmp/build && \
    wget  https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64 &&\
    mv ./jq-linux64 /usr/local/bin/jq &&\
    chmod a+x /usr/local/bin/jq &&\
    sArch="$(uname -m)" && \
    case "${sArch}" in \
        x86_64) consulArch='amd64' ;; \
        x86) consulArch='386' ;; \
        armhf) consulArch='armhfv6' ;; \
        armv71) consulArch='armhfv6' ;; \
        *) echo >&2 "error: unsupported architecture: ${sArch} (see ${HASHICORP_RELEASES}/consul/${CONSUL_VERSION}/)" && exit 1 ;; \
    esac && \
    wget ${HASHICORP_RELEASES}/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_linux_${consulArch}.zip && \
    wget ${HASHICORP_RELEASES}/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_SHA256SUMS && \
    wget ${HASHICORP_RELEASES}/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_SHA256SUMS.sig && \
    grep consul_${CONSUL_VERSION}_linux_${consulArch}.zip consul_${CONSUL_VERSION}_SHA256SUMS | sha256sum -c && \
    unzip consul_${CONSUL_VERSION}_linux_${consulArch}.zip && \
    mv ./consul /usr/local/bin/consul &&\
    chmod a+x /usr/local/bin/consul &&\
    rm -rf /tmp/build &&\

# tiny smoke test to ensure the binary we downloaded runs
    consul version

# The /consul/data dir is used by Consul to store state. The agent will be started
# with /consul/config as the configuration directory so you can add additional
# config files in that location.
RUN mkdir -p /consul/data && \
    mkdir -p /consul/config && \
    chown -R consul:consul /consul

# set up nsswitch.conf for Go's "netgo" implementation which is used by Consul,
# otherwise DNS supercedes the container's hosts file, which we don't want.
RUN test -e /etc/nsswitch.conf || echo 'hosts: files dns' > /etc/nsswitch.conf

# Expose the consul data directory as a volume since there's mutable state in there.
VOLUME /consul/data

# Server RPC is used for communication between Consul clients and servers for internal
# request forwarding.
EXPOSE 8300

# Serf LAN and WAN (WAN is used only by Consul servers) are used for gossip between
# Consul agents. LAN is within the datacenter and WAN is between just the Consul
# servers in all datacenters.
EXPOSE 8301 8301/udp 8302 8302/udp

# HTTP and DNS (both TCP and UDP) are the primary interfaces that applications
# use to interact with Consul.
EXPOSE 8500 8600 8600/udp

# Consul doesn't need root privileges so we run it as the consul user from the
# entry point script. The entry point script also uses dumb-init as the top-level
# process to reap any zombie processes created by Consul sub-processes.
COPY docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["docker-entrypoint.sh"]

# By default you'll get an insecure single-node development server that stores
# everything in RAM, exposes a web UI and HTTP endpoints, and bootstraps itself.
# Don't use this configuration for production.
CMD ["agent", "-dev", "-client", "0.0.0.0"]
