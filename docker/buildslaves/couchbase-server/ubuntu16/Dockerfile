# Docker container for Ubuntu 16.04

FROM ubuntu:16.04
MAINTAINER ceej@couchbase.com

USER root

# Install SSH server and sudo
RUN apt-get update && \
    apt-get install -y openssh-server sudo curl && \
    apt-get clean && \
    mkdir /var/run/sshd

# Create couchbase user with password-less sudo privs, and give
# ownership of /opt/couchbase
RUN useradd couchbase -G sudo -m -s /bin/bash && \
    mkdir -p /opt/couchbase && chown -R couchbase:couchbase /opt/couchbase && \
    echo 'couchbase:couchbase' | chpasswd && \
    sed -ri 's/ALL\) ALL/ALL) NOPASSWD:ALL/' /etc/sudoers

# Jenkins Swarm jar
RUN apt-get update && \
    apt-get install -y openjdk-8-jre-headless && \
    apt-get clean
RUN curl \
    https://repo.jenkins-ci.org/releases/org/jenkins-ci/plugins/swarm-client/3.13/swarm-client-3.13.jar \
    -o /usr/local/lib/swarm-client.jar

# tini, to avoid zombie issues when running with Swarm jar
ENV TINI_VERSION v0.18.0
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /tini
RUN chmod +x /tini

# Install Couchbase build dependencies
RUN apt-get update && \
    apt-get install -y \
        ccache \
        git-core \
        pkg-config \
        tar \
        libssl-dev \
        libnuma-dev \
        ruby \
        rake \
        ncurses-dev \
        python-dev \
        python-pip \
        devscripts \
        debhelper \
        dh-systemd \
        libsqlite3-dev \
        ed \
        man \
        pixz && \
    apt-get clean

# GCC (from couchbasebuild/ubuntu-1604-gcc)
COPY build/local /usr/local
RUN  (echo "/usr/local/lib64"; cat /etc/ld.so.conf) > /tmp/ld.so.conf && \
     mv /tmp/ld.so.conf /etc && ldconfig

# Install third-party build dependencies.
RUN apt-get update && \
    apt-get install -y m4 autoconf automake libtool && \
    apt-get clean

RUN echo 'PATH="/usr/lib/ccache:$PATH"' >> /home/couchbase/.profile
RUN mkdir /tmp/deploy && \
    curl https://cmake.org/files/v3.13/cmake-3.13.0-Linux-x86_64.sh -o /tmp/deploy/cmake.sh && \
    (echo y ; echo n) | sh /tmp/deploy/cmake.sh --prefix=/usr/local && \
    curl https://storage.googleapis.com/git-repo-downloads/repo -o /usr/local/bin/repo && \
    chmod a+x /usr/local/bin/repo && \
    cd /tmp && rm -rf /tmp/deploy

# Python 3-specific stuff for build scripts, mostly for Jira.
RUN apt-get update && \
    apt-get install -y software-properties-common && \
    add-apt-repository ppa:deadsnakes/ppa && \
    apt-get update && \
    apt-get -y install python3.6 python3.6-dev python3.6-venv libffi-dev && \
    apt-get clean && \
    update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.6 1 && \
    curl https://bootstrap.pypa.io/get-pip.py | python3 && \
    rm /usr/local/bin/pip /usr/local/bin/easy_install

RUN pip3 install lxml oauthlib==0.7.2 jwt pycrypto jira boto3 pbr && \
    apt-get autoremove --purge -y && \
    apt-get clean && \
    ccache --clear

# Still need paramiko in python2 for testrunner (necessary at least for simple-test,
# which runs in the build image rather than the full CV image)
RUN pip install paramiko

# s3cmd for upload script
RUN apt-get update && \
    apt-get install -y s3cmd && \
    apt-get clean

# Set 8G ccache limit
USER couchbase
RUN ccache -M 8G
USER root

# Update locale
RUN apt-get update && \
    apt-get install -y locales && \
    apt-get clean && \
    locale-gen en_US.UTF-8

# Set AWS credentials file environmental variable
ENV AWS_SHARED_CREDENTIALS_FILE=/home/couchbase/.ssh/aws-credentials

# Docker Swarm properties - keep this towards end of Dockerfile
COPY build/swarm*.properties /

# Expose SSH daemon and run our builder startup script
EXPOSE 22
ADD .ssh /home/couchbase/.ssh
COPY build/couchbuilder_start.sh /usr/sbin/
ENTRYPOINT [ "/tini", "--", "/usr/sbin/couchbuilder_start.sh" ]
CMD [ "swarm" ]

# Set LANG
ENV LANG=en_US.UTF-8
RUN echo "LANG=${LANG}" > /home/couchbase/.ssh/environment && \
    chown couchbase:couchbase /home/couchbase/.ssh/environment && \
    echo "PermitUserEnvironment yes" >> /etc/ssh/sshd_config

# Keep this stuff at the end, because the ARG declaration breaks
# Docker build caching
ARG CONTAINER_TAG_ARG
ENV CONTAINER_TAG=${CONTAINER_TAG_ARG}
