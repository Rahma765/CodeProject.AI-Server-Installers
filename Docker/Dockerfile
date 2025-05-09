# CodeProject.AI Server 3.0+
#
# Docker file for x86_64 CPU enabled image
#
# docker run --name CodeProject.AI-Server -d \
#    -p 32168:32168 -p 32168:32168/udp \
#   --mount type=bind,source=C:\ProgramData\CodeProject\AI\docker/data,target=/etc/codeproject/ai \
#   --mount type=bind,source=C:\ProgramData\CodeProject\AI\docker/modules,target=/app/modules \
#   codeproject/ai-server
#
# ASSUMPTION: This is being built from the root directory of the solution. This
# is set in the build_docker script as the last param to the docker buildx command,
# namely the "../..". This takes us from /Installers/Docker to /. and then this
# image file proceeds


# INITIAL SETUP ===============================================================

# Note: we're before the first 'FROM' statement so these values are not available
# inside the build stages. If you wish to use these inside a stage, re-grab them 
# inside that stage
ARG UBUNTU_VERSION=22.04
ARG DOTNET_VERSION=9.0
ARG CPAI_VERSION

# It's important to pull the correct base for the target architecture, otherwise
# building this image on, say, an Arm64 machine will result in the base image
# being Arm64, not x86_64 as assumed.
FROM amd64/ubuntu:$UBUNTU_VERSION AS base

WORKDIR /app

# Replace the sudo command, which doesn't exist in this image, with a noop so
# our scripts, which do contain sudo calls, will work
RUN if [ ! -x "$(command -v sudo)" ] && [ ! type sudo 2>/dev/null ]; then echo "#!/bin/sh\n\${@}" > /usr/sbin/sudo; chmod +x /usr/sbin/sudo; fi

# Environment vars ------------------------------------------------------------

ENV ASPNETCORE_URLS=http://+:32168;http://+:5000

# The simple log format is easier on my brain than json
ENV LOGGING__CONSOLE__FORMATTERNAME=simple

# Magic that was being done by the Microsoft ASP.NET base image that we aren't using anymore
ENV DOTNET_RUNNING_IN_CONTAINER=true

# noninteractive frontend means no prompts or questions are asked and whenever a call requires an
# answer, the default will be used. Installs will be non-interrupted and so won't hang. The Python
# installs, for instance, require this in this current environment
ENV DEBIAN_FRONTEND=noninteractive

# Grab the values passed in via command line here so we can use inside this stage
ARG UBUNTU_VERSION=22.04
ARG DOTNET_VERSION=9.0
ARG CPAI_VERSION

ENV CPAI_VERSION=$CPAI_VERSION
ENV UBUNTU_VERSION=$UBUNTU_VERSION
ENV DOTNET_VERSION=$DOTNET_VERSION

# This is so perl doesn't get upset
ENV LANGUAGE=C
ENV LC_ALL=C
ENV LANG=C
ENV LC_CTYPE=C

# Setup the ports -------------------------------------------------------------

EXPOSE 5000
EXPOSE 32168/tcp
EXPOSE 32168/udp

# Add some labels to the container --------------------------------------------

LABEL "Application"="CodeProject.AI Server"
LABEL "Publisher"="CodeProject Solutions Inc"
LABEL "Version"="$CPAI_VERSION"
LABEL "Target"="x64 Ubuntu $UBUNTU_VERSION CPU"
LABEL "Description"="CodeProject.AI Server with CPU-only processing."

# Install required libraries --------------------------------------------------

# *** TODO: *** all of this is in /app/server/install.sh. This script should be 
# called rather than having the same commands copy and pasted here.

# RUN apt-get update -y && apt-get upgrade -y --no-install-recommends apt-utils

# Install packages. In order:
#
# Required for SkiaSharp
#   libfontconfig1
# Required for System.Drawing
#   libgdplus
#   libjpeg-dev (maybe?)
#   zlib1g-dev (maybe?)
# Needed for opencv-python
#   ffmpeg libsm6 libxext6 libc6-dev
# So we can query glxinfo for GPU info
#   mesa-utils
# So we can install modules
#   curl jq unzip wget rsync ca-certificates (ca-certificates so we can do --no-check-certificate, jq for json parsing)
# So we can build the wheels of some modules (eg PaddleOCR)
#   patch
# This stops the "lsmod: not found" error
#   kmod
# This allows us to query process info
#   psmisc

