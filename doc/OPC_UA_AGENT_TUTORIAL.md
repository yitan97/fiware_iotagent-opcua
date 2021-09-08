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
>**Nota:** Para ver logs omitir -d en el comando anterior, o bien ejecutar
 ```sh
 $ docker-compose logs -f
 ```
### Error docker-compose up
Al realizar el comando `docker-compose up -d` se provocan varios errores
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

## Dispositivos en iotAgent
Para comprobar los dispositivos vinculados al iotAgent desde la terminal, introducir:
```sh
curl http://localhost:4001/iot/devices \
     -H "fiware-service: opcua_car" \
     -H "fiware-servicepath: /demo"
```
Devolverá algo similar a:
```json
{"count":1,"devices":[{"device_id":"age01_Car","service":"opcua_car","service_path":"/demo","entity_name":"age01_Car","entity_type":"Device","endpoint":"opc.tcp://iotcarsrv:5001/UA/CarServer","attributes":[{"object_id":"Acceleration","name":"Acceleration","type":"Number"},{"object_id":"EngineStopped","name":"EngineStopped","type":"Boolean"},{"object_id":"Engine_Temperature","name":"Engine_Temperature","type":"Number"},{"object_id":"Engine_Oxigen","name":"Engine_Oxigen","type":"Number"},{"object_id":"DataBlocksGlobal_3_dbRfidCntr_3_ID1_3_xBusy","name":"DataBlocksGlobal_3_dbRfidCntr_3_ID1_3_xBusy","type":"String"},{"object_id":"DataBlocksGlobal_3_dbRfidCntr_3_ID1_3_xBusyStatus","name":"DataBlocksGlobal_3_dbRfidCntr_3_ID1_3_xBusyStatus","type":"Boolean"},{"object_id":"DataBlocksGlobal_3_dbRfidCntr_3_ID1_3_xDone","name":"DataBlocksGlobal_3_dbRfidCntr_3_ID1_3_xDone","type":"String"},{"object_id":"DataBlocksGlobal_3_dbRfidCntr_3_ID1_3_xDoneStatus","name":"DataBlocksGlobal_3_dbRfidCntr_3_ID1_3_xDoneStatus","type":"Boolean"}],"lazy":[],"commands":[{"object_id":"ns=3;s=Stop","name":"Stop","type":"command"},{"object_id":"ns=3;s=Accelerate","name":"Accelerate","type":"command"}],"static_attributes":[]}]}
```

Se puede comprobar los datos en la base de datos MongoDb en mongo, por ejemplo con MongoDB Compass.
Hay que conectarse a `localhost:27017`, como se puede observar, hay un documento creado para *age01_Car*

![MongoDB_Compass age01_Car](img/20210907_200432.png)




