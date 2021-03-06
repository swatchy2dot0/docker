FROM adoptopenjdk:11-jdk-openj9-bionic

RUN apt-get update \
  && apt-get upgrade -y \
  && apt-get install -y --no-install-recommends git curl gnupg fontconfig unzip nano libxss1 libappindicator1 libindicator7 wget locales \
  && curl -s https://packagecloud.io/install/repositories/github/git-lfs/script.deb.sh | bash && apt-get install -y git-lfs && git lfs install \
  && wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb \
  && apt-get install -y ./google-chrome*.deb \
  && rm -rf /var/lib/apt/lists/* \
  && locale-gen de_DE.UTF-8
  
ENV LANG de_DE.UTF-8  
ENV LANGUAGE de_DE:de  
ENV LC_ALL de_DE.UTF-8

ARG user=jenkins
ARG group=jenkins
ARG uid=1000
ARG gid=1000
ARG http_port=8080
ARG agent_port=50000
ARG JENKINS_HOME=/var/jenkins_home
ARG REF=/usr/share/jenkins/ref

ENV JENKINS_HOME $JENKINS_HOME
ENV JENKINS_SLAVE_AGENT_PORT ${agent_port}
ENV REF $REF

# Jenkins is run with user `jenkins`, uid = 1000
# If you bind mount a volume from the host or a data container,
# ensure you use the same uid
RUN mkdir -p $JENKINS_HOME \
  && chown ${uid}:${gid} $JENKINS_HOME \
  && groupadd -g ${gid} ${group} \
  && useradd -d "$JENKINS_HOME" -u ${uid} -g ${gid} -m -s /bin/bash ${user}

# Jenkins home directory is a volume, so configuration and build history
# can be persisted and survive image upgrades
VOLUME $JENKINS_HOME

RUN mkdir /download \
  && chmod -R 777 /download
  
VOLUME /download

RUN mkdir /download2 \
  && chmod -R 777 /download2
  
VOLUME /download2

RUN mkdir /documents \
  && chmod -R 777 /documents
  
VOLUME /documents

# $REF (defaults to `/usr/share/jenkins/ref/`) contains all reference configuration we want
# to set on a fresh new installation. Use it to bundle additional plugins
# or config file with your custom jenkins Docker image.
RUN mkdir -p ${REF}/init.groovy.d

# Use tini as subreaper in Docker container to adopt zombie processes
ARG TINI_VERSION=v0.16.1
COPY tini_pub.gpg ${JENKINS_HOME}/tini_pub.gpg
RUN curl -fsSL https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini-static-$(dpkg --print-architecture) -o /sbin/tini \
  && curl -fsSL https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini-static-$(dpkg --print-architecture).asc -o /sbin/tini.asc \
  && gpg --no-tty --import ${JENKINS_HOME}/tini_pub.gpg \
  && gpg --verify /sbin/tini.asc \
  && rm -rf /sbin/tini.asc /root/.gnupg \
  && chmod +x /sbin/tini

# Can be used to customize where jenkins.war get downloaded from
ARG JENKINS_URL=https://updates.jenkins.io/latest/jenkins.war

# could use ADD but this one does not check Last-Modified header neither does it allow to control checksum
# see https://github.com/docker/docker/issues/8331
RUN curl -fsSL ${JENKINS_URL} -o /usr/share/jenkins/jenkins.war

ENV JENKINS_UC https://updates.jenkins.io
ENV JENKINS_UC_EXPERIMENTAL=https://updates.jenkins.io/experimental
ENV JENKINS_INCREMENTALS_REPO_MIRROR=https://repo.jenkins-ci.org/incrementals
RUN chown -R ${user} "$JENKINS_HOME" "$REF"

ARG PLUGIN_CLI_URL=https://github.com/jenkinsci/plugin-installation-manager-tool/releases/download/2.2.0/jenkins-plugin-manager-2.2.0.jar
RUN curl -fsSL ${PLUGIN_CLI_URL} -o /usr/lib/jenkins-plugin-manager.jar

# for main web interface:
EXPOSE ${http_port}

# will be used by attached slave agents:
EXPOSE ${agent_port}

ENV COPY_REFERENCE_FILE_LOG $JENKINS_HOME/copy_reference_file.log

USER ${user}

COPY jenkins-support /usr/local/bin/jenkins-support
COPY jenkins.sh /usr/local/bin/jenkins.sh
COPY tini-shim.sh /bin/tini
COPY jenkins-plugin-cli.sh /bin/jenkins-plugin-cli

# set variables to create shared class cache
ENV JAVA_SCC_OPTS "-Xshareclasses:name=jenkins_scc,cacheDir=/tmp,enableBCI -Xscmx80M"
ENV JAVA_OLD_OPTS ${JAVA_OPTS}
ENV JAVA_OPTS "${JAVA_SCC_OPTS} ${JAVA_OPTS}"

# create shared class cache
RUN /sbin/tini -- /usr/local/bin/jenkins.sh > /dev/null & sleep 20

# revert JAVA_OPTS to not have shared class cache
ENV JAVA_SCC_OPTS "-Xshareclasses:name=jenkins_scc,cacheDir=/tmp,readonly"
ENV JAVA_OPTS "${JAVA_SCC_OPTS} ${JAVA_OLD_OPTS}"

ENTRYPOINT ["/sbin/tini", "--", "/usr/local/bin/jenkins.sh"]

# from a derived Dockerfile, can use `RUN install-plugins.sh active.txt` to setup ${REF}/plugins from a support bundle
COPY install-plugins.sh /usr/local/bin/install-plugins.sh
