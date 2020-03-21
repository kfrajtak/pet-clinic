# Homework Submission

## Tasks
>- the services are configured using the “Configuration as code” approach
>    - figure out how, where is the configuration stored and find out how to modify it and try it (1 point)

TODO

>- the services must be started in proper order as stated in the readme file. However, docker-compose cannot guarantee the adequate start order 
>    - can you figure out how this can be achieved? (0.5 pt)

One can use [depends_on](https://docs.docker.com/compose/compose-file/#depends_on) which defines what service depends on what service.
That way when you need one service to start when using different service, define this dependency.
This however does not guarantee that the service `A` having dependency on `B` would start after the `B` is ready. 
In other words, docker-compose starts the container of `B` before starting container `A` but it does not care whether the service 
running inside that container is up and running - so the container `B` starts before the container `A`
but the server inside `A` can start before server in `B`.

To ensure service service startup order please read further.

>    - how can I make one service wait for others to start? (0.5 pt)

There are essentially two possible ways how to achieve waiting for the services.
 
 1. One can set [healthcheck](https://docs.docker.com/compose/compose-file/#healthcheck) which indicates whether the service is up and
  running and then set [restart_policy](https://docs.docker.com/compose/compose-file/#restart_policy) in a way that the service
  should be restarted always when it is unhealthy (basically using `on-failure` or `always` option). 
  In order to be able to set the order of 
  services startup one can create health check that would check different service - whether it is up and running -- this way,
  the waiting service will be restarted when its dependency is not up and running. **NOTE** that this is "hack" and should be used only
  when the following method can not be used for some esoteric reason. *(That being said I was once forced to use this and it is ugly as 
  you need to restart the service multiple times.)*
 
 1. Use external script which will check whether some service port is responding.
 If the port is responding, it is an indication that the service is running and then the service with dependency can start as well.
 I personally use modified [wait-for.sh](https://github.com/Eficode/wait-for) which is designed for the Alpine images.
 This project uses [dockerize](https://github.com/jwilder/dockerize) to achieve the same behavior. 

> the services are using an in-memory database; the data will be lost when the “network” is stopped

Here I would rephrase it into - the data are lost when the containers are stopped as each container uses its own in memory db.
Network in context of docker-compose is slightly different thing. 

> change whatever must be changed to stored data for each service in a separate database (1 point)
 
As there are resource files in the directory `mysql` and maven dependency on `mysql-connector-java`
 I decided to use [MySQL](https://hub.docker.com/_/mysql) database. 
 All containers use the shared database. This can be of course changed in the multiple databases but it is not necessary.
 
## Extra tasks
> collect the logs (1 point) (and eventually the metrics, 2 points) transparently and store them in ElasticSearch

TODO

> can you optimize the Dockerfile (docker/Dockerfile) so the image will be created only once? (2 points)

The thing is that this docker image is being built by the Spotify Docker client and therefore every service does not have its own
 Dockerfile. Quite interesting this is that you are not building the application inside the docker, but on the host machine 
 and then copying the final jar to the docker image... That is at least quite unusual.
 However, to optimize the build in a way that most stuff is taken from the cache (or cached file system layers), 
 I changed the position of arguments in a way that the Dockerize is downloaded just once when the first service is being built.
 Other steps are service dependent so they can't be optimized to share common steps with same arguments.
 
 What could be also done is to create new Docker Image and use it as a base for the `docker/Dockerfile` 
 such as this base contain Dockerize.
 Base (built with `docker build -t mybase .`) would look like this:
 ```dockerfile
FROM openjdk:8-jre-alpine

ARG DOCKERIZE_VERSION
ADD https://github.com/jwilder/dockerize/releases/download/${DOCKERIZE_VERSION}/dockerize-alpine-linux-amd64-${DOCKERIZE_VERSION}.tar.gz dockerize.tar.gz
RUN tar xzf dockerize.tar.gz
RUN chmod +x dockerize
``` 
And then the common Dockerfile:
```dockerfile
FROM mybase

VOLUME /tmp

ARG ARTIFACT_NAME
ADD ${ARTIFACT_NAME}.jar /app.jar
RUN touch /app.jar

ARG EXPOSED_PORT
EXPOSE ${EXPOSED_PORT}

ENV SPRING_PROFILES_ACTIVE docker
ENTRYPOINT ["java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```
However, I don't that this is necessary as the base docker image is really simple and while building all images,
 thanks to the [aufs](https://en.wikipedia.org/wiki/Aufs), is heavy cached.
