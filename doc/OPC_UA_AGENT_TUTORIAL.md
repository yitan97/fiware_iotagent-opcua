# OPC UA AGENT TUTORIAL 
-----

# Clonar repositorio OCUA en github
Se crea una ruta donde se clonará el repositorio:
```sh
$ mkdir -p /$HOME/docker/fiware/
```
```sh
$ git clone "https://github.com/Engineering-Research-and-Development/iotagent-opcua"
```

# Ejecutar el Testbed
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

## Ver dispositivos conectados a iotAgent
Para comprobar los dispositivos vinculados al iotAgent desde la terminal, introducir:
```sh
curl http://localhost:4001/iot/devices -H "fiware-service: opcua_car" -H "fiware-servicepath: /demo"
```
Devolverá algo similar a:
```json
{"count":1,"devices":[{"device_id":"age01_Car","service":"opcua_car","service_path":"/demo","entity_name":"age01_Car","entity_type":"Device","endpoint":"opc.tcp://iotcarsrv:5001/UA/CarServer","attributes":[{"object_id":"Acceleration","name":"Acceleration","type":"Number"},{"object_id":"EngineStopped","name":"EngineStopped","type":"Boolean"},{"object_id":"Engine_Temperature","name":"Engine_Temperature","type":"Number"},{"object_id":"Engine_Oxigen","name":"Engine_Oxigen","type":"Number"},{"object_id":"DataBlocksGlobal_3_dbRfidCntr_3_ID1_3_xBusy","name":"DataBlocksGlobal_3_dbRfidCntr_3_ID1_3_xBusy","type":"String"},{"object_id":"DataBlocksGlobal_3_dbRfidCntr_3_ID1_3_xBusyStatus","name":"DataBlocksGlobal_3_dbRfidCntr_3_ID1_3_xBusyStatus","type":"Boolean"},{"object_id":"DataBlocksGlobal_3_dbRfidCntr_3_ID1_3_xDone","name":"DataBlocksGlobal_3_dbRfidCntr_3_ID1_3_xDone","type":"String"},{"object_id":"DataBlocksGlobal_3_dbRfidCntr_3_ID1_3_xDoneStatus","name":"DataBlocksGlobal_3_dbRfidCntr_3_ID1_3_xDoneStatus","type":"Boolean"}],"lazy":[],"commands":[{"object_id":"ns=3;s=Stop","name":"Stop","type":"command"},{"object_id":"ns=3;s=Accelerate","name":"Accelerate","type":"command"}],"static_attributes":[]}]}
```

También se puede comprobar desde **postman**, como se muestra en la siguiente imagen

![20210921_185637](img/20210921_185637.png)





Se puede comprobar los datos en la base de datos MongoDb en mongo, por ejemplo con MongoDB Compass.
Hay que conectarse a `localhost:27017`, como se puede observar, hay un documento creado para *age01_Car*

![MongoDB_Compass age01_Car](img/20210907_200432.png)

## Ver dispositivos conectados a OrionCB
Para comprobar los dispositivos vinculados al OrionCB desde la terminal, introducir:
```sh
curl -X GET http://localhost:1026/v2/entities/age01_Car/attrs/Speed -H 'fiware-service: opcua_car' -H 'fiware-service-path: /demo'
```

También se puede comprobar desde **postman**, como se muestra en la siguiente imagen
![20210916_184715](img/20210916_184715.png)

> **Nota:** Observar como en el método `GET` realizado al iotagent se pasa en el comando `-H 'fiware-servicepath: /demo'` y en este caso se pasa el comando con `-H 'fiware-service-path: /demo'`, esto parece ser por un error en el código fuente del ioagent, que se intentará resolver más adelante.


# Ejecutar Testbed con un servidor OPCUA externo
Primero detener los contenedores creados anteriormente, para ello ir al directorio donde se ubica el archivo `docker-compose.yml` y ejecutar:
```sh
$ docker-compose down -v
```
>**Nota**: Con `-v` se eliminarán también los volumenes creados.

