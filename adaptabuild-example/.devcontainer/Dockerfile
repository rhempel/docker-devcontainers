FROM ubuntu:22.04

# Note: The GID is super critical to get right - on my Linux machine it does
#       not seem to matter, but on Windows I believe it requires the GID to
#       match your UID in the WSL system?

ARG USERNAME=adaptabuild
ARG USER_UID=1000
ARG USER_GID=$USER_UID

# Create the user
RUN groupadd --gid $USER_GID $USERNAME \
 && useradd --uid $USER_UID --gid $USER_GID -d /home/$USERNAME -m -s /usr/bin/bash $USERNAME

# Add some tolerance to the apt system ...
# potential solutions:
# [failed] https://stackoverflow.com/questions/48645936/docker-for-mac-hash-sum-mismatch 
# [worked] https://stackoverflow.com/questions/67732260/how-to-fix-hash-sum-mismatch-in-docker-on-mac
 
RUN echo "Acquire::http::Pipeline-Depth 0;"  > /etc/apt/apt.conf.d/99custom \
 && echo "Acquire::http::No-Cache true;" >> /etc/apt/apt.conf.d/99custom \
 && echo "Acquire::BrokenProxy    true;" >> /etc/apt/apt.conf.d/99custom

# [Optional] Add sudo support. Omit if you don't need to install software after connecting

RUN apt update \
 && apt install  --no-install-recommends -y sudo \
 && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME \
 && chmod 0440 /etc/sudoers.d/$USERNAME

# Add a layer with the correct locale and timezone

ENV LANG en_US.UTF-8
ENV TZ Etc/EST

ARG DEBIAN_FRONTEND=noninteractive

# For more information on managing locales in mimimal Ubuntu docker
# images, see: https://hub.docker.com/_/ubuntu

RUN apt install --no-install-recommends -y \
        tzdata \
        locales \
 && localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8

# Add all of the stuff needed to build an embedded target in a
# Docker container. Note that the debugger hardware runs on
# the host, not in the container.
#
# NOTE: Sometimes this step fails if Ubuntu has released a security update
#       for a package in this list. You may need to run:
#
#       sudo apt-cache policy packagename
#
#       to get a list of up to date candidates.
#
# See this note on how apt pinning works - we want to build containers
# with very specific versions, but also allow security updates so
# that a later container build doesn't break because of a patch.
#
# https://help.ubuntu.com/community/PinningHowto
#
COPY preferences /etc/apt/preferences

RUN apt install --no-install-recommends -y \
        git \
        ssh \
        python3 \
        build-essential \
        gcc-arm-none-eabi \
        picolibc-arm-none-eabi \
        gdb-multiarch \
        binutils-multiarch-dev \
        python3-clang \
        python3-pip \
        python3-sphinx \
        lcov \
        cpputest \
        graphviz \
        vim \
        bash-completion \
        wget

RUN ln -s /usr/bin/gdb-multiarch /usr/bin/arm-none-eabi-gdb \
 && ln -s /usr/bin/nm /usr/bin/nm-multiarch \
 && ln -s /usr/bin/objdump /usr/bin/objdump-multiarch
 
RUN wget -P /opt -x -nH --cut-dirs=1 https://github.com/plantuml/plantuml/releases/download/v1.2024.3/plantuml-1.2024.3.jar

# Now install all the additional Python packages
#
COPY requirements.txt /tmp/requirements.txt

RUN pip3 install -r /tmp/requirements.txt

# Don't do this until we are ready to rebuild the machine a final time
# RUN apt -y autopurge && \
#     apt -y autoclean && \
#     apt -y autoremove && \
#     rm -rf /var/lib/apt/lists/*

# [Optional] Set the default user - omit if you want to keep
# the default as root. (You don't)
#
USER $USERNAME
