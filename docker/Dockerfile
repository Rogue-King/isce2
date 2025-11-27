# --- Builder Stage ---
FROM ubuntu:20.04 as builder

# Set encoding and timezone
ENV LANG en_US.UTF-8
ENV TZ US/Pacific
ARG DEBIAN_FRONTEND=noninteractive

# Install all build dependencies
# ADDED: python3-dev (required for Python.h header files)
RUN set -ex \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        python3 \
        python3-dev \
        python3-pip \
        cmake \
        cython3 \
        gfortran \
        git \
        libfftw3-dev \
        libgdal-dev \
        libhdf4-alt-dev \
        libhdf5-dev \
        libopencv-dev \
        ninja-build \
        python3-gdal \
        python3-h5py \
        python3-numpy \
        python3-scipy \
        dpkg-dev \
        python3-setuptools \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && echo done

# Copy repo
COPY . /opt/isce2/src/isce2

# Build ISCE
# We explicitly set CMAKE_INSTALL_PREFIX to /opt/isce2/install
RUN set -ex \
    && cd /opt/isce2/src/isce2 \
    && mkdir build && cd build \
    && CMAKE_PYTHON_DIR=$(python3 -c 'import site; print(site.getsitepackages()[-1])') \
    && cmake .. \
        -DPYTHON_MODULE_DIR="$CMAKE_PYTHON_DIR" \
        -DCMAKE_INSTALL_PREFIX=/opt/isce2/install \
        -DCMAKE_BUILD_TYPE=Release \
    && make -j$(nproc) install \
    && cpack -G DEB \
    && cp isce*.deb /tmp/

# --- Production Stage ---
FROM ubuntu:20.04

# Set encoding and timezone
ENV LANG en_US.UTF-8
ENV TZ US/Pacific
ARG DEBIAN_FRONTEND=noninteractive

# *** PATH FIXES ***
# Ensure executables are found
ENV PATH="/opt/isce2/install/bin:$PATH"
# Ensure Python can import the module
ENV PYTHONPATH="/usr/lib/python3.8/dist-packages:$PYTHONPATH"

# Install runtime dependencies (python3-dev is NOT needed here, just python3)
RUN set -ex \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        python3 \
        libfftw3-3 \
        libgdal26 \
        libhdf4-0 \
        libhdf5-103 \
        libopencv-core4.2 \
        libopencv-highgui4.2 \
        libopencv-imgproc4.2 \
        python3-gdal \
        python3-h5py \
        python3-numpy \
        python3-scipy \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && echo done

# Install ISCE from DEB
COPY --from=builder /tmp/isce*.deb /tmp/isce2.deb

RUN set -ex \
    && dpkg -i /tmp/isce2.deb \
    && rm /tmp/isce2.deb

# Symlink compatibility
RUN ln -s /usr/lib/python3.8/dist-packages/isce2 /usr/lib/python3.8/dist-packages/isce
