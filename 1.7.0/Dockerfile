FROM alpine:latest AS build

# Based on https://github.com/tatsushid/docker-alpine-py3-tensorflow-jupyter/blob/master/Dockerfile
# Changes:
# - Bumping versions of Bazel and Tensorflow
# - Add -Xmx to the Java params when building Bazel
# - Disable TF_GENERATE_BACKTRACE and TF_GENERATE_STACKTRACE

ENV JAVA_HOME /usr/lib/jvm/java-1.8-openjdk
ENV LOCAL_RESOURCES 2048,.5,1.0
ENV BAZEL_VERSION 0.10.1
RUN apk add --no-cache python3 python3-tkinter py3-numpy py3-numpy-f2py freetype libpng libjpeg-turbo imagemagick graphviz git
RUN apk add --no-cache py3-pip
RUN apk add --no-cache --virtual=.build-deps \
        bash \
        cmake \
        curl \
        freetype-dev \
        g++ \
        libjpeg-turbo-dev \
        libpng-dev \
        linux-headers \
        make \
        musl-dev \
        openblas-dev \
        openjdk8 \
        patch \
        perl \
        python3-dev \
        rsync \
        sed \
        swig \
        zip \
    && cd /tmp \
    && pip3 install --no-cache-dir wheel \
    && $(cd /usr/bin && ln -s python3 python)

# Bazel download
RUN curl -SLO https://github.com/bazelbuild/bazel/releases/download/0.10.1/bazel-0.10.1-dist.zip \
    && mkdir bazel-0.10.1 \
    && unzip -qd bazel-0.10.1 bazel-0.10.1-dist.zip

#
# Alpine unzip is a little overzealous in its zip bomb detection, so...
# replace it with a wrapper which calls 7zip
#
RUN apk add --no-cache p7zip && rm -rf /usr/bin/unzip
COPY patches/unzip /usr/bin/unzip

# Bazel install. Let it fail once, make an Alpine patch, then resume and finish
RUN cd bazel-0.10.1 \
    && sed -i -e 's/-classpath/-J-Xmx8192m -J-Xms128m -classpath/g' scripts/bootstrap/compile.sh \
    && bash compile.sh; exit 0
RUN grep -v static.long.gettid /bazel-0.10.1/third_party/grpc/src/core/support/log_linux.c > t && \
    mv t /bazel-0.10.1/third_party/grpc/src/core/support/log_linux.c
RUN cd /bazel-0.10.1 && bash compile.sh && cp -p output/bazel /usr/bin/

# Download Tensorflow
ENV TENSORFLOW_VERSION 1.7.0
RUN cd /tmp \
    && curl -SL https://github.com/tensorflow/tensorflow/archive/v${TENSORFLOW_VERSION}.tar.gz \
        | tar xzf -

# Build Tensorflow
RUN cd /tmp/tensorflow-${TENSORFLOW_VERSION} \
    && : musl-libc does not have "secure_getenv" function \
    && sed -i -e '/define TF_GENERATE_BACKTRACE/d' tensorflow/core/platform/default/stacktrace.h \
    && PYTHON_BIN_PATH=/usr/bin/python \
        PYTHON_LIB_PATH=/usr/lib/python3.9/site-packages \
        CC_OPT_FLAGS="-march=native" \
        TF_NEED_JEMALLOC=1 \
        TF_NEED_GCP=0 \
        TF_NEED_HDFS=0 \
        TF_NEED_S3=0 \
        TF_ENABLE_XLA=0 \
        TF_NEED_GDR=0 \
        TF_NEED_VERBS=0 \
        TF_NEED_OPENCL=0 \
        TF_NEED_CUDA=0 \
        TF_NEED_MPI=0 \
        bash configure