RUN apt-get update -y && apt-get install -y --no-install-recommends \
    libfontconfig1  \
    libgdiplus      \
    libjpeg-dev     \
    zlib1g-dev      \
                    \
    ffmpeg          \
    libc6-dev       \
    libsm6          \
    libxext6        \
                    \
    mesa-utils      \
                    \
    ca-certificates \
    curl            \
    jq              \
    rsync           \
    unzip           \
    wget            \
                    \
    psmisc          \
                    \
    patch           \
                    \
    kmod            > /dev/null


# .NET -------------------------------------------------------------------------

# Update packages and install .NET
RUN apt install software-properties-common -y                               && \
    add-apt-repository ppa:dotnet/backports -y                              && \
    apt-get update && apt-get install aspnetcore-runtime-$DOTNET_VERSION -y > /dev/null


# Setup Python ----------------------------------------------------------------

# *** TODO: *** all of this is in /app/SDK/Scripts/utils.sh under "setupPython". 
# We should use that method rather than having the same commands copy and pasted 
# here. BEWARE that the list of apt installed here differs a little from what is
# installed via setupPython. installRequiredPythonPackages also installs some 
# tools from the list below

# Combining all these together into a single layer
# 1. Add deadsnakes repo
# 2. Install pip for python3,  python3-apt and python3-setuptools
#    Note that this (currently) installs python3.8 as a dependency. It could be
#    in the future that a different version of python is installed instead, in
#    which case we may need to pin the installed python version.
# 3. Install Python 3.8 as well as dev and distutils so packages can be built
#    if needed, and venv for creating virtual environments
# 4. Using the now installed Python 3.8, upgrade the python setuptools and dev
#    packages to the latest
# 5. Install virtualenv virtualenvwrapper for python 3.8

RUN apt-get install software-properties-common -y &&      \
    add-apt-repository ppa:deadsnakes/ppa -y      &&      \
    apt-get install -y --no-install-recommends            \
        python3-pip                                       \
        python3-apt                                       \
        python3-setuptools                        &&      \
    apt-get install -y --no-install-recommends            \
        python3.8                                 &&      \
    apt-get install -y --no-install-recommends            \
        python3.8-distutils                               \
        python3.8-dev                                     \ 
        python3.8-venv                            &&      \
    python3.8 -m pip install --upgrade setuptools &&      \
    python3.8 -m pip install --upgrade pip        &&      \
    python3.8 -m pip install virtualenv virtualenvwrapper > /dev/null

# Python 3.9: Combining all these together into a single layer, but leveraging
# the work done in Python 3.8:
# 1. Install Python 3.9 as well as dev and distutils so packages can be built
#    if needed, and venv for creating virtual environments
# 4. Using the now installed Python 3.8, upgrade the python setuptools and dev
#    packages to the latest
# 5. Install virtualenv virtualenvwrapper for python 3.8

RUN apt-get install -y --no-install-recommends            \
        python3.9                                 &&      \
    apt-get install -y --no-install-recommends            \
        python3.9-distutils                               \
        python3.9-dev                                     \ 
        python3.9-venv                            &&      \
    python3.9 -m pip install --upgrade setuptools &&      \
    python3.9 -m pip install --upgrade pip        &&      \
    python3.9 -m pip install virtualenv virtualenvwrapper > /dev/null

# Install Python3.11, distutils and dev tools (so packages can be built if needed)
# and pip install so we can install python packages. We don't need this for the
# built in modules, but we know we have downloadable modules that could use Python3.11
# so we preempt the need.
RUN apt-get install -y --no-install-recommends             \
        python3.11                                &&       \
    apt-get install -y --no-install-recommends             \
        python3.11-distutils                               \
        python3.11-dev                                     \ 
        python3.11-venv                           &&       \
    python3.11 -m pip install --upgrade setuptools &&      \
    python3.11 -m pip install --upgrade pip        &&      \
    python3.11 -m pip install virtualenv virtualenvwrapper > /dev/null


