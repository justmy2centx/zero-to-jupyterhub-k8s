# syntax = docker/dockerfile:1.3
# VULN_SCAN_TIME=2023-01-09_05:14:43


# The build stage
# ---------------
# This stage is building Python wheels for use in later stages by using a base
# image that has more pre-requisites to do so, such as a C++ compiler.
#
FROM python:3.11-bullseye as build-stage
# RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs -o rustup-init.sh


RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain 1.56.0
# RUN sh rustup-init.sh -y
ENV PATH="/root/.cargo/bin:$PATH"
# RUN rustup install 1.56.0


# Build wheels
#
# We set pip's cache directory and expose it across build stages via an
# ephemeral docker cache (--mount=type=cache,target=${PIP_CACHE_DIR}).
#
COPY requirements.txt requirements.txt

RUN pip install --upgrade pip
ARG PIP_CACHE_DIR=/tmp/pip-cache
RUN --mount=type=cache,target=${PIP_CACHE_DIR} \
    pip install build \
 && pip wheel \
        --wheel-dir=/tmp/wheels \
        -r requirements.txt \
        # Additional wheels for default-stage. Updates below should be repeated
        # in default-stage.
        #
        py-spy


# The final stage - slim version
# ------------------------------
# This stage is built and published as jupyterhub/k8s-hub-slim. It is meant to
# provide no non-essential packages.
#
FROM python:3.11-slim-bullseye as slim-stage
ENV DEBIAN_FRONTEND=noninteractive
ARG ADMIN_PASSWORD=jupyterhub003
RUN apt-get update && apt-get install -y --no-install-recommends \
    rustc \
    cargo


ARG ADMIN_USER=admin \
    ADMIN_UID=1000 \
    HOME=/home/admin
RUN useradd -m -s /bin/bash admin -p "$(openssl passwd -1 ${ADMIN_PASSWORD})"

ARG NB_USER=jovyan \
    NB_UID=1001 \
    HOME=/home/jovyan
RUN adduser \
        --disabled-password \
        --gecos "Default user" \
        --uid ${NB_UID} \
        --home ${HOME} \
        --force-badname \
        ${NB_USER}

RUN apt-get update \
 && apt-get upgrade -y \
 && apt-get install -y --no-install-recommends \
        # requirement for pycurl
        libcurl4 \
        # requirement for using a local sqlite database
        sqlite3 \
        tini \
 && rm -rf /var/lib/apt/lists/*

# install wheels built in the build-stage
COPY requirements.txt /tmp/requirements.txt
RUN pip install --upgrade pip
# RUN pip install rust
ARG PIP_CACHE_DIR=/tmp/pip-cache
RUN --mount=type=cache,target=${PIP_CACHE_DIR} \
    --mount=type=cache,from=build-stage,source=/tmp/wheels,target=/tmp/wheels \
    pip install \
        --find-links=/tmp/wheels/ \
        -r /tmp/requirements.txt

WORKDIR /srv/jupyterhub
RUN chown ${NB_USER}:${NB_USER} /srv/jupyterhub
USER ${NB_USER}

EXPOSE 8081
ENTRYPOINT ["tini", "--"]
CMD ["jupyterhub", "--config", "/usr/local/etc/jupyterhub/jupyterhub_config.py"]


# The final stage - default version
# ---------------------------------
# We add a few non-critical packages on top of the slim version to provide some
# additional utility.
#
FROM slim-stage as default-stage

USER root
RUN pip install --upgrade pip
# RUN pip install rust
RUN apt-get update && apt-get install -y --no-install-recommends \
    rustc \
    cargo


ARG PIP_CACHE_DIR=/tmp/pip-cache
RUN --mount=type=cache,target=${PIP_CACHE_DIR} \
    --mount=type=cache,from=build-stage,source=/tmp/wheels,target=/tmp/wheels \
    pip install \
        --find-links=/tmp/wheels/ \
        # Updates below should be repeated in build-stage.
        #
        # py-spy is useful for profiling performance of running hubs
        py-spy

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
        curl \
        dnsutils \
        git \
        less \
        vim \
 && rm -rf /var/lib/apt/lists/*

USER ${NB_USER}
