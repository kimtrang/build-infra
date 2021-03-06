# Docker container for RHEL 8.0

FROM registry.access.redhat.com/ubi8/ubi:latest
MAINTAINER build-team@couchbase.com

USER root
ARG RH_USER
ARG RH_PASSWORD
ARG RH_POOL_ID

# Initialize a subscription and update the repositories
RUN subscription-manager register --username ${RH_USER} --password ${RH_PASSWORD} && \
    subscription-manager refresh && subscription-manager attach --pool=${RH_POOL_ID} && \
    yum makecache # cachebust3

# Jenkins Swarm jar
RUN dnf install -y java-1.8.0-openjdk-headless curl && \
    dnf clean packages
RUN curl \
    https://repo.jenkins-ci.org/releases/org/jenkins-ci/plugins/swarm-client/3.13/swarm-client-3.13.jar \
    -o /usr/local/lib/swarm-client.jar

# tini, to avoid zombie issues when running with Swarm jar
ENV TINI_VERSION v0.18.0
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /tini
RUN chmod +x /tini

### Install Couchbase build dependencies ######################################
# * Packages from the base CentOS repositories
RUN dnf install --setopt=keepcache=0 -y \
                autoconf \
                automake \
                ed \
                git \
                glibc-devel \
                glibc-langpack-en \
                hostname \
                libtirpc-devel \
                libtool \
                make \
                man \
                ncurses-devel \
                numactl-devel \
                openssh-clients \
                openssl-devel \
                perl-Thread-Queue \
                python2 \
                python2-devel \
                python36-devel \
                redhat-lsb-core \
                rpm-build \
                ruby rubygems rubygem-rake \
                sqlite-devel \
                sudo \
                unzip && \
    dnf clean packages && \
    ln -s /usr/bin/python2.7 /usr/bin/python2.6

# Create couchbase user with password-less sudo privs, and give
# ownership of /opt/couchbase
RUN groupadd -g1000 couchbase && \
    useradd couchbase -g couchbase -u1000 -G wheel -m -s /bin/bash && \
    mkdir /opt/couchbase && chown -R couchbase:couchbase /opt/couchbase && \
    echo 'couchbase:couchbase' | chpasswd && \
    echo '%wheel ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers.d/wheel_group && \
    echo 'Defaults:%wheel !requiretty' >> /etc/sudoers.d/wheel_group && \
    chmod 440 /etc/sudoers.d/wheel_group && \
    usermod -G wheel root

# Set default Python to Python 2
RUN alternatives --set python /usr/bin/python2

# Python 2 saves the GCC build flags from when it was built in this file,
# and uses them when building extensions. That doesn't work for us, because
# this Python 2 was built by Red Hat using GCC 8, and some of the flags
# don't work in our GCC 7.3. So I manually removed the bad ones from this
# file. This is a hack solution, but it's only necessary for building the
# python-snappy module in Alice; we removed all such Python dependencies
# from cbdeps in Mad-Hatter and don't build them ourselves anymore (they come
# in from Anaconda). So this workaround gets us going for now.
# In any case, this probably is unnecessary for 3.7.2+ and 3.6.8+.
# https://fedoraproject.org/wiki/Changes/Python_Extension_Flags
COPY _sysconfigdata.py /usr/lib64/python2.7

# GCC (from couchbasebuild/rhel-8-gcc)
COPY build/local /usr/local
RUN  (echo "/usr/local/lib64"; cat /etc/ld.so.conf) > /tmp/ld.so.conf && \
     mv /tmp/ld.so.conf /etc && ldconfig

# * CMake (from cmake.org)
RUN mkdir /tmp/deploy && \
    curl https://cmake.org/files/v3.13/cmake-3.13.0-Linux-x86_64.sh -o /tmp/deploy/cmake.sh && \
    (echo y ; echo n) | sh /tmp/deploy/cmake.sh --prefix=/usr/local && \
    rm -fr /tmp/deploy

# * ccache (from source)
RUN mkdir /tmp/deploy && \
    curl -L https://www.samba.org/ftp/ccache/ccache-3.1.9.tar.bz2 -o /tmp/deploy/ccache.tar.bz2 && \
    cd /tmp/deploy && tar -xjf ccache.tar.bz2 && \
    cd ccache-3.1.9 && ./configure --prefix=/usr/local && make -j8 && make install && \
    rm -fr /tmp/deploy

# * repo
RUN curl https://storage.googleapis.com/git-repo-downloads/repo -o /usr/local/bin/repo && \
    chmod a+x /usr/local/bin/repo

# Set LANG
ENV LANG=en_US.UTF-8

# Docker Swarm properties - keep this towards end of Dockerfile
COPY build/swarm*.properties /

# Expose SSH daemon and run our builder startup script
EXPOSE 22
ADD .ssh /home/couchbase/.ssh
COPY build/couchbuilder_start.sh /usr/sbin/
ENTRYPOINT [ "/tini", "--", "/usr/sbin/couchbuilder_start.sh" ]
CMD [ "swarm" ]

# Keep this stuff at the end, because the ARG declaration breaks
# Docker build caching
ARG CONTAINER_TAG_ARG
ENV CONTAINER_TAG=${CONTAINER_TAG_ARG}
