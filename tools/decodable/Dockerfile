###############################################################################
#  Licensed to the Apache Software Foundation (ASF) under one
#  or more contributor license agreements.  See the NOTICE file
#  distributed with this work for additional information
#  regarding copyright ownership.  The ASF licenses this file
#  to you under the Apache License, Version 2.0 (the
#  "License"); you may not use this file except in compliance
#  with the License.  You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
#  Unless required by applicable law or agreed to in writing, software
#  distributed under the License is distributed on an "AS IS" BASIS,
#  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#  See the License for the specific language governing permissions and
# limitations under the License.
###############################################################################

# To build locally (adjust version as needed):
#
# docker build -f tools/decodable/Dockerfile \
#   -t decodable/decodable-product-flink:custom-1.19.1-deco2-java11 \
#   --build-context entrypoint=tools/decodable \
#   build-target

FROM eclipse-temurin:11-jdk-jammy

# Install dependencies
RUN set -ex; \
  apt-get update; \
  apt-get -y install gpg libsnappy1v5 gettext-base libjemalloc-dev jq unzip; \
  rm -rf /var/lib/apt/lists/*

# Install Python 3.11 and pip for Python 3.11
# The default on Ubuntu 22.04 (Jammy Jellyfish) is Python 3.10,
# and the Jammy 3.11 package is for a preview version. Hence:
#
# - Getting 3.11 from deadsnakes/ppa
# - Installing pip manually into 3.11
# - Manually symlinking 3.11 to python3
RUN apt-get update -y && \
  apt-get install -y software-properties-common build-essential && \
  add-apt-repository ppa:deadsnakes/ppa -y && \
  apt-get remove -y --auto-remove software-properties-common && \
  apt-get install -y python3.11 python3.11-dev && \
  rm -rf /var/lib/apt/lists/* && \
  ln -s /usr/bin/python3.11 /usr/bin/python3 && \
  curl -sS https://bootstrap.pypa.io/get-pip.py | python3 && \
  python3 -m pip install --upgrade pip

# Setup AWS CLI 
RUN pip3 install awscli --upgrade
ENV PATH=$PATH:/usr/local/bin/

# Install yq for patching config.yaml; a kingdom for an OS package...
ENV YQ_VERSION="v4.44.3"
RUN mkdir /tmp/jq_release; \
  wget -qO /tmp/jq_release "https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/checksums"; \
  wget -qO /tmp/jq_release "https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/checksums_hashes_order"; \
  wget -qO /tmp/jq_release "https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/extract-checksum.sh"; \
  wget -qO /usr/local/bin/yq "https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/yq_linux_amd64-$(dpkg --print-architecture)"; \
  /tmp/jq_release/extract-checksum.sh MD5 yq_linux_$(dpkg --print-architecture).tar.gz | awk '{ print $2 " " $1}' | md5sum -c -; \
  rm -rf /tmp/jq_release; \
  chmod +x /usr/local/bin/yq

# Grab gosu for easy step-down from root
ENV GOSU_VERSION 1.11
RUN set -ex; \
  wget -nv -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)"; \
  wget -nv -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc"; \
  export GNUPGHOME="$(mktemp -d)"; \
  for server in ha.pool.sks-keyservers.net $(shuf -e \
                          hkp://p80.pool.sks-keyservers.net:80 \
                          keyserver.ubuntu.com \
                          hkp://keyserver.ubuntu.com:80 \
                          pgp.mit.edu) ; do \
      gpg --batch --keyserver "$server" --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 && break || : ; \
  done && \
  gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \
  gpgconf --kill all; \
  rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc; \
  chmod +x /usr/local/bin/gosu; \
  gosu nobody true

# Prepare environment
ENV FLINK_HOME=/opt/flink
ENV PATH=$FLINK_HOME/bin:$PATH
RUN groupadd --system --gid=9999 flink && \
    useradd --system --home-dir $FLINK_HOME --uid=9999 --gid=flink flink
WORKDIR $FLINK_HOME

COPY --chown=flink:flink . $FLINK_HOME/

# Install Flink
RUN set -ex; \
  \
  # Replace default REST/RPC endpoint bind address to use the container's network interface \
  yq -i '.rest.address = "0.0.0.0"' FLINK_HOME/conf/config.yaml; \
  yq -i '.rest.bind-address = "0.0.0.0"' FLINK_HOME/conf/config.yaml; \
  yq -i '.jobmanager.bind-host = "0.0.0.0"' FLINK_HOME/conf/config.yaml; \
  yq -i '.taskmanager.bind-host = "0.0.0.0"' FLINK_HOME/conf/config.yaml; \
  yq -i '.taskmanager.host = "0.0.0.0"' FLINK_HOME/conf/config.yaml;

# Configure container
COPY  --from=entrypoint docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
EXPOSE 6123 8081
CMD ["help"]
