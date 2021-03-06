ARG APP_VERSION
ARG SBT_VERSION=1.3.6
ARG SBT_GPG_KEY=99E82A75642AC823
ARG DOCKERIZE_VERSION=0.6.1
ARG BASE_HREF="/"
ARG DOCS_URL="https://zoonavigator.elkozmon.com/en/latest"
ARG DEBUG=0

FROM ubuntu:18.04 as dockerize
MAINTAINER Lubos Kozmon <contact@elkozmon.com>

ARG DOCKERIZE_VERSION

# Install dependencies
RUN apt-get update \
  && apt-get install -y curl

# Get dockerize
RUN curl \
    -Lo dockerize-alpine-linux-amd64-v${DOCKERIZE_VERSION}.tar.gz \
    https://github.com/jwilder/dockerize/releases/download/v${DOCKERIZE_VERSION}/dockerize-alpine-linux-amd64-v${DOCKERIZE_VERSION}.tar.gz \
  && tar xzvf dockerize-alpine-linux-amd64-v${DOCKERIZE_VERSION}.tar.gz -C /usr/local/bin \
  && rm dockerize-alpine-linux-amd64-v${DOCKERIZE_VERSION}.tar.gz

FROM node:13.6.0-stretch-slim as npm

ARG APP_VERSION
ARG BASE_HREF
ARG DOCS_URL
ARG DEBUG

# Validate base href
RUN echo ${BASE_HREF} | grep -Eq '^\/(.*\/)?$' || (>&2 echo "BASE_HREF must start and end with a forward slash" && exit 1)

# Copy source code
WORKDIR /src
COPY /zoonavigator-web .
COPY /build/docker/files/zoonavigator-web .

# Install required packages
RUN apt-get update \
  && apt-get install -y python-dev make g++

# Get dockerize
COPY --from=dockerize /usr/local/bin/dockerize /usr/local/bin/dockerize

# Create env config for web app
RUN dockerize \
    -template ./environment.ts.template:./src/environments/environment.prod.ts

# Install dependencies & build
RUN npm install -g @angular/cli \
  && npm install \
  && ng build --prod --base-href ${BASE_HREF} \
  && mv dist /app

FROM openjdk:11-jdk-slim as sbt

ARG SBT_VERSION
ARG SBT_GPG_KEY

# Install required packages
RUN apt-get update \
  && apt-get install -y zip

# Copy source files
WORKDIR /src
COPY /zoonavigator-api .
COPY --from=npm /app ./play/public

# Install sbt
RUN apt-get update \
  && apt-get install -y aria2 gnupg ca-certificates \
  && aria2c -x4 "https://github.com/sbt/sbt/releases/download/v${SBT_VERSION}/sbt-${SBT_VERSION}.tgz" \
  && aria2c -x4 "https://github.com/sbt/sbt/releases/download/v${SBT_VERSION}/sbt-${SBT_VERSION}.tgz.asc" \
  && gpg --keyserver keyserver.ubuntu.com --recv-keys ${SBT_GPG_KEY} \
  && gpg --verify sbt-${SBT_VERSION}.tgz.asc sbt-${SBT_VERSION}.tgz \
  && tar xvfz sbt-${SBT_VERSION}.tgz -C /usr/local \
  && ln -s /usr/local/sbt/bin/sbt /usr/bin/sbt

# Build project
RUN sbt play/dist \
  && VERSION=$(ls play/target/universal/zoonavigator-play-*.zip | sed -E 's/.*zoonavigator-play-(.*).zip$/\1/') \
  && unzip play/target/universal/zoonavigator-play-$VERSION.zip \
  && mv zoonavigator-play-$VERSION /app

FROM ubuntu:18.04 as ubuntu

ARG BASE_HREF

# Get dockerize
COPY --from=dockerize /usr/local/bin/dockerize /usr/local/bin/dockerize

# Copy app files
WORKDIR /app
COPY --from=sbt /app .
COPY /build/docker/files/zoonavigator-api .
COPY /build/docker/files/zoonavigator-web .

# Create scripts and make them executable
RUN dockerize \
    -template ./run.sh.template:./run.sh \
    -template ./healthcheck.sh.template:./healthcheck.sh \
  && chmod +x \
    ./run.sh \
    ./healthcheck.sh

# Change permissions
RUN chgrp -R 0 . \
  && chmod -R g=u .

FROM openjdk:11-jdk-slim

# Default config
ENV HTTP_PORT=9000 \
    SESSION_TIMEOUT_MILLIS=3600000 \
    REQUEST_TIMEOUT_MILLIS=10000 \
    ZK_CLIENT_TIMEOUT_MILLIS=5000 \
    ZK_CONNECT_TIMEOUT_MILLIS=5000

# Install dependencies
RUN apt-get update \
  && apt-get install -y curl krb5-user \
  && rm -rf /var/lib/apt/lists/*

# Get dockerize
COPY --from=dockerize /usr/local/bin/dockerize /usr/local/bin/dockerize

# Change working directory
WORKDIR /app

# Change permissions
RUN chgrp -R 0 . \
  && chmod -R g=u .

# Copy app files
COPY --from=ubuntu /app .

# Add health check
HEALTHCHECK --interval=30s --timeout=3s \
    CMD ./healthcheck.sh

# Expose default HTTP port
EXPOSE 9000

ENTRYPOINT ["./run.sh"]
USER 1000
