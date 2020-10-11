# fleet-management
Fleet management demo artifacts 

# Overview
This repo contains artifacts to help build a fleet management demo using Confluent Cloud and MongoDB

**Original blog post** : https://www.confluent.io/blog/fleet-management-gps-tracking-with-confluent-cloud-mongodb/

# Steps

## 1. Create a cluster in Confluent Cloud

## 2. Generate API Key and Secrets for the broker and schema registry

## 3. Bring up Kafka connect node (using the docker-compose.yml)

## 4. Create topics

* **Mongo.Fleets.Drivers**
* **location**
* **status**
* **events**

## 5. Generate fleets telemetry

Deploy a voluble connector using this connector config, it generates mock telemetry data to location, status & events topics. 

## 6. Create a ksqlDB Application in Confluent Cloud

## 7. Allow access to ksqlDB cluster

Add ACL's to allow access location, status, events topics from ksqlDB cluster

ccloud kafka cluster list
ccloud kafka cluster use <confluent-cloud-cluster-id>
ccloud ksql app list
ccloud ksql app configure-acls <ksqldb-app-name> Mongo.Fleets.Drivers location status events
  
## 8. Stream Processing

* ** Create a Stream of Fleet Events **

CREATE STREAM FLEET_EVENT_STREAM (
 eventType varchar, speedCaptured integer,
 latitude double, longitude double,
 timestamp varchar, driverId varchar,
 fleetId varchar
) WITH (
 kafka_topic = 'events', value_format = 'JSON',
 timestamp = 'timestamp', timestamp_format = 'E MMM dd HH:mm:ss z yyyy'
);

* ** Pick up hazard events **

CREATE TABLE HAZARDS AS
SELECT driverId,
 COUNT(*)
FROM FLEET_EVENT_STREAM
WINDOW TUMBLING (SIZE 300 SECONDS)
WHERE eventType='HARSH_BRAKING'
GROUP BY driverId
HAVING COUNT(*) > 3;

* ** Capture status requests in a Stream **

CREATE STREAM STATUS_REQUEST (timestamp varchar, fleetId varchar) WITH (kafka_topic='status', value_format='JSON', timestamp='timestamp', timestamp_format='E MMM dd HH:mm:ss z yyyy');

* ** Stream of GPS locations **

CREATE STREAM LOCATION_STREAM (latitude double, longitude double, timestamp varchar, fleetId varchar) WITH (kafka_topic='location', value_format='JSON', timestamp='timestamp', timestamp_format='E MMM dd HH:mm:ss z yyyy');

* ** Tracking - Match Status Request & GPS co-ordinates

CREATE STREAM STATUS_NOTIFICATIONS AS
SELECT s.fleetId, l.latitude, l.longitude
FROM STATUS_REQUEST s
INNER JOIN LOCATION_STREAM l
WITHIN 1 DAYS
ON s.fleetId = l.fleetId
EMIT CHANGES;