Se ejecutara el archivo docker-compose-external-service.yml, que tendrá el siguiente contenido:
```yaml
version: "3"
#secrets:
#   age_idm_auth:
#      file: age_idm_auth.txt

services:
  iotage:
    hostname: iotage
    image: iotagent4fiware/iotagent-opcua:latest
    networks:
      - hostnet
      - iotnet
    ports:
      - "4001:4001"
      - "4081:8080"
    extra_hosts:
      - "iotcarsrv:192.168.40.42"
      - "pietro-deb.pietro.local:192.168.40.42"
    depends_on:
      - iotmongo
      - orion
    volumes:
      - ./AGECONF:/opt/iotagent-opcua/conf
    command: /usr/bin/tail -f /var/log/lastlog

  iotmongo:
    hostname: iotmongo
    image: mongo:4.2
    networks:
      - iotnet
    volumes:
      - iotmongo_data:/data/db
      - iotmongo_conf:/data/configdb

  ################ OCB ################

  orion:
    hostname: orion
    image: fiware/orion-ld:0.7.0 #replace with orion:latest if you mind using NGSIv2
    networks:
      - hostnet
      - ocbnet
    ports:
      - "1026:1026"
    depends_on:
      - orion_mongo
    #command: -dbhost mongo
    #entrypoint: /usr/bin/contextBroker -fg -multiservice -ngsiv1Autocast -statCounters -dbhost mongo -logForHumans -logLevel DEBUG #-t 255
    command: -statCounters -dbhost orion_mongo -logForHumans -logLevel DEBUG

  orion_mongo:
    hostname: orion_mongo
    image: mongo:4.2
    networks:
      ocbnet:
        aliases:
          - mongo
    volumes:
      - orion_mongo_data:/data/db
      - orion_mongo_conf:/data/configdb
    command: --nojournal

volumes:
  iotmongo_data:
  iotmongo_conf:
  orion_mongo_data:
  orion_mongo_conf:

networks:
  hostnet:
  iotnet:
  ocbnet:
```
Arrancar con docker-compose:
```sh
$ docker-compose -f docker-compose-external-server.yml up -d
```
Comprobar los log:
```sh
$ docker-compose logs -f
```
Observar como en este caso no aparace en ejecución el contenedor de *iotcarsrv*:
```sh
$ docker ps -a

CONTAINER ID   IMAGE                                   COMMAND                  CREATED         STATUS         PORTS                                                                                  NAMES
b658de8d891d   iotagent4fiware/iotagent-opcua:latest   "docker-entrypoint.s…"   2 minutes ago   Up 2 minutes   0.0.0.0:4001->4001/tcp, :::4001->4001/tcp, 0.0.0.0:4081->8080/tcp, :::4081->8080/tcp   iotagent-opcua_iotage_1
f2e24b318e21   fiware/orion-ld:0.7.0                   "orionld -fg -multis…"   2 minutes ago   Up 2 minutes   0.0.0.0:1026->1026/tcp, :::1026->1026/tcp                                              iotagent-opcua_orion_1
0b8251932707   mongo:4.2                               "docker-entrypoint.s…"   6 minutes ago   Up 6 minutes   27017/tcp                                                                              iotagent-opcua_orion_mongo_1
bddc1005c75f   mongo:4.2                               "docker-entrypoint.s…"   6 minutes ago   Up 6 minutes   27017/tcp                                                                              iotagent-opcua_iotmongo_1
```

Por último comprobar, tal y como se explica en apartados anteriores,desde la terminal con `curl` o desde **postman** los datos en el **iotagent**.

# Problemas detectados en el iotagent 

## Problemas en la ruta de los volumenes creados

Se ha detectado que para que lac onfiguración introducida en el directorio `AGECONF`  y `certificates`se haga efectiva, ha de cambiarse la ruta del volumen creado según la versión de iotagent que se use:

* **iotagent4fiware/iotagent-opcua:latest**
  * `./AGECONF:/usr/src/app/conf`
  * `./certificates:/usr/src/app/certificates`
* **iotagent4fiware/iotagent-opcua:1.3.4**
  * `./AGECONF:/opt/iotagent-opcua/conf`
  * `./certificates:/opt/iotagent-opcua/certificates`

Observando el fichero en el repositorio `iotagent-opcua/docker/Dockerfile` se puede observar como aparace la linea `WORKDIR /opt/iotagent-opcua` para crear la imagen. Posiblemente para la versión `latest ` esto haya cambiado a `WORKDIR /usr/src/app`

**Por tanto, según la versión de iotagent4fiware/iotagent-opcua que se use, se deberá cambiar las rutas de los volumenes.**

## Problemas con ficheros internos 

Como se pudo observar, en el método `GET` realizado al **iotagent**, se pasa el comando `-H 'fiware-servicepath: /demo'` y en el caso de atacar al **OrionCB** se le pasa el metodo `GET` con el comando `-H 'fiware-service-path: /demo'`, esto puede deberse a un error en el código fuente del **iotagent**, donde se haya definido `fiware-servicepath` en lugar de `fiware-service-path`

### Sustituir fiware-servicepath por fiware-service-path

> Esta parte está bajo estudio, aún no se ha conseguido ningún resultado.

Para ello hay que modificar en todos los lugares donde aparece definido `fiware-servicepath` por `fiware-service-path` hay que modificar varios ficheros dentro del contenedor del **iotagent**.

`/usr/src/app/iot-agent-modules/run/mongogroup.js`

`/usr/src/app/tests/add-test.js`

`/usr/src/app/tests/test.js`









