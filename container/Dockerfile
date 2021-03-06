FROM nvidia/cuda:9.0-devel-ubuntu16.04

ENV TENSORFLOW_VERSION=1.12.0
ENV HOROVOD_VERSION=0.15.2
ENV CUDNN_VERSION=7.1.4.18-1+cuda9.0
ENV NCCL_VERSION=2.2.13-1+cuda9.0


RUN apt-get update && apt-get install -y --no-install-recommends --allow-downgrades --allow-change-held-packages \
      build-essential \
      cmake \
      git \
      curl \
      vim \
      wget \
      ca-certificates \
      libcudnn7=${CUDNN_VERSION} \
      libnccl2=${NCCL_VERSION} \
      libnccl-dev=${NCCL_VERSION} \
      libjpeg-dev \
      libpng-dev  \
      openssl \
      libssl-dev \
      unzip \
      net-tools

RUN wget https://www.python.org/ftp/python/3.6.6/Python-3.6.6.tgz
RUN tar -xvf Python-3.6.6.tgz

RUN cd Python-3.6.6 && ./configure && make && make install

RUN echo "deb http://developer.download.nvidia.com/compute/machine-learning/repos/ubuntu1604/x86_64 /" > /etc/apt/sources.list.d/nvidia-ml.list

RUN apt-get install -y --no-install-recommends --allow-downgrades --allow-change-held-packages libgtk2.0-dev

RUN pip3 install --upgrade pip

# Install Open MPI
RUN mkdir /tmp/openmpi && \
    cd /tmp/openmpi && \
    wget https://www.open-mpi.org/software/ompi/v3.0/downloads/openmpi-3.0.0.tar.gz && \
    tar zxf openmpi-3.0.0.tar.gz && \
    cd openmpi-3.0.0 && \
    ./configure --enable-orterun-prefix-by-default && \
    make -j $(nproc) all && \
    make install && \
    ldconfig && \
    rm -rf /tmp/openmpi


# Install TensorFlow and Keras
RUN pip3 install tensorflow-gpu==${TENSORFLOW_VERSION} keras h5py

# Install Horovod, temporarily using CUDA stubs
RUN ldconfig /usr/local/cuda-9.0/targets/x86_64-linux/lib/stubs && \
    HOROVOD_GPU_ALLREDUCE=NCCL HOROVOD_WITH_TENSORFLOW=1 pip3 install --no-cache-dir horovod==${HOROVOD_VERSION} && \
    ldconfig

# Create a wrapper for OpenMPI to allow running as root by default
RUN mv /usr/local/bin/mpirun /usr/local/bin/mpirun.real && \
    echo '#!/bin/bash' > /usr/local/bin/mpirun && \
    echo 'mpirun.real --allow-run-as-root "$@"' >> /usr/local/bin/mpirun && \
    chmod a+x /usr/local/bin/mpirun

# Configure OpenMPI to run good defaults:
#   --bind-to none --map-by slot --mca btl_tcp_if_exclude lo,docker0
RUN echo "hwloc_base_binding_policy = none" >> /usr/local/etc/openmpi-mca-params.conf && \
    echo "rmaps_base_mapping_policy = slot" >> /usr/local/etc/openmpi-mca-params.conf

# Set default NCCL parameters
RUN echo NCCL_DEBUG=INFO >> /etc/nccl.conf

ENV LD_LIBRARY_PATH=/usr/local/openmpi/lib:$LD_LIBRARY_PATH
ENV PATH /usr/local/openmpi/bin/:$PATH
ENV PATH=/usr/local/nvidia/bin:$PATH

ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1

# Install OpenSSH for MPI to communicate between containers
RUN apt-get install -y --no-install-recommends openssh-client openssh-server
RUN mkdir -p /var/run/sshd && \
  mkdir -p /root/.ssh/ && \
  ssh-keygen -q -t rsa -N '' -f /root/.ssh/id_rsa && \
  cp /root/.ssh/id_rsa.pub /root/.ssh/authorized_keys && \
  printf "Host *\n  StrictHostKeyChecking no\n" >> /root/.ssh/config


# Install packages here to avoid OpenMPI/Python rebuild for every change
RUN apt-get install -y --no-install-recommends --allow-downgrades --allow-change-held-packages \
    ethtool \
    less \
    iproute

RUN pip3 install awscli
RUN pip3 install boto3
RUN pip3 install ujson
RUN pip3 install opencv-python
RUN pip3 install Cython
RUN pip3 install pycocotools
RUN pip3 install matplotlib 

RUN git clone https://github.com/tensorpack/tensorpack.git /tensorpack 
RUN cd /tensorpack && git fetch origin  2ce43d70456afd618167c9206d868c12e80e3f5a
RUN cd /tensorpack && git reset --hard 2ce43d70456afd618167c9206d868c12e80e3f5a
RUN pip3 install --ignore-installed -e /tensorpack
