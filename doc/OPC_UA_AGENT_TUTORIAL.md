# OPC UA AGENT TUTORIAL 

## Clonar repositorio OCUA en github
Se crea una ruta donde se clonará el repositorio:
```sh
$ mkdir -p /$HOME/docker/fiware/
```
```sh
$ git clone "https://github.com/Engineering-Research-and-Development/iotagent-opcua"
```

## Ejecutar el Testbed
Para lanzar el Testbed ejecutar el comando:
```sh
$ docker-compose up -d
```
### Error docker-compose up
Al realizar el comando `docker-compose up -d` se probocan varios errores
1. Error en OrionCB
```sh
orion_1        | INFO@2021-09-06T18:14:51.452Z  contextBroker.cpp[1012]: start command line </usr/bin/contextBroker -fg -multiservice -ngsiv1Autocast -disableFileLog -statCounters -dbhost orion_mongo -logForHumans -logLevel INFO -t 255>
orion_1        | incompatible options: traceLevels cannot be used without setting -logLevel to DEBUG
```
Se observa que en el archivo `docker-compose.yml` en la sección del `orion/commad`, aparecen los comandos -statCounters -dbhost orion_mongo -logForHumans -logLevel INFO -t 255.
```yaml
  orion:
    hostname: orion
    image: fiware/orion:latest
    networks:
      - hostnet
      - ocbnet
    ports:
      - "1026:1026"
    depends_on:
      - orion_mongo
    command: -statCounters -dbhost orion_mongo -logForHumans -logLevel INFO -t 255
```

2. Error en iotAgent-OPCUA
por lo que el contenedor del `iotagen` se detiene.
El error lanzado es:
```sh
iotage_1       | npm ERR! missing script: start
iotage_1       | 
iotage_1       | npm ERR! A complete log of this run can be found in:
iotage_1       | npm ERR!     /root/.npm/_logs/2021-09-06T18_14_53_269Z-debug.log
iotagent-opcua_iotage_1 exited with code 1
```
Se observa que en el archivo `docker-compose.yml` en la sección del `iotage/commad`, aparece el comando npm start.
```yaml
iotage:
    hostname: iotage
    image: iotagent4fiware/iotagent-opcua:latest
    networks:
      - hostnet
      - iotnet
    ports:
      - "4001:4001"
      - "4081:8080"
    depends_on:
      - iotcarsrv
      - iotmongo
      - orion
    volumes:
      - ./AGECONF:/opt/iotagent-opcua/conf
      - ./certificates:/opt/iotagent-opcua/certificates
    command: npm start
    environment:
      - IOTA_REGISTRY_TYPE=mongodb #Whether to hold IoT device info in memory or in a database
      - IOTA_LOG_LEVEL=DEBUG # The log level of the IoT Agent
      - IOTA_MONGO_HOST=iot_mongo # The host name of MongoDB
      - IOTA_MONGO_DB=iotagent_opcua # The name of the database used in mongoDB
      - IOTA_CB_NGSI_VERSION=ld # use NGSI-LD when sending updates for active attributes
      - IOTA_JSON_LD_CONTEXT=https://uri.etsi.org/ngsi-ld/v1/ngsi-ld-core-context-v1.3.jsonld
      - IOTA_FALLBACK_TENANT=opcua_car
      - IOTA_RELAX_TEMPLATE_VALIDATION=true
```

Comentando en la sección `orion/command` la parte final `-t 255`, se solucionan los errores.
```yaml
  orion:
    hostname: orion
    image: fiware/orion:latest
    networks:
      - hostnet
      - ocbnet
    ports:
      - "1026:1026"
    depends_on:
      - orion_mongo
    command: -statCounters -dbhost orion_mongo -logForHumans -logLevel INFO #-t 255
```


Se procede a acceder al contenedor en cuestión.
Para ello se ejecutal el comando:
```sh
$ docker run --rm -it 


