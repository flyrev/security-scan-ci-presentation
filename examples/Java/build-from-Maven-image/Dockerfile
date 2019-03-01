# Use these versions:
ARG JAVA_VERSION=8
ARG MAVEN_VERSION=3.5

FROM maven:${MAVEN_VERSION}-jdk-${JAVA_VERSION} as pom
RUN apt-get update
RUN apt-get -yqq install ca-certificates openssl
COPY certs/*.crt /usr/local/share/ca-certificates/
RUN update-ca-certificates
COPY maven-settings.xml /root/.m2/settings.xml

WORKDIR /app
COPY app/pom.xml pom.xml
# Set proxy, and silence Maven as much as possible.
# Use --build-arg VARIABLE=VALUE to set these values.
# A better alternative is to set proxy settings in maven-settings.xml and remove these lines.
ARG HTTP_PROXY_HOST=""
ARG HTTP_PROXY_PORT=80
ARG HTTPS_PROXY_HOST=""
ARG HTTPS_PROXY_PORT=80
ARG NON_PROXY_HOSTS="\"\""
ENV MAVEN_OPTS="-DproxySet=true -Dhttp.proxyHost=${HTTP_PROXY_HOST} -Dhttp.proxyPort=${HTTP_PROXY_PORT} -Dhttps.proxyHost=${HTTPS_PROXY_HOST} -Dhttps.proxyPort=${HTTPS_PROXY_PORT} -Dhttp.nonProxyHosts=${NON_PROXY_HOSTS} -Dproxy.protocol=http -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn -Dorg.slf4j.simpleLogger.showDateTime=true -Djava.awt.headless=true -Dsurefire.useSystemClassLoader=false"
ENV MAVEN_CLI_OPTS="--batch-mode --quiet --fail-fast"

FROM pom as dependency_check
# If you have dependency-check-maven in your pom.xml, you can replace "org.owasp:dependency-check-maven:5.0.0-M1" with "dependency-check-maven" in the following.
# 1) Update the dependency-check cache as a separate step. It is not strictly needed, but it is nice and explicit.
RUN mvn ${MAVEN_CLI_OPTS} org.owasp:dependency-check-maven:5.0.0-M1:update-only
# 2) Run the dependency check
RUN mvn ${MAVEN_CLI_OPTS} org.owasp:dependency-check-maven:5.0.0-M1:check

# Copy the source code itself
FROM pom as source
COPY app/src src

# Run test separately, effectively discarding any artifacts produced during the test phase
FROM source as test
RUN mvn ${MAVEN_CLI_OPTS} test

# Create a package (JAR)
FROM source as package
RUN mvn ${MAVEN_CLI_OPTS} package

# Use a jre image for running the application
FROM openjdk:${JAVA_VERSION}-jre-slim
WORKDIR /app
COPY --from=package /app/target/app.jar app.jar
CMD ["java", "-jar", "app.jar"]
