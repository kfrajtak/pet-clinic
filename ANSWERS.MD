# Homework assignment 1:
##### 1) the services are configured using the “Configuration as code” approach
##### figure out how, where is the configuration stored and find out how to modify it and try it (1 point)

The configuration stored in external configuration repository:  ```https://github.com/spring-petclinic/spring-petclinic-microservices-config```
The configuration URI can be set in this file:
```pet-clinic/spring-petclinic-config-server/src/main/resources/bootstrap.yml```

To change configuration we could either change it in .yml files in configuration repository or create another external repo and link in in ```spring-petclinic-config-server/.../bootstrap.yml``` or another option is to create configuration folder locally with all configuration files

##### 2) the services must be started in proper order as stated in the readme file. However, docker-compose cannot guarantee the adequate start order
##### --- Can you figure out how this can be achieved? (0.5 pt)
You can control the order of service startup and shutdown with the depends_on option. Compose always starts and stops containers in dependency order, where dependencies are determined by depends_on, links, volumes_from, and network_mode: "service:...".
**However, for startup Compose does not wait until a container is “ready” (whatever that means for your particular application) - only until it’s running.**
You can work around the problem with a wrapper script:
Use a tool such as ```wait-for-it```, ```dockerize```, or sh-compatible ```wait-for```. These are small wrapper scripts which you can include in your application’s image to poll a given host and port until it’s accepting TCP connections
##### --- how can I make one service wait for others to start? (0.5 pt)
PetClinic application uses ```dockerize``` for these purposes. To control it go to ```docker-compose.yml``` and in entrypoint for each servise you can tell each servise what to wait for and for how long (timeout)
```entrypoint: ["./dockerize","-wait=tcp://discovery-server:8761","-wait=tcp://db-customers-service:3306", "-timeout=160s","--","java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]```
Further specification of parameters you can find here https://github.com/jwilder/dockerize  


##### 3) the services are using an in-memory database; the data will be lost when the “network” is stopped
##### change whatever must be changed to stored data for each service in a separate database (1 point)
We need to connect to Database to store the data. Db for each servise is added in ```docker-compose.yml``` and configured in ```application.properties``` of each servise 


### Extra tasks
##### 4) collect the logs (1 point) (and eventually the metrics, 2 points) transparently and store them in ElasticSearch
I attempted to collect logs and store them in ElasticSearch, but unfortunatelly something went wrong and I stopped at adding ElasticSearch in docker-compose due to deadline and lack of time. This task unfortunately is not done. 

##### 5) Can you optimize the Dockerfile (docker/Dockerfile) so the image will be created only once? (2 points)
We need to make use of cache. I've changed order of lines in Dockerfile to make it use cache better. The chages can be seen in diff
