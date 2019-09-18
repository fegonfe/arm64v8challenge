ARG DOCKER_REGISTRY
FROM ${DOCKER_REGISTRY}/arm64v8/base

RUN apt-get update \
    && apt-get install -y --no-install-recommends wget ca-certificates python3 \
    && wget -nv https://bootstrap.pypa.io/get-pip.py \
    ## installs pip using recommended method
    && python3 get-pip.py --disable-pip-version-check --no-cache-dir \
    && apt-get purge -y wget \
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/* \
    && rm -f get-pip.py