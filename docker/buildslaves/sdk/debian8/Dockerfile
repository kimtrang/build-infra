# Docker container for Debian 8

FROM debian:8.2
MAINTAINER build-team@couchbase.com

USER root

# Install Couchbase build dependencies (jessie-backports needed for OpenJDK)
RUN apt-get update && \
    apt-get install -y build-essential curl git libssl-dev lsb-release sudo tar && \
    apt-get install -y groff-base python-dev python-pip python-virtualenv && \
    apt-get install -y libuv0.10-dev libev-dev libevent-dev libbz2-dev libffi-dev && \
    apt-get install -y libunwind8 libicu52 libssl1.0.0 && \
    apt-get clean

# Create couchbase user with password-less sudo privs, and give
# ownership of /opt/couchbase
RUN useradd couchbase -G sudo -m -s /bin/bash && \
    mkdir -p /opt/couchbase && chown -R couchbase:couchbase /opt/couchbase && \
    echo 'couchbase:couchbase' | chpasswd && \
    sed -ri 's/ALL\) ALL/ALL) NOPASSWD:ALL/' /etc/sudoers

# Update locale
RUN apt-get update && \
    apt-get install -y locales && \
    apt-get clean && \
    locale-gen en_US.UTF-8
ENV LANG=en_US.UTF-8

# Repo
RUN curl https://storage.googleapis.com/git-repo-downloads/repo \
    -o /usr/local/bin/repo && \
    chmod a+x /usr/local/bin/repo

# CMake
RUN mkdir /tmp/deploy && \
    curl -L https://cmake.org/files/v3.9/cmake-3.9.4-Linux-x86_64.sh -o /tmp/deploy/cmake.sh && \
    (echo y ; echo n) | sh /tmp/deploy/cmake.sh --prefix=/usr/local && \
    rm /usr/local/bin/cmake-gui && \
    rm -rf /tmp/deploy

# Oracle JDK (for Jenkins).
RUN mkdir /tmp/deploy && \
    cd /tmp/deploy && \
    curl -L --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" \
    http://download.oracle.com/otn-pub/java/jdk/8u131-b11/d54c1d3a095b4ff2b6607d096fa80163/jdk-8u131-linux-x64.tar.gz -o jdk.tgz && \
    cd /usr/local && \
    tar xvzf /tmp/deploy/jdk.tgz && \
    ln -s jdk* java && \
    for file in /usr/local/java/bin/*; do ln -s $file /usr/local/bin; done && \
    rm -rf /tmp/deploy
ENV JAVA_HOME=/usr/local/java

# Jenkins Swarm client jar
RUN curl https://repo.jenkins-ci.org/releases/org/jenkins-ci/plugins/swarm-client/3.8/swarm-client-3.8.jar -o /usr/local/lib/swarm-client.jar

# Ensure /opt/cbdeps is writable
RUN mkdir /opt/cbdeps && \
    chown couchbase:couchbase /opt/cbdeps && \
    chmod 755 /opt/cbdeps

# libcouchbase
RUN mkdir /tmp/deploy && \
    cd /tmp/deploy && \
    curl -L -o couchbase-release.deb http://packages.couchbase.com/releases/couchbase-release/couchbase-release-1.0-6-amd64.deb && \
    dpkg -i couchbase-release.deb && \
    apt-get update && \
    apt-get install -y libcouchbase-dev libcouchbase2-libevent libcouchbase2-libev libcouchbase2-bin && \
    cd /tmp && \
    rm -rf /tmp/deploy && \
    apt-get clean

# Temp install step for "cbdep" tool
ARG CACHEBUST=1
RUN curl http://nas-n.mgt.couchbase.com/builds/downloads/cbdep/cbdep.linux -o /usr/local/bin/cbdep && chmod 755 /usr/local/bin/cbdep

# Run our builder startup script
COPY build/couchbuilder_start.sh /usr/sbin/
ENTRYPOINT [ "/usr/sbin/couchbuilder_start.sh" ]
CMD [ "swarm" ]

# Ensure appropriate directories exist and are owned by 'couchbase'
USER couchbase
RUN mkdir /home/couchbase/.ssh

# Keep this stuff at the end, because the ARG declaration breaks Docker build caching
ARG CONTAINER_TAG_ARG
ENV CONTAINER_TAG=${CONTAINER_TAG_ARG}
