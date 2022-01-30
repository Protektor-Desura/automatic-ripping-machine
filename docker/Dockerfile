
# use --build-arg target=(ui|ripper|combined)
ARG target

###########################################################
# base image, used for build stages and final images
FROM ubuntu:20.04 as base

# override at runtime to match user that ARM runs as local user
ENV RUN_AS_USER=true
ENV UID=1000
ENV GID=1000
# override at runtime to change makemkv key
ENV MAKEMKV_APP_KEY=""

# local apt/deb proxy for builds
ARG APT_PROXY=""
RUN if [ -n "${APT_PROXY}" ] ; then \
  printf 'Acquire::http::Proxy "%s";' "${APT_PROXY}" \
  > /etc/apt/apt.conf.d/30proxy ; fi

RUN mkdir /opt/arm
WORKDIR /opt/arm

COPY ./scripts/add-ppa.sh /root/add-ppa.sh
# setup Python virtualenv and gnupg/wget for add-ppa.sh
RUN \
  apt update -y && \
  DEBIAN_FRONTEND=noninteractive apt upgrade -y && \
  DEBIAN_FRONTEND=noninteractive apt install -y --no-install-recommends \
    gnupg \
    gosu \
    python3 \
    python3-venv \
    udev \
    wget \
    build-essential \
    && \
  DEBIAN_FRONTEND=noninteractive apt clean -y && \
  rm -rf /var/lib/apt/lists/* 

ENV VIRTUAL_ENV=/opt/venv
RUN python3 -m venv "${VIRTUAL_ENV}"
ENV PATH="${VIRTUAL_ENV}/bin:${PATH}"

###########################################################
# build libdvd in a separate stage, pulls in tons of deps
FROM base as libdvd


RUN \
  bash /root/add-ppa.sh ppa:mc3man/focal6 && \
  apt update -y && \
  DEBIAN_FRONTEND=noninteractive apt install -y --no-install-recommends libdvd-pkg && \
  DEBIAN_FRONTEND=noninteractive dpkg-reconfigure libdvd-pkg && \
  DEBIAN_FRONTEND=noninteractive apt clean -y && \
  rm -rf /var/lib/apt/lists/*

###########################################################
# build pip reqs for ripper in separate stage
FROM base as pip-ripper
COPY requirements.ripper.txt /requirements.txt
RUN \
  apt update -y && \
  DEBIAN_FRONTEND=noninteractive apt install -y --no-install-recommends \
    build-essential \
    libcurl4-openssl-dev \
    libssl-dev \
    python3 \
    python3-dev \
    python3-pyudev \
    python3-wheel \
    udev \
    libudev-dev \
    python3-pip \
    && \
  pip3 install --upgrade pip wheel setuptools \
  && \
  pip3 install pyudev \
  && \
  pip3 install \
    --ignore-installed \
    --prefer-binary \
    -r /requirements.txt


###########################################################
# build pip reqs for UI in separate stage
FROM base as pip-ui
COPY requirements.ui.txt /requirements.txt
RUN \
  apt update -y && \
  DEBIAN_FRONTEND=noninteractive apt install -y --no-install-recommends \
    build-essential \
    python3 \
    python3-dev \
    python3-pip \
    udev \
    && \
  pip3 install \
    --ignore-installed \
    --prefer-binary \
    -r /requirements.txt

###########################################################
# install deps for ripper
FROM base as deps-ripper
RUN \
  bash /root/add-ppa.sh ppa:heyarje/makemkv-beta && \
  bash /root/add-ppa.sh ppa:stebbins/handbrake-releases && \
  apt update -y && \
  DEBIAN_FRONTEND=noninteractive apt install -y --no-install-recommends \
    abcde \
    eyed3 \
    atomicparsley \
    cdparanoia \
    eject \
    ffmpeg \
    flac \
    glyrc \
    default-jre-headless \
    handbrake-cli \
    libavcodec-extra \
    makemkv-bin \
    makemkv-oss \
    udev \
    python3 \
    python3-dev \
    python3-pip \
    python3-venv \
    libudev-dev \
    python3-wheel \
    python-psutil \
    python3-pyudev \
    && \
    pip3 install wheel \
    && \
    pip3 install --upgrade pip wheel setuptools \
    && \
    pip3 install --upgrade psutil \
    && \
    pip3 install pyudev \
    && \
  DEBIAN_FRONTEND=noninteractive apt clean -y && \
  rm -rf /var/lib/apt/lists/*


# copy pip reqs from build stage
COPY --from=pip-ripper /opt/venv /opt/venv

# copy just the .deb from libdvd build stage
COPY --from=libdvd /usr/src/libdvd-pkg/libdvdcss2_*.deb /opt/arm
# installing with --ignore-depends to avoid all it's deps
# leaves apt in a broken state so do package install last
RUN DEBIAN_FRONTEND=noninteractive dpkg -i --ignore-depends=libdvd-pkg /opt/arm/libdvdcss2_*.deb

###########################################################
# all the UI deps are handled by pip
FROM base as deps-ui

# copy pip reqs from build stage
COPY --from=pip-ui /opt/venv /opt/venv

###########################################################
# combined image=ripper+ui
FROM deps-ripper as deps-combined

# copy pip reqs from build stage
COPY --from=pip-ui /opt/venv /opt/venv
COPY --from=pip-ripper /opt/venv /opt/venv
###########################################################
# build final image
FROM deps-combined AS install
# default directories and configs
RUN \
  mkdir -m 0777 -p /home/arm /home/arm/config /mnt/dev/sr0 /mnt/dev/sr1 /mnt/dev/sr2 /mnt/dev/sr3 /mnt/dev/sr4 && \
  ln -sv /home/arm/config/arm.yaml /opt/arm/arm.yaml && \
  ln -sv /opt/arm/apprise.yaml /home/arm/config/apprise.yaml && \
  echo "/dev/sr0  /mnt/dev/sr0  udf,iso9660  users,noauto,exec,utf8,ro  0  0" >> /etc/fstab  && \
  echo "/dev/sr1  /mnt/dev/sr1  udf,iso9660  users,noauto,exec,utf8,ro  0  0" >> /etc/fstab  && \
  echo "/dev/sr2  /mnt/dev/sr2  udf,iso9660  users,noauto,exec,utf8,ro  0  0" >> /etc/fstab  && \
  echo "/dev/sr3  /mnt/dev/sr3  udf,iso9660  users,noauto,exec,utf8,ro  0  0" >> /etc/fstab  && \
  echo "/dev/sr4  /mnt/dev/sr4  udf,iso9660  users,noauto,exec,utf8,ro  0  0" >> /etc/fstab

# copy ARM source last, helps with Docker build caching
COPY . /opt/arm/ 

EXPOSE 8080
#VOLUME /home/arm
VOLUME /home/arm/Music
VOLUME /home/arm/logs
VOLUME /home/arm/media
VOLUME /home/arm/config
WORKDIR /home/arm

ENTRYPOINT ["/opt/arm/scripts/docker-entrypoint.sh"]
CMD ["python3", "/opt/arm/arm/runui.py"]


# pass build args for labeling
ARG image_revision=2.5.3
ARG image_created="2021-04-29"

LABEL org.opencontainers.image.source=https://github.com/1337-server/automatic-ripping-machine
LABEL org.opencontainers.image.revision="2.5.3"
LABEL org.opencontainers.image.created="2021-04-29"
LABEL org.opencontainers.image.license=MIT
