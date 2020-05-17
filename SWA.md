# SWA Úkol 1

## Tasks

 - the services are configured using the “Configuration as code” approach
   - figure out how, where is the configuration stored and find out how to modify it and try it 

Konkrétní konfigurace je v samostatném repozitáři odkud si ji pulluje [config server](https://cloud.spring.io/spring-cloud-config/multi/multi__spring_cloud_config_server.html) (ten pak konkrétní konfiguraci poskytuje jednotlivým službám). Abych konfiguraci mohl editovat tak jsem si udělal vlastní fork přislušného [repozitáře](https://github.com/Silaedru/spring-petclinic-microservices-config). Změna konfigurace je pak snadná - stačí upravit příslušný soubor podle názvu služby a ten do repozitáře pushnout. Pak je ještě potřeba systém restartovat, aby se změny projevily.

 - the services must be started in proper order as stated in the readme file. However, docker-compose cannot guarantee the adequate start order
   - can you figure out how this can be achieved? 
   - how can I make one service wait for others to start? 

docker-compose umožňuje definovat pořadí, v jakém bude služby spouštět (resp. vzájemné závislosti definovaných služeb) pomocí ["depends\_on"](https://docs.docker.com/compose/compose-file/#depends_on) - u tohoto projektu to už v `docker-compose.yml` je nastaveno. 

Čekání na dokončení spuštění konkrétní služby se v tomto projektu řeší pomocí nástroje [dockerize](https://github.com/jwilder/dockerize), opět už je to nakonfigurované. Obecně je to asi trochu složitější problém, protože "jak poznám, že je služba spuštěná?" Dockerize toto řeší tak, že zkouší vytvořit TCP spojení s cílovou adresou, a ve chvíli, kdy se to povede, tak službu považuje za připravenou.

 - the services are using an in-memory database; the data will be lost when the “network” is stopped
   - change whatever must be changed to stored data for each service in a separate database 

Upravil jsem `docker-compose.yml` a konfigurace jednotlivých služeb v [repozitáři s konfiguracemi](https://github.com/Silaedru/spring-petclinic-microservices-config). Též bylo potřeba trochu upravit SQL skripty, které vytvářejí schéma DB a vzorová data.

## Extra tasks
 - collect the logs (1 point) (and eventually the metrics, 2 points) transparently and store them in ElasticSearch

Nemám.

 - can you optimize the Dockerfile (docker/Dockerfile) so the image will be created only once? (2 points)

Jedna možná optimalizace je ve využití cache pro intermediate layer se staženým dockerize, o to jsem se pokusil. Zbytek jen zahrnuje zkopírování zkompilovaného jar do kontejneru a trochu docker omáčky, tam moc místa pro zlepšení nevidím.
