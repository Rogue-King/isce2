# --- Builder Stage: Compile ISCE2 ---
FROM ubuntu:20.04 as builder

# Set encoding and timezone
ENV LANG en_US.UTF-8
ENV TZ US/Pacific
ARG DEBIAN_FRONTEND=noninteractive

# Install all build dependencies
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

# --- Production Stage: Install and Configure ---
FROM ubuntu:20.04

# Set encoding and timezone
ENV LANG en_US.UTF-8
ENV TZ US/Pacific
ARG DEBIAN_FRONTEND=noninteractive

# Install runtime dependencies
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
        python3-opencv \
        python3-gdal \
        python3-h5py \
        python3-numpy \
        python3-scipy \
        python3-matplotlib \
        zip \
        unzip \
        curl \
        wget \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && echo done

# 1. Install ISCE from DEB
COPY --from=builder /tmp/isce*.deb /tmp/isce2.deb
RUN set -ex \
    && dpkg -i /tmp/isce2.deb \
    && rm /tmp/isce2.deb

# 2. Setup Stack Processors (contrib)
COPY --from=builder /opt/isce2/src/isce2/contrib /opt/isce2/contrib

# 3. Symlink Stack Scripts to /usr/local/bin (ROBUST FIX)
# Instead of relying on PATH variables which can be overwritten, we link executables
# directly to a standard system bin directory.
RUN chmod -R +x /opt/isce2/contrib/stack \
    && ln -sf /opt/isce2/contrib/stack/topsStack/*.py /usr/local/bin/ \
    && ln -sf /opt/isce2/contrib/stack/stripmapStack/*.py /usr/local/bin/

# 4. Configure Environment Variables
# ISCE_HOME: Hardcoded location of binaries
# PATH: Add ISCE main bin (Stack scripts are now in /usr/local/bin so they don't need PATH entry)
# PYTHONPATH: Add 'contrib/stack' so python can find the modules "topsStack" and "stripmapStack"
ENV ISCE_HOME="/opt/isce2/install"
ENV PATH="$ISCE_HOME/bin:$PATH"
ENV PYTHONPATH="/opt/isce2/contrib/stack:/usr/lib/python3.8/dist-packages:$PYTHONPATH"

# 5. Compatibility Fixes
# - Symlink 'isce2' to 'isce' for scripts expecting the old package name.
# - Patch __init__.py with 'stanford_license = True' because stack processors (legacy)
#   still check for this attribute, even though ISCE2 is now open source.
RUN ln -s /usr/lib/python3.8/dist-packages/isce2 /usr/lib/python3.8/dist-packages/isce \
    && echo "stanford_license = True" >> /usr/lib/python3.8/dist-packages/isce2/__init__.py
