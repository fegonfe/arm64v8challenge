ARG DOCKER_REGISTRY
FROM ${DOCKER_REGISTRY}/arm64v8/python3.5

COPY prereqs prereqs/

RUN apt-get update \
    && apt-get install -y --no-install-recommends build-essential python3-dev libatlas3-base libhdf5-dev \
    && ls ./prereqs/* > requirements.txt \
    && pip3 install -r requirements.txt \
    && export HDF5_DIR="/usr/lib/aarch64-linux-gnu/hdf5/serial" \
    && export CC=h5cc \
    && pip3 wheel --no-deps --wheel-dir=./dist h5py