# Note, when updating ther NiFi version, there are three places it must be changed.  The FROM clause in Stage 1 and the ARG NIFI_VERSION assignment in each stage.

FROM apache/nifi:1.13.0 as nifibase

############
## Stage 1 #
############

ARG NIFI_VERSION=1.13.0
ARG RELEASE_VERSION=32
ARG DIST_MIRROR=https://archive.apache.org/dist
ARG MAVEN_VERSION=3.6.2
ARG NEWRELIC_AGENT_VERSION=5.7.0

# Install Apache Maven
# Download New Relic agent library
RUN curl -o /tmp/maven.tar.gz ${DIST_MIRROR}/maven/maven-3/${MAVEN_VERSION}/binaries/apache-maven-${MAVEN_VERSION}-bin.tar.gz \
    && tar -C /tmp -xzf /tmp/maven.tar.gz \
    && /tmp/apache-maven-${MAVEN_VERSION}/bin/mvn dependency:copy -Dartifact=com.newrelic.agent.java:newrelic-agent:${NEWRELIC_AGENT_VERSION} -DoutputDirectory=/tmp \
    && rm /tmp/maven.tar.gz

############
## Stage 2 #
############

FROM registry.access.redhat.com/ubi8:8.4-211

ARG NIFI_VERSION=1.13.0
ARG RELEASE_VERSION=32
ARG DIST_MIRROR=https://archive.apache.org/dist
ARG MAVEN_VERSION=3.6.2
ARG NEWRELIC_AGENT_VERSION=5.7.0

LABEL name="nifi-docker"
LABEL vendor="IBM"
LABEL version=${NIFI_VERSION}
LABEL release=${RELEASE_VERSION}
LABEL summary="Apache NiFi and NiFi Toolkit"
LABEL description="Apache NiFi and Toolkit, including New Relic installed by Maven, and \
 Java 11 for HTTPS TLS protocol"

ENV NIFI_BASE_DIR /opt/nifi
ENV NIFI_HOME /opt/nifi/nifi-current
ENV NIFI_TOOLKIT_HOME /opt/nifi/nifi-toolkit-current
ENV NIFI_PID_DIR=${NIFI_HOME}/run
ENV NIFI_LOG_DIR=${NIFI_HOME}/logs

# Install package dependencies
# Setup NiFi user and create necessary directories
# Clear nifi-env.sh in favour of configuring all environment variables in the Dockerfile
RUN microdnf update -y && microdnf install yum && \
    yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm && \
    yum update -y && microdnf clean all

RUN microdnf -y install git rsync nano net-tools vim wget nmap jq xmlstarlet procps && \
    yum install -y java-11-openjdk-headless

COPY --chown=0:0 --from=nifibase $NIFI_BASE_DIR $NIFI_BASE_DIR

# copy status scripts for dependencies to be executed and add them to path
COPY status.sh ${NIFI_HOME}/bin

# Clear nifi-env.sh in favour of configuring all environment variables in the Dockerfile
RUN echo "#!/bin/sh\n" > ${NIFI_HOME}/bin/nifi-env.sh && \
    ln -s ${NIFI_HOME}/bin/* /usr/bin && \
    chmod -R g=u $NIFI_BASE_DIR

VOLUME ${NIFI_LOG_DIR} \
       ${NIFI_HOME}/conf \
       ${NIFI_HOME}/database_repository \
       ${NIFI_HOME}/flowfile_repository \
       ${NIFI_HOME}/content_repository \
       ${NIFI_HOME}/provenance_repository \
       ${NIFI_HOME}/state

# Web HTTP(s) & Socket Site-to-Site Ports
EXPOSE 8080 8443 10000 8000

WORKDIR ${NIFI_HOME}

# Apply configuration and start NiFi
#
# We need to use the exec form to avoid running our command in a subshell and omitting signals,
# thus being unable to shut down gracefully:
# https://docs.docker.com/engine/reference/builder/#entrypoint
#
# Also we need to use relative path, because the exec form does not invoke a command shell,
# thus normal shell processing does not happen:
# https://docs.docker.com/engine/reference/builder/#exec-form-entrypoint-example
ENTRYPOINT ["../scripts/start.sh"]