# Builder phase remains unchanged
FROM debian:bullseye-slim as builder
ARG DOCKER_TAG
ARG BUILD_CONCURRENCY
RUN mkdir -p /src  && mkdir -p /opt

RUN apt-get update && \
    apt-get -y --no-install-recommends install ca-certificates cmake make git gcc g++ libbz2-dev libxml2-dev wget \
    libzip-dev libboost1.74-all-dev lua5.4 liblua5.4-dev pkg-config -o APT::Install-Suggests=0 -o APT::Install-Recommends=0

RUN NPROC=${BUILD_CONCURRENCY:-$(nproc)} && \
    ldconfig /usr/local/lib && \
    git clone --branch v2021.3.0 --single-branch https://github.com/oneapi-src/oneTBB.git && \
    cd oneTBB && \
    mkdir build && \
    cd build && \
    cmake -DTBB_TEST=OFF -DCMAKE_BUILD_TYPE=Release ..  && \
    cmake --build . && \
    cmake --install .

COPY . /src
WORKDIR /src

RUN NPROC=${BUILD_CONCURRENCY:-$(nproc)} && \
    echo "Building OSRM ${DOCKER_TAG}" && \
    git show --format="%H" | head -n1 > /opt/OSRM_GITSHA && \
    echo "Building OSRM gitsha $(cat /opt/OSRM_GITSHA)" && \
    mkdir -p build && \
    cd build && \
    BUILD_TYPE="Release" && \
    ENABLE_ASSERTIONS="Off" && \
    BUILD_TOOLS="Off" && \
    case ${DOCKER_TAG} in *"-debug"*) BUILD_TYPE="Debug";; esac && \
    case ${DOCKER_TAG} in *"-assertions"*) BUILD_TYPE="RelWithDebInfo" && ENABLE_ASSERTIONS="On" && BUILD_TOOLS="On";; esac && \
    echo "Building ${BUILD_TYPE} with ENABLE_ASSERTIONS=${ENABLE_ASSERTIONS} BUILD_TOOLS=${BUILD_TOOLS}" && \
    cmake .. -DCMAKE_BUILD_TYPE=${BUILD_TYPE} -DENABLE_ASSERTIONS=${ENABLE_ASSERTIONS} -DBUILD_TOOLS=${BUILD_TOOLS} -DENABLE_LTO=On && \
    make -j${NPROC} install && \
    cd ../profiles && \
    cp -r * /opt && \
    strip /usr/local/bin/* && \
    rm -rf /src

# Second stage build: changing from Debian to Ubuntu 22.04
FROM ubuntu:22.04 as runstage

COPY --from=builder /usr/local /usr/local
COPY --from=builder /opt /opt

RUN apt-get update && \
    apt-get install -y --no-install-recommends libboost-program-options-dev libboost-regex-dev \
        libboost-date-time-dev libboost-chrono-dev libboost-filesystem-dev \
        libboost-iostreams-dev libboost-system-dev libboost-thread-dev \
        expat liblua5.4-0 && \
        python3 python3-pip && \
    rm -rf /var/lib/apt/lists/* && \
    ldconfig /usr/local/lib

# Install Python packages
RUN pip3 install fastapi uvicorn

RUN /usr/local/bin/osrm-extract --help && \
    /usr/local/bin/osrm-routed --help && \
    /usr/local/bin/osrm-contract --help && \
    /usr/local/bin/osrm-partition --help && \
    /usr/local/bin/osrm-customize --help

WORKDIR /

EXPOSE 5000
CMD ["sleep", "infinity"]