# start the build process; allow it to fail but create directories which need some patching
RUN mv /root/embedded_tools /root/.cache/bazel/_bazel_root/install/*/_embedded_binaries/ && \
    echo "https://dl-cdn.alpinelinux.org/alpine/latest-stable/community" >> /etc/apk/repositories && apk update && apk add py3-numpy-dev && \
    ln -s /usr/lib/python3.9/site-packages/numpy/core/include/numpy/ /usr/include/numpy
RUN cd /tmp/tensorflow-${TENSORFLOW_VERSION} \
    && bazel build --config=opt -c opt --local_resources ${LOCAL_RESOURCES} //tensorflow/tools/pip_package:build_pip_package; exit 0

# alpine patches
RUN find /root/.cache/bazel/_bazel_root/*/external/cython/Cython -type f -exec sed -i -e 's/exc_/curexc_/g' {} \; && \
    find /root/.cache/bazel/_bazel_root/*/external/protobuf_archive/python/google/protobuf/pyext -name "*.cc" -exec sed -i "s/PyUnicode_AsUTF8/(char*)PyUnicode_AsUTF8/g" {} \; && \
    find /root/.cache/bazel/_bazel_root/*/external/cython/Cython -type f -exec sed -i -e 's/curcurexc_/curexc_/g' {} \; && \
    find /tmp/tensorflow-1.7.0/ -name "*.cc" -exec sed -i "s/PyUnicode_AsUTF8/(char*)PyUnicode_AsUTF8/g" {} \; && \
    grep -v static.long.gettid /root/.cache/bazel/_bazel_root/*/external/grpc/src/core/lib/support/log_linux.cc > t && \
    mv t /root/.cache/bazel/_bazel_root/*/external/grpc/src/core/lib/support/log_linux.cc

# more alpine patches
COPY patches /patches
RUN cp /patches/stacktrace_handler.cc /tmp/tensorflow-1.7.0/tensorflow/core/platform/stacktrace_handler.cc && \
    cp /patches/ExprNodes.py /root/.cache/bazel/_bazel_root/*/external/cython/Cython/Compiler/ && \
    cp /patches/ParseTreeTransforms.py /root/.cache/bazel/_bazel_root/*/external/cython/Cython/Compiler/ && \
    cp /patches/jemalloc.c /root/.cache/bazel/_bazel_root/*/external/jemalloc/src && \
    cp /patches/pycapsule.pxd /root/.cache/bazel/_bazel_root/*/external/cython/Cython/Includes/cpython && \
    cp /patches/pywrap_tfe_src.cc /tmp/tensorflow-1.7.0/tensorflow/python/eager/ && \
    cp /patches/bfloat16.cc /tmp/tensorflow-1.7.0/tensorflow/python/lib/core && \
    cp /patches/ndarray_tensor_bridge.cc /tmp/tensorflow-1.7.0/tensorflow/python/lib/core && \
    cp /patches/python_configure.bzl /tmp/tensorflow-1.7.0/third_party/py

# resume and complete the patched build
RUN cp /patches/CROSSTOOL /tmp/tensorflow-1.7.0/third_party/toolchains/gpus/crosstool/CROSSTOOL && \
    cp /patches/CROSSTOOL /tmp/tensorflow-1.7.0/CROSSTOOL && \
    cd /tmp/tensorflow-${TENSORFLOW_VERSION} \
    && echo "cxx_builtin_include_directory: \"/tmp/tensorflow-${TENSORFLOW_VERSION}/bazel-out/k8-py3-opt/genfiles/external/local_config_python/numpy_include/numpy\"" >> /tmp/tensorflow-${TENSORFLOW_VERSION}/CROSSTOOL \
    && bazel build --config=opt -c opt --local_resources ${LOCAL_RESOURCES} //tensorflow/tools/pip_package:build_pip_package; exit 0
RUN find /tmp/tensorflow-1.7.0/tensorflow/python/eager/pywrap_tfe_src.cc -exec sed -i "s/return PyUnicode_AsUTF8/return (char*)PyUnicode_AsUTF8/g" {} \; && \
    cd /tmp/tensorflow-${TENSORFLOW_VERSION} && \
    bazel build --config=opt -c opt --local_resources ${LOCAL_RESOURCES} //tensorflow/tools/pip_package:build_pip_package; exit 0
RUN cd /tmp/tensorflow-${TENSORFLOW_VERSION} \
    && ./bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg
RUN cp /tmp/tensorflow_pkg/tensorflow-${TENSORFLOW_VERSION}-cp39-cp39-linux_x86_64.whl /root

# copy wheel over and install

FROM alpine:latest

ENV TENSORFLOW_VERSION 1.7.0
COPY --from=build /root/tensorflow-${TENSORFLOW_VERSION}-cp39-cp39-linux_x86_64.whl /
RUN apk add --no-cache python3 py3-pip build-base python3-dev linux-headers && \
    apk add py3-grpcio && \
    python3 -m pip install /tensorflow-${TENSORFLOW_VERSION}-cp39-cp39-linux_x86_64.whl && \
    apk add openblas-dev py3-setuptools py3-scipy py3-h5py && \
    python3 -m pip install keras==2.1.6 && \
    apk del --purge build-base
COPY patches/models.py /usr/lib/python3.9/site-packages/keras/models.py