# Build the .NET source in a separate, discardable build container --------------

FROM mcr.microsoft.com/dotnet/sdk:$DOTNET_VERSION AS build

# Grab the values passed in via command line here so we can use inside this stage
# See https://docs.docker.com/engine/reference/builder/#scope
ARG UBUNTU_VERSION=22.04
ARG DOTNET_VERSION=9.0
ARG CPAI_VERSION
ARG REPO_NAME
ARG MODULES_REPO_NAME=CodeProject.AI-Modules

ENV CPAI_VERSION=$CPAI_VERSION
ENV UBUNTU_VERSION=$UBUNTU_VERSION
ENV DOTNET_VERSION=$DOTNET_VERSION
ENV REPO_NAME=$REPO_NAME
ENV MODULES_REPO_NAME=$MODULES_REPO_NAME

# REVIEW: [Matthew] This seems way too messy.
# 1. We should do a "cd / && mkdir CodeProject.AI" so we have a clean folder that's ours
#    Instead of COPY src/server/ src/server/ it will be COPY src/server/ CodeProject.AI/server/
#    WORKDIR /src would be WORKDIR /CodeProject.AI 
#    WORKDIR /src/src/server would be /CodeProject.AI/src/server
# 2. We don't need to copy any of the Python stuff to this .NET build server. We should
#    only copy ObjectDetectionYOLOv5Net, not the other modules
#    So, in the "FROM base AS final" stage we would do copy over the Python modules:
#    COPY ["src/modules/FaceProcessing/", "/app/preinstalled-modules/FaceProcessing"]
#    RUN mv  -f "/app/preinstalled-modules/FaceProcessing/modulesettings.docker.build.json" \
#               "/app/preinstalled-modules/FaceProcessing/modulesettings.docker.json"    && \
#        rm -rf /app/preinstalled-modules/FaceProcessing/intelligencelayer/__pycache__   && \
#        rm -rf /app/preinstalled-modules/FaceProcessing/bin                             && \
#        rm -f /app/preinstalled-modules/FaceProcessing/*.zip        \
#              /app/preinstalled-modules/FaceProcessing/package.bat  \
#              /app/preinstalled-modules/FaceProcessing/*.pyprog     > /dev/null
# 3. Similarly, we shouldn't copy over the .NET assets. ie:
#    Do not do in the build stage
#      COPY [ "/src/modules/ObjectDetectionYOLOv5Net/assets", "/app/publish/preinstalled-modules/ObjectDetectionYOLOv5Net/assets" 
#    Instead, in the last stage:
#      COPY [ "/src/modules/ObjectDetectionYOLOv5Net/assets", "/app/preinstalled-modules/ObjectDetectionYOLOv5Net/assets" 


# change directory to /src in the build image
WORKDIR /src

# Copy only the bits we need to for the .NET compilation steps
COPY /${REPO_NAME}/src/server/                        src/server/
COPY /${REPO_NAME}/src/SDK/                           src/SDK/
COPY /${REPO_NAME}/src/scripts/                       src/scripts/
COPY /${REPO_NAME}/modules/ObjectDetectionYOLOv5Net/  modules/ObjectDetectionYOLOv5Net/

# Build and publish Server
WORKDIR "/src/src/server"
RUN dotnet publish "Server.csproj" -c Release --no-self-contained --force -o /app/publish/server

# Build and publish .NET modules, copy over docker-specific modulesettings, and 
# then copy over models

# NOTE: For modules that we're installing directly into the Docker image, we will
#       use the /preinstalled-modules folder, not the /modules folder. This is
#       because we are asking users to mount a folder on their host machine mapped
#       to the /modules folder in this docker image. When they do that, anything
#       in the /modules folder in this image disappears and is replaced by the 
#       contents of the folder they mounted.
#
#       Two solutions:
#         1. We install pre-installed modules in a different folder (this is what
#            we're doing). The modules can still be uninstalled, and if
#            re-installed, they will be installed in the modules directory as 
#            "not preinstalled".
#         2. We work out how to mount a folder in a way that makes the stuff we've
#            already installed still be visible, while also allowing us to add
#            installed modules to the folder. This  probably involves volumes

