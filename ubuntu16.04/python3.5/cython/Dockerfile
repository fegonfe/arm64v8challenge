ARG DOCKER_REGISTRY
FROM ${DOCKER_REGISTRY}/arm64v8/python3.5

RUN apt-get update \
  && apt-get install -y --no-install-recommends build-essential python3-dev \
  && pip3 wheel --no-deps --wheel-dir=./dist cython==0.29.13