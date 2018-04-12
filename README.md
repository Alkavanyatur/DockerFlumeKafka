# Practica de Docker Flume Kafka

Vamos a añadir Kafka y FLume al archivo docker-compose.yml de la práctica.
Para que kafka funcione, tenemos además que añadir zookeeper.

# Parte 1

En esta parte vemos que el archivo es un docker-compose version 2
Están las imagenes de docker-ui y hadoop

```
version: "2.0"
services:
   curso-docker-ui:
    image: uifd/ui-for-docker
    container_name: docker-ui
    hostname: docker-ui
    privileged: true
    ports:
      - "9999:9000"
    volumes:
      -  /var/run/docker.sock:/var/run/docker.sock
  
   curso-hadoop:
     image: smizy/hadoop-base
     container_name: hadoop
     hostname: hadoop
     ports:
       - "50071:50071"
       - "50076:50076"
       - "8021:8021"
```

### Zookeeper

En esta parte añadimos zookeeper, especificando el puerto desde el que se conectará
```
   curso-zookeeper:
     image: jplock/zookeeper
     hostname: zookeeper
     ports:
       - "2188:2188"
     environment:
      ZOOKEEPER_CLIENT_PORT: 2188
      ZOOKEEPER_TICK_TIME: 2000
```

### Flume

Imagen de Flume, esta imagen la he clonado de docker-hub a mi docker local para volver a subirla a DockerHub.
En la parte de volumes se le indica que copie el archivo local practica_flume_kafka.conf a la carpeta /opt/flume-confir/flume.conf en la imagen.
En el environment se le indica el TOPIC de kafka, el cual adjunto.

```
   curso-flume:
     image: elorzaa/flume:v0.1
     container_name: flume
     hostname: flume
     ports:
       - "44441:44441"
       - "44442:44442"
       - "44443:44443"
     volumes:
       - ./practica_flume_kafka.conf:/opt/flume-config/flume.conf
     environment:
       FLUME_AGENT_NAME: "tweets_name"
```

### Kafka

Aquí ponemos la imagen de kafka y le añadimos la dependencia de zookeeper. El puerto 9093 y el puerto de zookeeper 2188 en el environment. También indicamos el topic de kafka y un Timeout de 36 segundos para que le de tiempo a conectarse a zookeeper.

```

   curso-kafka:
    image: wurstmeister/kafka
    depends_on:
      - curso-zookeeper
    container_name: kafka
    hostname: kafka
    ports:
      - "9093:9093"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: "kafka"
      KAFKA_ADVERTISED_PORT: "9093"
      KAFKA_ZOOKEEPER_CONNECT: "zookeeper:2188" 
      KAFKA_CREATE_TOPICS: "tweets_name"
      KAFKA_ZOOKEEPER_TIMEOUT_MS: 36000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

```

### Master

```
   curso-master:
     image: gettyimages/spark
     container_name: master
     hostname: master
     command: bin/spark-class org.apache.spark.deploy.master.Master -h master
     environment:
       MASTER: spark://master:7077
       SPARK_CONF_DIR: /conf
       SPARK_PUBLIC_DNS: localhost
     links:
       - curso-hadoop:curso-hadoop
       - curso-kafka
       - curso-flume
       - curso-zookeeper
     expose:
       - 7001
       - 7002
       - 7003
       - 7004
       - 7005
       - 7006
       - 7077
       - 6066
     ports:
       - 4040:4040
       - 6066:6066
       - 7077:7077
       - 8080:8080
     volumes:
       - ./conf/master:/conf
       - ./data:/tmp/data
```

### Worker

```
   curso-worker:
     image: gettyimages/spark
     hostname: worker
     command: bin/spark-class org.apache.spark.deploy.worker.Worker spark://master:7077
     environment:
       SPARK_CONF_DIR: /conf
       SPARK_WORKER_CORES: 1
       SPARK_WORKER_MEMORY: 2g
       SPARK_WORKER_PORT: 8881
       SPARK_WORKER_WEBUI_PORT: 8081
       SPARK_PUBLIC_DNS: worker
     links:
       - curso-master:curso-master
     expose:
       - 7012
       - 7013
       - 7014
       - 7015
       - 7016
       - 8881
     ports:
       - 8081
     volumes:
       - ./conf/worker:/conf
       - ./data:/tmp/data

```

### Red

Creamos una red privada para que los servicios no interfieran con los de otros repositorios ajenos a esta red.

```
networks:
  vnet:
    external:
      name: vnet
```


# Parte 2

### Descargamos las imagenes

```
docker-compose up -d
```

Esto descargará las imagenes, posteriormente nos ofrece un resultado como el siguiente:

```
Starting hadoop ... 
environment_curso-zookeeper_1 is up-to-date
docker-ui is up-to-date
flume is up-to-date
Starting hadoop ... done
Starting kafka ... done
environment_curso-worker_1 is up-to-date
```

Comprobamos que tenemos las imagenes descargadas y que las tenemos funcionando con:

```
docker images -a

docker ps -s
```

También al haber añadido docker-ui, podremos visualizar los contenedores en:

http://localhost:9999/

### Arrancamos Flume

```
docker run --name some_flume -e FLUME_AGENT_NAME=myagent -v /opt/flumetest/practica_flume_kafka.conf:/opt/flume-config/flume.conf -d elorzaa/flume:v0.1
```
44c442e0bc36107a16146894ba9b23fe08cacacc6a2c5c08ade6ac4d61e2a3a0


### Arrancamos el master y los worker

Ahora mismo deberíamos tener arrancado el master y un worker, concretamente "master" y "environment_curso-worker_1"

En caso de no ser así, podriamos levantarlos usando lo siguiente:

```
docker run -d --name master gettyimages/spark master master_id=$(docker inspect --format '{{ .NetworkSettings.IPAddress }}' master)
```

Y 

```
docker run -d --name environment_curso-worker_1 gettyimages/spark environment_curso-worker_1 spark://$master_ip:7077
```
Y para levantar el worker 2 y 3

```
docker run -d --name environment_curso-worker_2 gettyimages/spark environment_curso-worker_1 spark://$master_ip:7077

docker run -d --name environment_curso-worker_3 gettyimages/spark environment_curso-worker_1 spark://$master_ip:7077
```


# Parte 3

### Probamos todo

Entramos en master con el siguiente comando.

```
docker exec -it master bash
```

Y arrancamos kafka
```
bin/kafka-server-start.sh config/server.properties
```

Creamos un TOPIC
```
kafka-topics.sh --create --zookeeper $ZOOKEEPER --replication-factor 1 --partitions 2 --topic word-count
```

Creamos un producer

```
kafka-console-producer.sh --broker-list $KAFKA --topic word-count
```

Y Escribimos, deberíamos ver la salida en {...}