WORKDIR "/src/modules/ObjectDetectionYOLOv5Net"
RUN dotnet publish ObjectDetectionYOLOv5Net.csproj -c Release --no-self-contained /p:DefineConstants=GPU_NONE \
                   --force -o /app/publish/preinstalled-modules/ObjectDetectionYOLOv5Net/bin &&       \
    mv -f /app/publish/preinstalled-modules/ObjectDetectionYOLOv5Net/bin/modulesettings.*             \
          /app/publish/preinstalled-modules/ObjectDetectionYOLOv5Net/                        &&       \
    mv -f /app/publish/preinstalled-modules/ObjectDetectionYOLOv5Net/modulesettings.docker.build.json \
          /app/publish/preinstalled-modules/ObjectDetectionYOLOv5Net/modulesettings.docker.json > /dev/null

# BEGIN REVIEW ================================================================

# REVIEW: [Matthew] This section should go in the "FROM base AS final" part, not
#                   here. We're copying the orig code to the build container and
#                   then from build to final. Should just copy from code to final.

# TODO: Grab these assets from our S3 bucket. Otherwise we have to ensure we've done a Dev setup on
#       the local machine first, to ensure these files are present
COPY [ "/${REPO_NAME}/modules/ObjectDetectionYOLOv5Net/assets",        "/app/publish/preinstalled-modules/ObjectDetectionYOLOv5Net/assets" ]
COPY [ "/${REPO_NAME}/modules/ObjectDetectionYOLOv5Net/custom-models", "/app/publish/preinstalled-modules/ObjectDetectionYOLOv5Net/custom-models" ]

# And ensure required assets are copied over. 
COPY [ "/${REPO_NAME}/modules/ObjectDetectionYOLOv5Net/explore.html", "/app/publish/preinstalled-modules/ObjectDetectionYOLOv5Net" ]

## Move non-compiled code (Python, script) into place --------------------------

WORKDIR /src

# We need some way to tell the server that for modules built into the Docker
# image, the venv will be in one place, but for modules downloaded at runtime, 
# the venv for the module will be in a different place.
#
# We do this by having a modulesettings.docker.build.json file that has the
# settings for modules pre-installed into the image. this build.json file is 
# copied into the image and renamed to just .docker.json at Docker build time,
# and then when the module loads up, everything works.
# 
# If the module were downloaded and installed during runtime, then the usual 
# modulesettings.linux.json would be used, and then modulesettings.docker.json
# would then be loaded (if it existed). The modulesettings.docker.build.json 
# file would be ignored.
#
# This enables us to have a modulesettings for inside a docker image, and module
# settings for modules outside the docker image.

COPY [ "/${MODULES_REPO_NAME}/CodeProject.AI-FaceProcessing/", "/app/publish/preinstalled-modules/FaceProcessing"]
RUN mv  -f "/app/publish/preinstalled-modules/FaceProcessing/modulesettings.docker.build.json" \
           "/app/publish/preinstalled-modules/FaceProcessing/modulesettings.docker.json"    && \
    rm -rf "/app/publish/preinstalled-modules/FaceProcessing/intelligencelayer/__pycache__" && \
    rm -rf "/app/publish/preinstalled-modules/FaceProcessing/bin"                           && \
    rm -f "/app/publish/preinstalled-modules/FaceProcessing/*.zip"        \
          "/app/publish/preinstalled-modules/FaceProcessing/package.bat"  \
          "/app/publish/preinstalled-modules/FaceProcessing/*.pyprog"     > /dev/null

COPY ["/${REPO_NAME}/modules/ObjectDetectionYOLOv5-6.2/", "/app/publish/preinstalled-modules/ObjectDetectionYOLOv5-6.2"]
RUN mv -f  "/app/publish/preinstalled-modules/ObjectDetectionYOLOv5-6.2/modulesettings.docker.build.json" \
           "/app/publish/preinstalled-modules/ObjectDetectionYOLOv5-6.2/modulesettings.docker.json" &&    \
    rm -rf "/app/publish/preinstalled-modules/ObjectDetectionYOLOv5-6.2/__pycache__"                &&    \
    rm -rf "/app/publish/preinstalled-modules/ObjectDetectionYOLOv5-6.2/bin"                        &&    \
    rm -f  "/app/publish/preinstalled-modules/ObjectDetectionYOLOv5-6.2/*.zip"       \
           "/app/publish/preinstalled-modules/ObjectDetectionYOLOv5-6.2/package.bat" \
           "/app/publish/preinstalled-modules/ObjectDetectionYOLOv5-6.2/*.pyprog"    > /dev/null

