# Start declaration of Build-Arg to determine where the image is getting built (DevOps agents or local)
ARG AGENTBUILD
ARG PYTHON_BASE_TAG
ARG PHP_BUILD_BASE_TAG
FROM debian:stretch-slim AS main
# End declaration of Build-Arg to determine where the image is getting built (DevOps agents or local)

# Configure locale (required for Python)
# NOTE: Do NOT move it from here as it could have global implications
ENV LANG C.UTF-8

# Install basic build tools
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        git \
        jq \
        make \
        unzip \
        # The tools in this package are used when installing packages for Python
        build-essential \
        # Required for ts
        moreutils \
        rsync \
        zip \
    && rm -rf /var/lib/apt/lists/*

RUN apt-get update && apt-get install -y curl
RUN apt-get install -y gnupg gnupg2

# Install .NET Core
FROM main AS dotnet-install
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        libc6 \
        libgcc1 \
        libgssapi-krb5-2 \
        libicu57 \
        liblttng-ust0 \
        libssl1.0.2 \
        libstdc++6 \
        zlib1g \
        # For .NET Core 1.1
        libcurl3 \
        libuuid1 \
        libunwind8 \
    && rm -rf /var/lib/apt/lists/*

ENV DOTNET_RUNNING_IN_CONTAINER=true \
    DOTNET_USE_POLLING_FILE_WATCHER=true \
    NUGET_XMLDOC_MODE=skip \
    DOTNET_SKIP_FIRST_TIME_EXPERIENCE=1 \
    NUGET_PACKAGES=/var/nuget

RUN mkdir /var/nuget
COPY build/__dotNetCoreSdkVersions.sh /tmp
COPY build/__dotNetCoreRunTimeVersions.sh /tmp
COPY images/build/installDotNetCore.sh /
RUN chmod +x /installDotNetCore.sh

RUN . /tmp/__dotNetCoreSdkVersions.sh && \
    DOTNET_SDK_VER=$DOT_NET_CORE_22_SDK_VERSION \
    DOTNET_SDK_SHA=$DOT_NET_CORE_22_SDK_SHA512 \
    /installDotNetCore.sh

RUN set -ex \
    rm -rf /tmp/NuGetScratch \
    && find /var/nuget -type d -exec chmod 777 {} \;

RUN set -ex \
 && sdksDir=/opt/dotnet/sdks \
 && cd $sdksDir \
 && ln -s 2.1 2

RUN set -ex \
 && dotnetDir=/opt/dotnet \
 && sdksDir=$dotnetDir/sdks \
 && runtimesDir=$dotnetDir/runtimes \
 && mkdir -p $runtimesDir \
 && cd $runtimesDir \
 && . /tmp/__dotNetCoreSdkVersions.sh \
 && . /tmp/__dotNetCoreRunTimeVersions.sh \
 # 2.2 sdk <-- 2.2 runtime's sdk
 && mkdir $NET_CORE_APP_22 \
 && ln -s $NET_CORE_APP_22 2.2 \
 && ln -s $sdksDir/$DOT_NET_CORE_22_SDK_VERSION $NET_CORE_APP_22/sdk \
 # LTS sdk <-- LTS runtime's sdk
 && ln -s 2.2 lts \
 && ltsSdk=$(readlink lts/sdk) \
 && ln -s $ltsSdk/dotnet /usr/local/bin/dotnet

# Install Node.js, NPM, Yarn
FROM main AS node-install
COPY build/__nodeVersions.sh /tmp
RUN chmod a+x /tmp/__nodeVersions.sh \
 && . /tmp/__nodeVersions.sh \
 && curl -sL https://git.io/n-install | bash -s -- -ny - \
 && ~/n/bin/n -d 10.14.2 \
 && ~/n/bin/n -d $NODE10_VERSION \
 && mv /usr/local/n/versions/node /opt/nodejs \
 && rm -rf /usr/local/n ~/

RUN set -e \
 && for ver in `ls /opt/nodejs`; do \
        nodeModulesDir="/opt/nodejs/$ver/lib/node_modules"; \
        npm_ver=`jq -r .version $nodeModulesDir/npm/package.json`; \
        if [ ! "$npm_ver" = "${npm_ver#6.}" ]; then \
            echo "Upgrading node $ver's npm version from $npm_ver to 6.9.0"; \
            cd $nodeModulesDir; \
            PATH="/opt/nodejs/$ver/bin:$PATH" \
            "$nodeModulesDir/npm/bin/npm-cli.js" install npm@6.9.0; \
            echo; \
        fi; \
    done
RUN set -ex \
 && for ver in `ls /opt/nodejs`; do \
        npm_ver=`jq -r .version /opt/nodejs/$ver/lib/node_modules/npm/package.json`; \
        if [ ! -d /opt/npm/$npm_ver ]; then \
            mkdir -p /opt/npm/$npm_ver; \
            ln -s /opt/nodejs/$ver/lib/node_modules /opt/npm/$npm_ver/node_modules; \
            ln -s /opt/nodejs/$ver/lib/node_modules/npm/bin/npm /opt/npm/$npm_ver/npm; \
            if [ -e /opt/nodejs/$ver/lib/node_modules/npm/bin/npx ]; then \
                chmod +x /opt/nodejs/$ver/lib/node_modules/npm/bin/npx; \
                ln -s /opt/nodejs/$ver/lib/node_modules/npm/bin/npx /opt/npm/$npm_ver/npx; \
            fi; \
        fi; \
    done

RUN set -ex \
 && . /tmp/__nodeVersions.sh \
 && GPG_KEY=6A010C5166006599AA17F08146C2130DFD2497F5 \
 && for i in {1..5}; do \
      gpg --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys "$GPG_KEY" || \
      gpg --keyserver hkp://ipv4.pool.sks-keyservers.net --recv-keys "$GPG_KEY" || \
      gpg --keyserver hkp://pgp.mit.edu:80 --recv-keys "$GPG_KEY"; \
      if [ $? -eq 0 ]; then break; fi \
    done \
 && curl -fsSLO --compressed "https://yarnpkg.com/downloads/$YARN_VERSION/yarn-v$YARN_VERSION.tar.gz" \
 && curl -fsSLO --compressed "https://yarnpkg.com/downloads/$YARN_VERSION/yarn-v$YARN_VERSION.tar.gz.asc" \
 && gpg --batch --verify yarn-v$YARN_VERSION.tar.gz.asc yarn-v$YARN_VERSION.tar.gz \
 && mkdir -p /opt/yarn \
 && tar -xzf yarn-v$YARN_VERSION.tar.gz -C /opt/yarn \
 && mv /opt/yarn/yarn-v$YARN_VERSION /opt/yarn/$YARN_VERSION \
 && rm yarn-v$YARN_VERSION.tar.gz.asc yarn-v$YARN_VERSION.tar.gz

RUN set -ex \
 && ln -s 10.14.2 /opt/nodejs/10.14 \
 && . /tmp/__nodeVersions.sh \
 && ln -s $NODE10_VERSION /opt/nodejs/$NODE10_MAJOR_MINOR_VERSION \
 && ln -s $NODE10_MAJOR_MINOR_VERSION /opt/nodejs/10 \
 && ln -s 10 /opt/nodejs/lts
RUN set -ex \
 && ln -s 6.9.0 /opt/npm/6.9 \
 && ln -s 6.9 /opt/npm/6 \
 && ln -s 6 /opt/npm/latest
RUN set -ex \
 && . /tmp/__nodeVersions.sh \
 && ln -s $YARN_VERSION /opt/yarn/stable \
 && ln -s $YARN_VERSION /opt/yarn/latest \
 && ln -s $YARN_VERSION /opt/yarn/$YARN_MINOR_VERSION \
 && ln -s $YARN_MINOR_VERSION /opt/yarn/$YARN_MAJOR_VERSION
RUN set -ex \
 && mkdir -p /links \
 && cp -s /opt/nodejs/lts/bin/* /links \
 && cp -s /opt/yarn/stable/bin/yarn /opt/yarn/stable/bin/yarnpkg /links


# This stage is used only when building locally
FROM dotnet-install AS buildscriptbuilder
COPY src/BuildScriptGenerator /usr/oryx/src/BuildScriptGenerator
COPY src/BuildScriptGeneratorCli /usr/oryx/src/BuildScriptGeneratorCli
COPY src/Common /usr/oryx/src/Common
COPY build/FinalPublicKey.snk usr/oryx/build/
COPY src/CommonFiles /usr/oryx/src/CommonFiles
# This statement copies signed oryx binaries from during agent build.
# For local/dev contents of blank/empty directory named binaries are getting copied
COPY binaries /opt/buildscriptgen/
WORKDIR /usr/oryx/src
ARG GIT_COMMIT=unspecified
ARG AGENTBUILD=${AGENTBUILD}
ARG BUILD_NUMBER=unspecified
ENV GIT_COMMIT=${GIT_COMMIT}
ENV BUILD_NUMBER=${BUILD_NUMBER}
COPY images/build/benv.sh /usr/local/bin/benv
RUN chmod +x /usr/local/bin/benv
RUN if [ -z "$AGENTBUILD" ]; then \
        dotnet publish -r linux-x64 -o /opt/buildscriptgen/ -c Release BuildScriptGeneratorCli/BuildScriptGeneratorCli.csproj; \
    fi
RUN chmod a+x /opt/buildscriptgen/GenerateBuildScript


###
# Build run script generators (to be used by the `oryx run-script` command)
###
FROM golang:1.11-stretch as startupScriptGens

# GOPATH is set to "/go" in the base image
WORKDIR /go/src
COPY src/startupscriptgenerator/src .

ARG GIT_COMMIT=unspecified
ARG BUILD_NUMBER=unspecified
ENV GIT_COMMIT=${GIT_COMMIT}
ENV BUILD_NUMBER=${BUILD_NUMBER}

RUN ./build.sh node       /opt/startupcmdgen/nodejs
###
# End build run script generators
###

FROM main AS final
WORKDIR /

COPY images/build/benv.sh /usr/local/bin/benv
RUN chmod +x /usr/local/bin/benv

# Grant read-write permissions to the nuget folder so that dotnet restore
# can write into it.

# Copy NodeJs, NPM and Yarn related content
COPY --from=node-install /opt /opt
COPY --from=node-install /links/ /usr/local/bin

# Build script generator content. Docker doesn't support variables in --from
# so we are building an extra stage to copy binaries from correct build stage
COPY --from=buildscriptbuilder /opt/buildscriptgen/ /opt/buildscriptgen/
RUN ln -s /opt/buildscriptgen/GenerateBuildScript /usr/local/bin/oryx

# Oryx depends on the run script generators for most of its
# `IProgrammingPlatform.GenerateBashRunScript()` implementations
COPY --from=startupScriptGens /opt/startupcmdgen/ /opt/startupcmdgen/

# Bake Application Insights key from pipeline variable into final image
ARG AI_KEY
ENV ORYX_AI_INSTRUMENTATION_KEY=${AI_KEY}

ARG GIT_COMMIT=unspecified
ARG BUILD_NUMBER=unspecified
LABEL com.microsoft.oryx.git-commit=${GIT_COMMIT}
LABEL com.microsoft.oryx.build-number=${BUILD_NUMBER}

FROM mcr.microsoft.com/dotnet/core/runtime-deps:2.2

COPY --from=final / /

ENV LANG=C.UTF-8 \
    TERM=xterm \
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
    DOTNET_RUNNING_IN_CONTAINER=true \
    DOTNET_USE_POLLING_FILE_WATCHER=true \
    NUGET_XMLDOC_MODE=skip \
    DOTNET_SKIP_FIRST_TIME_EXPERIENCE=1 \
    NUGET_PACKAGES=/var/nuge \
    LANG=C.UTF-8

ENTRYPOINT [ "benv" ]
