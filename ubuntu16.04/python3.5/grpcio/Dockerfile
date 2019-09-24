ARG DOCKER_REGISTRY
FROM ${DOCKER_REGISTRY}/arm64v8/python3.5

RUN apt-get update \
  && apt-get install -y --no-install-recommends build-essential python3-dev wget \
  && wget https://raw.githubusercontent.com/grpc/grpc/master/requirements.txt \
  && pip install -r requirements.txt \
  && GRPC_PYTHON_BUILD_WITH_CYTHON=1 pip3 wheel --no-deps --wheel-dir=./dist grpcio==1.23.0