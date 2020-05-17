FROM openjdk:8-jre-alpine
VOLUME /tmp
ARG DOCKERIZE_VERSION

RUN wget -O dockerize.tar.gz https://github.com/jwilder/dockerize/releases/download/${DOCKERIZE_VERSION}/dockerize-alpine-linux-amd64-${DOCKERIZE_VERSION}.tar.gz
RUN tar xzf dockerize.tar.gz
RUN chmod +x dockerize
# For most commands, if the text of the command hasn’t changed, the version from the cache will be used. thats why we put unneded args after run
# https://pythonspeed.com/articles/docker-caching-model/
ARG ARTIFACT_NAME
ARG EXPOSED_PORT
ENV SPRING_PROFILES_ACTIVE docker

ADD ${ARTIFACT_NAME}.jar /app.jar

EXPOSE ${EXPOSED_PORT}

ENTRYPOINT ["java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
