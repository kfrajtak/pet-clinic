# Homework Submission

## Tasks
>- the services are configured using the “Configuration as code” approach
>    - figure out how, where is the configuration stored and find out how to modify it and try it (1 point)

This demo uses [Spring Cloud Config](https://cloud.spring.io/spring-cloud-config/reference/html/) -
in [this file](spring-petclinic-config-server/src/main/resources/bootstrap.yml). 

Then all services (i.e. `spring-petclinic-vets-service`) that use the configuration from this server must have specification where tho find such configuration
```yaml
spring:
  cloud:
    config:
      uri: http://config-server:8888
```

To change the configuration one can either change the targeted Github repository - original value is
[spring-petclinic-microservices-config](https://github.com/spring-petclinic/spring-petclinic-microservices-config).
or to use provided fallback by creating simple folder `repo-with-configuration` in the root of the project with all necessary configuration 
and then edit `docker-compose.yml`
```yaml
  config-server:
    image: springcommunity/spring-petclinic-config-server
    container_name: config-server
    mem_limit: 512M
    ports:
      - 8888:8888
    environment:
      - GIT_REPO=/config
    volumes:
    - ./repo-with-configuration:/config
```

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
 Another thing I did is replacing `ADD` by `RUN wget` - therefore the HTTP request for Dockerize will be executed just once.
 As `ADD` first download the file and then creates hash of the layer.
 `RUN` computes hash of the URL -> no file downloaded.  
 
 All previously described optimizations result in:
 ```bash
Step 1/12 : FROM openjdk:8-jre-alpine

 ---> f7a292bbb70c
Step 2/12 : VOLUME /tmp

 ---> Using cache
 ---> 674131380226
Step 3/12 : ARG DOCKERIZE_VERSION

 ---> Using cache
 ---> 4e71f5a79327
Step 4/12 : RUN wget -O dockerize.tar.gz https://github.com/jwilder/dockerize/releases/download/${DOCKERIZE_VERSION}/dockerize-alpine-linux-amd64-${DOCKERIZE_VERSION}.tar.gz

 ---> Using cache
 ---> f04603ca89a8
Step 5/12 : RUN tar xzf dockerize.tar.gz

 ---> Using cache
 ---> b3b45f125cc1
Step 6/12 : RUN chmod +x dockerize

 ---> Using cache
 ---> 06b6c870566e
Step 7/12 : ARG ARTIFACT_NAME

 ---> Using cache
 ---> 834646a63ccb
```
First step which can not be cached is copy of the produced `jar`. 
 
 What could be also done is to create new Docker Image and use it as a base for the `docker/Dockerfile` 
 such as this base contain Dockerize.
 Base (built with `docker build -t mybase .`) would look like this:
 ```dockerfile
FROM openjdk:8-jre-alpine

# Download dockerize and cache that layer
ARG DOCKERIZE_VERSION
RUN wget -O dockerize.tar.gz https://github.com/jwilder/dockerize/releases/download/${DOCKERIZE_VERSION}/dockerize-alpine-linux-amd64-${DOCKERIZE_VERSION}.tar.gz
# Extract layer
RUN tar xzf dockerize.tar.gz
RUN chmod +x dockerize
``` 
And then the common Dockerfile:
```dockerfile
FROM mybase

ARG ARTIFACT_NAME
ADD ${ARTIFACT_NAME}.jar /app.jar

ARG EXPOSED_PORT
EXPOSE ${EXPOSED_PORT}

ENV SPRING_PROFILES_ACTIVE docker

VOLUME /tmp
ENTRYPOINT ["java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```
However, I don't that this is necessary as the base docker image is really simple and while building all images,
 thanks to the [aufs](https://en.wikipedia.org/wiki/Aufs), is heavily cached.

To the question asked:
> so the image will be created only once

This requires more collaboration - essentially yes,
 that is possible by copying everything inside the Dockerfile and then building everything. 
 Then you have one image... However, this is at least unwise. 
 But is it somehow possible? That it will be build once but it will follow best practice?   
 
The answer to that question are docker multistage builds.
One can create base image that will be build exactly once.
The idea is to copy all modules to this particular image, build them all by using 
`mvn -Dmaven.test.skip=true install` - this will produce (with proper plugin set up) `fatJar`s of all projects.
Now, each project have "run" image, which would just copy proper jar from the base like this:
```dockerfile
FROM my-base-all-builds AS base

FROM openjdk:8-jre-alpine AS runtime
# Obtain dockerize
COPY --from=base /dockerize  /
# Obtain correct JAR
COPY --from=base /path/to/correct/service-jar/app.jar  /app.jar
# Set correct entry point
ENTRYPOINT ["java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```


## Original Repository
Funny thing, original repository had opened [issue](https://github.com/spring-petclinic/spring-petclinic-microservices/issues/136) - on 
fixing the dockerfile and to introduce caching of Dockerize - as I was able to do that in this homework, 
I created [pull request](https://github.com/spring-petclinic/spring-petclinic-microservices/pull/148) for that. 
