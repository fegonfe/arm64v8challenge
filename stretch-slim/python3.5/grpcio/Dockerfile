ARG DOCKER_REGISTRY
FROM ${DOCKER_REGISTRY}/arm64v8/python3.5

COPY prereqs prereqs/

RUN apt-get update \
  && apt-get install -y --no-install-recommends build-essential \
  && ls ./prereqs/* > requirements.txt \
  && pip3 install -r requirements.txt \
  && GRPC_PYTHON_BUILD_WITH_CYTHON=1 pip3 wheel --no-deps --wheel-dir=./dist grpcio==1.23.0