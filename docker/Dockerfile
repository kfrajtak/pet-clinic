FROM openjdk:8-jre-alpine

ARG DOCKERIZE_VERSION

RUN wget https://github.com/jwilder/dockerize/releases/download/${DOCKERIZE_VERSION}/dockerize-alpine-linux-amd64-${DOCKERIZE_VERSION}.tar.gz -O dockerize.tar.gz && \
 tar xzf dockerize.tar.gz && \
 chmod +x dockerize

VOLUME /tmp

ARG ARTIFACT_NAME
ARG EXPOSED_PORT
ENV SPRING_PROFILES_ACTIVE docker

ADD ${ARTIFACT_NAME}.jar /app.jar

EXPOSE ${EXPOSED_PORT}

ENTRYPOINT ["java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
