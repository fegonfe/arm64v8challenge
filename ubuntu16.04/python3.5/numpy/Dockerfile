ARG DOCKER_REGISTRY
FROM ${DOCKER_REGISTRY}/arm64v8/python3.5

RUN apt-get update \
  && apt-get install -y --no-install-recommends build-essential python-dev libatlas-base-dev \
  && pip3 wheel --no-deps --wheel-dir=./dist numpy==1.16.4