# SDK and install scripts
COPY ["/${REPO_NAME}/src/SDK/Python",        "/app/publish/SDK/Python"]
COPY ["/${REPO_NAME}/src/SDK/install.sh",    "/app/publish/SDK/install.sh"]
COPY ["/${REPO_NAME}/src/server/install.sh", "/app/publish/server/install.sh"]
COPY ["/${REPO_NAME}/src/scripts",           "/app/publish/scripts"]
COPY ["/${REPO_NAME}/src/setup.sh",          "/app/publish/setup.sh"]
COPY ["/${REPO_NAME}/.env",                  "/app/publish/.env"]

# Cleanup / setup
RUN rm -rf /app/publish/SDK/Python/__pycache__ \
    mkdir /app/publish/runtimes

# END REVIEW ===================================================================


# CREATE THE FINAL IMAGE ======================================================

FROM base AS final

# Note: UBUNTU_VERSION, DOTNET_VERSION and all ENV declared in 'base' are still here.

ARG REPO_NAME
ENV REPO_NAME=$REPO_NAME

# Move published server and modules into place --------------------------------

WORKDIR /app
COPY --from=build /app/publish .

# Install required Python packages --------------------------------------------

# *** TODO: *** all of this is in /app/SDK/Scripts/utils.sh under 
# "installRequiredPythonPackages". We should use that method rather than having
# the same commands copy and pasted here. NOTE that installRequiredPythonPackages
# installs some packages that, combined with the packages installed by setupPython,
# provide coverage for *almost* all the packages installed in this docker file. 
# When switching to using our scripts we need to ensure we cover all installs and
# test.

# *** WARNING ON ABOVE NOTE *** To save space in the Docker containers we don't
# create venv's for pre-installed/hardcoded modules. We install these modules
# directly into the system's python installation folders. This means that if we
# choose to use the utils.sh script for setting up python then we'd need to take
# this into account

# See https://towardsdatascience.com/how-to-shrink-numpy-scipy-pandas-and-matplotlib-for-your-data-product-4ec8d7e86ee4
# for a discussion on reducing PIP install sizes. That article is woefully out of date, with the
# --compile and --global-option now deprecated, the CLFAGS to nothing, but the --no-cache-dir does help

RUN python3.8 -m pip --no-cache-dir install \
    -r "/app/SDK/Python/requirements.txt" \
    -r "/app/preinstalled-modules/FaceProcessing/requirements.linux.txt" \
    -r "/app/preinstalled-modules/ObjectDetectionYOLOv5-6.2/requirements.linux.txt"

RUN python3.9 -m pip --no-cache-dir install \
    -r /app/SDK/Python/requirements.txt
    
# Add folders for storing persisted user data and modules. This should be mapped
# to a folder on the host. Typically C:\ProgramData\CodeProject\AI on Windows, 
# /etc/codeproject/ai on Linux, and /Library/Application Support/CodeProject/AI 
# on macOS.
# We also make a .vscode folder so we have some bits in place if we wish to run
# VSCode inside the container
RUN mkdir --parents /etc/codeproject/ai                 && \
    mkdir --parents /app/downloads/modules /app/modules && \
    mkdir /app/.vscode

# For debugging the live container using VS Code
COPY [ "/${REPO_NAME}/.vscode/launch.docker.json", "/app/.vscode/launch.json" ]
COPY [ "/${REPO_NAME}/.vscode/tasks.docker.json",  "/app/.vscode/tasks.json" ]

# Start the server
WORKDIR /app/server
ENTRYPOINT ["./CodeProject.AI.Server"]

# For debug
# ENTRYPOINT ["tail", "-f", "/dev/null"]