############################################################
# Dockerfile to build OpenZWave Library container images
# Based on CentOS7
############################################################

ARG distcchosts=''
ARG concurrency="12"
ARG qtversion="5.15.1"


# Set the base image to Alpine
#FROM fishwaldo/ozw-base:5.12.9 as builder
FROM debian:bullseye-slim as builder

# File Author / Maintainer
LABEL maintainer="igor@m3ki.net"


ENV LC_ALL C
ENV DEBIAN_FRONTEND noninteractive

WORKDIR /opt

RUN apt update && apt-get -y install build-essential rapidjson-dev cmake pkgconf python libunwind-dev libcurl4-openssl-dev libx11-dev libglu1-mesa-dev libexecline-dev qtbase5-dev qt5-qmake qt5-qmake-bin libqt5core5a libqt5remoteobjects5 libqt5websockets5 libqt5svg5 libqt5serialport5-dev libqt5remoteobjects5-dev libqt5websockets5-dev libqt5svg5-dev libqt5serialport5-dev qtbase5-private-dev git

ARG distcchosts=''
ARG concurrency="12"
ARG qtversion="5.15.1"

ENV DISTCC_HOSTS="$distcchosts"

RUN if [ -n "${DISTCC_HOSTS}" ]; then cd /usr/local/bin; ln /usr/bin/distcc gcc;  ln /usr/bin/distcc g++; echo "Using DistCC hosts ${DISTCC_HOSTS}"; fi

RUN git clone https://github.com/OpenZWave/open-zwave.git \
	&& cd open-zwave \
	&& make -j${concurrency} \
	&& make install instlibdir=/usr/local/lib/

COPY .qmake.conf /opt/qt-openzwave/
COPY qt-openzwave /opt/qt-openzwave/qt-openzwave
COPY qt-openzwave.pri /opt/qt-openzwave/
COPY qt-openzwave.pro /opt/qt-openzwave/
COPY qt-openzwavedatabase /opt/qt-openzwave/qt-openzwavedatabase
COPY qt-ozwdaemon /opt/qt-openzwave/qt-ozwdaemon
COPY qt-openzwave.pro /opt/qt-openzwave/

ARG BP_CLIENTID
ARG BP_CLIENTKEY
ARG BUILDNUMBER=0

RUN git clone --depth 1 -b \
        ${qtversion} https://github.com/qt/qtmqtt /usr/src/qtmqtt \
    && cd /usr/src/qtmqtt \
    && /usr/lib/qt5/bin/qmake CONFIG+=release QT_BUILD_PARTS="libs tools" \
    && make -j $(( $(nproc) + 1 )) \
    && make install


RUN cd qt-openzwave \
	&& if [ -f Makefile ]; then qmake -r; make distclean; fi \
	&& qmake -r CONFIG+=release "BP_CLIENTID=$BP_CLIENTID" "BP_CLIENTKEY=$BP_CLIENTKEY" "BUILDNUMBER=$BUILDNUMBER" \
	&& make -j${concurrency} \
	&& make install

RUN git clone https://github.com/OpenZWave/ozw-admin.git \
	&& cd ozw-admin \
	&& if [ -f Makefile ]; then qmake -r; make distclean; fi \
	&& qmake -r CONFIG+=release \
	&& make -j${concurrency} \
	&& make install


ENV LD_LIBRARY_PATH="/usr/local/lib:$LD_LIBRARY_PATH"

FROM debian:bullseye-slim as base

ARG qtversion="5.15.1"

ENV LC_ALL C
ENV DEBIAN_FRONTEND noninteractive

LABEL maintainer="igor@m3ki.net"

WORKDIR /opt

RUN apt update \
    && apt-get -y install libunwind8 libcurl4 binutils libglib2.0-0 libicu67 wget \
    && apt-get -y purge binutils \
    && apt-get -y autoremove \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN case `uname -m` in x86_64) var="amd64";; armhf) var="armhf";; armv7l) var="armhf";; aarch64) var="aarch64";; esac && wget https://github.com/just-containers/s6-overlay/releases/download/v2.0.0.1/s6-overlay-$var.tar.gz -O /tmp/s6-overlay.tar.gz \
	&& tar xzf /tmp/s6-overlay.tar.gz -C / \
	&& rm /tmp/s6-overlay.tar.gz

RUN apt update\
    && apt-get -y install libqt5core5a libqt5remoteobjects5 libqt5websockets5 libqt5network5

COPY --from=builder /usr/local/bin/ozwdaemon /usr/local/bin/s6* /usr/local/bin/
COPY --from=builder /usr/local/lib/libopenzwave.so.1.6 /usr/local/lib/
COPY --from=builder /usr/local/lib/libQt5Mqtt* /usr/local/lib/
COPY --from=builder /usr/local/lib/libqt-openzwave* /usr/local/lib/
COPY --from=builder /usr/share/qt5/qt-openzwavedatabase.rcc /usr/share/qt5/
COPY --from=builder /usr/local/bin/* /usr/local/bin/
COPY --from=builder /usr/lib/*/libqt-* /usr/local/lib/
COPY --from=builder /usr/local/lib*/libopenzwave* /usr/local/lib/
COPY --from=builder /usr/lib/*/libQt5Mqtt* /usr/local/lib/


COPY Docker/base/ /

RUN ldconfig -v

ENV BP_DB_PATH="/opt/ozw/config/crashes/"
ENV S6_LOGGING_SCRIPT="n20 s10000000 T"
WORKDIR /opt/ozw/
EXPOSE 1983
VOLUME ["/opt/ozw/config/"]
ENTRYPOINT ["/init"]

FROM base as allinone

ARG qtversion="5.15.1"

ENV LC_ALL C
ENV DEBIAN_FRONTEND noninteractive

ADD https://github.com/novnc/websockify/archive/v0.9.0.tar.gz /opt/source/
ADD https://github.com/novnc/noVNC/archive/v1.2.0.tar.gz /opt/source/

RUN apt update \
    && apt-get -y install libgl1 libpng16-16 libharfbuzz0b libfontconfig1 libmtdev1 libinput10 libxkbcommon0 libdbus-1-3 python3-setuptools python3-numpy libqt5xml5 libqt5svg5 libqt5serialport5 libqt5dbus5 libqt5gui5 libqt5widgets5\
    && apt-get -y autoremove \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
	&& cd /opt/source/ \
	&& tar -xzvf v0.9.0.tar.gz \
	&& cd websockify-0.9.0 \
	&& python3 setup.py install \
	&& cd /opt/source/ \
	&& tar -xzvf v1.2.0.tar.gz \
	&& cp -R /opt/source/noVNC-1.2.0/ /opt/novnc/ \
	&& rm -rf /opt/source/*


COPY --from=builder /usr/local/bin/ozwadmin /usr/local/bin/

COPY Docker/allinone/ /

ENV BP_DB_PATH="/opt/ozw/config/crashes/"
ENV VNC_PORT=5901
ENV WEB_PORT=7800
ENV QT_ASSUME_STDERR_HAS_CONSOLE=1
ENV S6_LOGGING_SCRIPT="n20 s10000000 T"
WORKDIR /opt/ozw/
EXPOSE 1983
EXPOSE 5901
EXPOSE 7800
VOLUME ["/opt/ozw/config/"]

ENTRYPOINT ["/init"]
