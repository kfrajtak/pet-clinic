# SWA - HW1

**Student(KOS username) - ```matijmic```**

**The services are configured using the “Configuration as code” approach
figure out how, where is the configuration stored and find out how to modify it and try it (1 point).**

Configuration we can see in ```.yml``` files such as ```boostrap.yml```, where we can change uri, ports, profiles, names of applications etc.

Next configuration we can see in this repo. https://github.com/spring-petclinic/spring-petclinic-microservices-config
I placed files from repo to configuration folder in project.

**The services must be started in proper order as stated in the readme file. However, docker-compose cannot guarantee the adequate start order can you figure out how this can be achieved? (0.5 pt)**

Order of the service startup can be controlled by depends_on. Then docker compose always starts containers in dependency order, where dependencies are determined by depends_on, links, volums_from and network_mode. 

```
services:
  config-server:
    image: springcommunity/spring-petclinic-config-server
    container_name: config-server
    mem_limit: 512M
    ports:
     - 8888:8888

  discovery-server:
    image: springcommunity/spring-petclinic-discovery-server
    container_name: discovery-server
    mem_limit: 512M
    depends_on:
      - config-server
    entrypoint: ["./dockerize","-wait=tcp://config-server:8888","-timeout=60s","--","java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
    ports:
     - 8761:8761
```

Depends_on doesn't ensure that one container wait until another container is "ready". How resolve it? See below.

**How can I make one service wait for others to start? (0.5 pt)**

I can use tools such as ```wait-for-it, dockerize or wait-for```. These are small wrapper scripts which you can include in your application’s image to poll a given host and port until it’s accepting TCP connections. For exemple
```
version: "2"
services:
  web:
    build: .
    ports:
      - "80:8000"
    depends_on:
      - "db"
    command: ["./wait-for-it.sh", "db:5432", "--", "python", "app.py"]
  db:
    image: postgres
```
Container is waiting for db on port 5432. But it have limitation because container is waiting until port is ready but database need not be really ready. Next we can write own wrapper script to perfom a more application-specific health check.

**The services are using an in-memory database; the data will be lost when the “network” is stopped change whatever must be changed to stored data for each service in a separate database (1 point)**

In Dockerfile I added profile mysql
```
ENV SPRING_PROFILES_ACTIVE docker,mysql
```
In ```docker-compose.yml``` I added mysql with depends_on in customer, vets and visits services.
```
mysql-db:
    image: mysql:5.7.8
    container_name: mysql-db
    mem_limit: 512M
    environment:
      - MYSQL_ROOT_PASSWORD=petclinic
      - MYSQL_DATABASE=petclinic
    ports:
      - 3306:3306
```

***Collect the logs (1 point) (and eventually the metrics, 2 points) transparently and store them in ElasticSearch***

I added ```docker-elk``` folder for logs. First, I started fluentd, elasticsearch and kibana and wait until services were ready. To each services in docker-compose.yml I added
```
logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
```
Next run ```docker-compose.yml``` and see logs in kibana.

For metrcis I added maven dependency to services
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-elastic</artifactId>
    <version>1.1.4</version>
</dependency>
```
And added property to ```application.yml``` in configuration folder.
```
management.metrics.export.elastic.host: https://elasticsearch:9200
```

**Can you optimize the Dockerfile (docker/Dockerfile) so the image will be created only once? (2 points)**

I replaced download dockerize with local file and moved grafana and prometheus folders to ```docker-im``` folder.


