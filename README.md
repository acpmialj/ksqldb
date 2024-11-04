# ksqlDB
Los elementos necesarios se ponen en marcha con docker compose. Incluyen
* Un servidor ksqlDB
* Un cliente CLI
* Un broker Kafka -- nótese que no necesita ZooKeeper. Está a la escucha en "broker:29092"

## Operaciones
Lanzamiento de contenedores cliente, servidor ksqlDB y servidor Kafka. Apertura de una terminal en el cliente.
```
docker compose up -d
docker exec -it ksqldb-cli ksql http://ksqldb-server:8088
```
En la consola aparece el prompt "ksql>" donde podemos insertar sentencias SQL. 

### Creación stream
Creamos un stream (asociado a un topic Kafka, que se crea en este momento, luego está vacío)
```
CREATE STREAM riderLocations (profileId VARCHAR, latitude DOUBLE, longitude DOUBLE)
  WITH (kafka_topic='locations', value_format='json', partitions=1);
```
Se espera que los mensajes tengan formato JSON, como este:
```
{"profileId": "c2309eec", "latitude": 37.7877, "longitude": -122.4205}
```
### Creación de vistas materializadas
Son tablas virtuales asociadas a consultas sobre el stream. Creamos dos:
```
-- Create the currentLocation table
CREATE TABLE currentLocation AS
  SELECT profileId,
         LATEST_BY_OFFSET(latitude) AS la,
         LATEST_BY_OFFSET(longitude) AS lo
  FROM riderlocations
  GROUP BY profileId
  EMIT CHANGES;
```

```
-- Create the ridersNearMountainView table
CREATE TABLE ridersNearMountainView AS
  SELECT ROUND(GEO_DISTANCE(la, lo, 37.4133, -122.1162), -1) AS distanceInMiles,
         COLLECT_LIST(profileId) AS riders,
         COUNT(*) AS count
  FROM currentLocation
 GROUP BY ROUND(GEO_DISTANCE(la, lo, 37.4133, -122.1162), -1);
```

Las vistas materializadas están creadas, se manejarán como tablas cuando tengan contenido.

### Consultas push
Son consultas que quedan en ejecución permanente (en este caso, bloqueando el terminal)
```
-- Mountain View lat, long: 37.4133, -122.1162
SELECT * FROM riderLocations
  WHERE GEO_DISTANCE(latitude, longitude, 37.4133, -122.1162) <= 5 EMIT CHANGES;
```
Como no hay datos, aparecerá lo siguiente:
```
+--------------------+-------------------+--------------------+
|PROFILEID           |LATITUDE           |LONGITUDE           |
+--------------------+-------------------+--------------------+

Press CTRL-C to interrupt
```
### Inserción de eventos
Abrimos otro terminal y en el, otro CLI, sea CLI2 (el anterior, CLI1)
```
docker exec -it ksqldb-cli ksql http://ksqldb-server:8088
```
En CLI2, ejecutamos operaciones de inserción:
```
INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('c2309eec', 37.7877, -122.4205);
INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('18f4ea86', 37.3903, -122.0643);
INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('4ab5cbad', 37.3952, -122.0813);
INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('8b6eae59', 37.3944, -122.0813);
INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('4a7c7b41', 37.4049, -122.0822);
INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('4ddad000', 37.7857, -122.4011);
```
Mientras tanto, en CLI1 veremos que han empezado a aparecer datos como resultado de nuestra consluta push:
```
+--------------------+-------------------+--------------------+
|PROFILEID           |LATITUDE           |LONGITUDE           |
+--------------------+-------------------+--------------------+
|4ab5cbad            |37.3952            |-122.0813           |
|8b6eae59            |37.3944            |-122.0813           |
|4a7c7b41            |37.4049            |-122.0822           |

Press CTRL-C to interrupt
```
### Observando el contenido de Kafka
Desde otro terminal, podemos usar el siguiente comando para comprobar que Kafka contiene los eventos insertados:
```
docker compose exec -it broker kafka-console-consumer --bootstrap-server broker:9092 --topic locations --from-beginning
{"PROFILEID":"c2309eec","LATITUDE":37.7877,"LONGITUDE":-122.4205}
{"PROFILEID":"18f4ea86","LATITUDE":37.3903,"LONGITUDE":-122.0643}
{"PROFILEID":"4ab5cbad","LATITUDE":37.3952,"LONGITUDE":-122.0813}
{"PROFILEID":"8b6eae59","LATITUDE":37.3944,"LONGITUDE":-122.0813}
{"PROFILEID":"4a7c7b41","LATITUDE":37.4049,"LONGITUDE":-122.0822}
{"PROFILEID":"4ddad000","LATITUDE":37.7857,"LONGITUDE":-122.4011}
```
Para salir, pulsar Ctrl-C. 


### Consultas pull
Ahora que hay datos, podemos usar nuestras vistas materializadas como tablas y hacer consultas, que devuelven los resultados inmediatamente (no hay "EMIT")
```
SELECT * from ridersNearMountainView WHERE distanceInMiles <= 10;

+--------------------+-------------------------------+------+
|DISTANCEINMILES     |RIDERS                         |COUNT |
+--------------------+-------------------------------+------+
|0.0                 |[4ab5cbad, 8b6eae59, 4a7c7b41] |3     |
|10.0                |[18f4ea86]                     |1     |
Query terminated
```

## Limpieza
Pulsamos Ctrl-C en CLI1 para terminar la consulta PUSH. Salimos de CLI1 y CLI2 con "exit". Destruimos el despliegue con "docker compose down". 
