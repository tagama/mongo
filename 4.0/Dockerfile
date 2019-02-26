FROM ubuntu:xenial
ENV SUMMARY="MongoDB NoSQL database server" \
    DESCRIPTION="MongoDB (from humongous) is a free and open-source \
cross-platform document-oriented database program. Classified as a NoSQL \
database program, MongoDB uses JSON-like documents with schemas. This \
container image contains programs to run mongod server."

LABEL summary="$SUMMARY" \
      description="$DESCRIPTION" \
      io.k8s.description="$DESCRIPTION" \
      io.k8s.display-name="MongoDB 4" \
      io.openshift.expose-services="27017:mongodb" \
      io.openshift.tags="database,mongodb,mongodb4" \
      com.redhat.component="rh-mongodb4-container" \
      name="mongo:4-xenial" \
      version="1" \
      maintainer="SoftwareCollections.org <sclorg@redhat.com>"

ENV MONGODB_VERSION=4.0.6 \
    # Set paths to avoid hard-coding them in scripts.
    APP_DATA=/opt/app-root/src \
    HOME=/var/lib/mongodb \
    CONTAINER_SCRIPTS_PATH=/usr/share/container-scripts/mongodb \
    # Incantations to enable Software Collections on `bash` and `sh -i`.
    ENABLED_COLLECTIONS=mongo-org \
    BASH_ENV="\${CONTAINER_SCRIPTS_PATH}/scl_enable" \
    ENV="\${CONTAINER_SCRIPTS_PATH}/scl_enable" \
    PROMPT_COMMAND=". \${CONTAINER_SCRIPTS_PATH}/scl_enable"


RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		ca-certificates \
		jq \
		numactl \
	; \
	if ! command -v ps > /dev/null; then \
		apt-get install -y --no-install-recommends procps; \
	fi; \
	rm -rf /var/lib/apt/lists/*

# grab gosu for easy step-down from root (https://github.com/tianon/gosu/releases)
ENV GOSU_VERSION 1.10
# grab "js-yaml" for parsing mongod's YAML config files (https://github.com/nodeca/js-yaml/releases)
ENV JSYAML_VERSION 3.10.0

RUN set -ex; \
	\
	apt-get update; \
	apt-get install -y --no-install-recommends \
		wget \
	; \
	if ! command -v gpg > /dev/null; then \
		apt-get install -y --no-install-recommends gnupg dirmngr; \
	fi; \
	rm -rf /var/lib/apt/lists/*; \
	\
	dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')"; \
	wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch"; \
	wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch.asc"; \
	export GNUPGHOME="$(mktemp -d)"; \
	gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4; \
	gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \
	command -v gpgconf && gpgconf --kill all || :; \
	rm -r "$GNUPGHOME" /usr/local/bin/gosu.asc; \
	chmod +x /usr/local/bin/gosu; \
	gosu nobody true; \
	\
	wget -O /js-yaml.js "https://github.com/nodeca/js-yaml/raw/${JSYAML_VERSION}/dist/js-yaml.js"; \
# TODO some sort of download verification here
	\
	apt-get purge -y --auto-remove wget

RUN mkdir /docker-entrypoint-initdb.d

ENV GPG_KEYS 9DA31620334BD75D9DCB49F368818C72E52529D4
RUN set -ex; \
	export GNUPGHOME="$(mktemp -d)"; \
	for key in $GPG_KEYS; do \
		gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
	done; \
	gpg --batch --export $GPG_KEYS > /etc/apt/trusted.gpg.d/mongodb.gpg; \
	command -v gpgconf && gpgconf --kill all || :; \
	rm -r "$GNUPGHOME"; \
	apt-key list

# Allow build-time overrides (eg. to build image with MongoDB Enterprise version)
# Options for MONGO_PACKAGE: mongodb-org OR mongodb-enterprise
# Options for MONGO_REPO: repo.mongodb.org OR repo.mongodb.com
# Example: docker build --build-arg MONGO_PACKAGE=mongodb-enterprise --build-arg MONGO_REPO=repo.mongodb.com .
ARG MONGO_PACKAGE=mongodb-org
ARG MONGO_REPO=repo.mongodb.org
ENV MONGO_PACKAGE=${MONGO_PACKAGE} MONGO_REPO=${MONGO_REPO}

ENV MONGO_MAJOR 4.0
ENV MONGO_VERSION 4.0.6

RUN echo "deb http://$MONGO_REPO/apt/ubuntu xenial/${MONGO_PACKAGE%-unstable}/$MONGO_MAJOR multiverse" | tee "/etc/apt/sources.list.d/${MONGO_PACKAGE%-unstable}.list"

RUN set -x \
	&& apt-get update \
	&& apt-get install -y \
		${MONGO_PACKAGE}=$MONGO_VERSION \
		${MONGO_PACKAGE}-server=$MONGO_VERSION \
		${MONGO_PACKAGE}-shell=$MONGO_VERSION \
		${MONGO_PACKAGE}-mongos=$MONGO_VERSION \
		${MONGO_PACKAGE}-tools=$MONGO_VERSION \
	&& rm -rf /var/lib/apt/lists/* \
	&& rm -rf /var/lib/mongodb \
	&& mv /etc/mongod.conf /etc/mongod.conf.orig

COPY s2i/bin/ $STI_SCRIPTS_PATH
COPY root /

#RUN mkdir -p /data/db /data/configdb \
#	&& chown -R 0 /data/db /data/configdb
#	&& chmod -R g+rwX /data/db /data/configdb

RUN : > /etc/mongod.conf && \
    mkdir -p ${HOME}/data && \
    # Set owner 'mongodb:0' and 'g+rw(x)' permission - to avoid problems running container with arbitrary UID
    /usr/libexec/fix-permissions /etc/mongod.conf ${CONTAINER_SCRIPTS_PATH}/mongod.conf.template \
    ${HOME} ${APP_DATA}/.. && \
    usermod -a -G root mongodb && \

VOLUME /data/db /data/configdb

COPY docker-entrypoint.sh /usr/local/bin/


EXPOSE 27017
ENTRYPOINT ["container-entrypoint"]
CMD ["run-mongod"]