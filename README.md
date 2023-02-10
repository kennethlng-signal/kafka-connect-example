# kafka-connect-example

This is based on the [Confluent Platform quickstart](https://docs.confluent.io/platform/current/platform-quickstart.html#quick-start-for-cp). The Docker Compose file is pulled from the [Github repo](https://docs.confluent.io/platform/current/platform-quickstart.html#step-1-download-and-start-cp). Also take a look at Confluent's [Zero to Hero Kafka Connect demo](https://github.com/confluentinc/demo-scene/blob/master/kafka-connect-zero-to-hero/demo_zero-to-hero-with-kafka-connect.adoc).

To create the custom Kafka Connect image, I followed the guide for "[extending Confluent Platform images](https://docs.confluent.io/platform/current/installation/docker/development.html#extend-cp-images)". To build the Dockerfile, run:

```
docker build . -t kennethlng/kafka-connect:1.0.0
```

If you have the Docker desktop app, you will find the newly built image in the Images tab with the name "kennethlng/kafka-connect".

![screenshot](https://github.com/kennethlng-signal/kafka-connect-example/blob/main/images/Screenshot1.png)

The connector is enabled in `connect` in the Docker Compose file.

```
services:
  connect:
    image: kennethlng/kafka-connect-snowflake-sink:1.0.0
    build:
      context: .
      dockerfile: Dockerfile
```

The rest of the fields are untouched from the original Docker Compose file in the Quickstart project.

To start the container, run:

```
docker-compose up -d
```

To verify that the Snowflake connector is available, run:

```
# With jq installed
curl -s localhost:8083/connector-plugins|jq '.[].class'

# Without jq installed
curl -s localhost:8083/connector-plugins
```

The [Kafka Connect REST API](https://docs.confluent.io/platform/current/connect/references/restapi.html#kconnect-rest-interface) is the primary interface to the cluster for managing connectors.

To create the Snowflake sink connector, run:

```
curl -i -X PUT -H  "Content-Type:application/json" \
  http://localhost:8083/connectors/snowflake-sink-connector/config \
  -d '{
    "connector.class": "com.snowflake.kafka.connector.SnowflakeSinkConnector",
  }'
```

Check the status of the Snowflake sink connector:

```
curl -s "http://localhost:8083/connectors?expand=info&expand=status"

curl -s "http://localhost:8083/connectors?expand=info&expand=status" | \
       jq '. | to_entries[] | [ .value.info.type, .key, .value.status.connector.state,.value.status.tasks[].state,.value.info.config."connector.class"]|join(":|:")' | \
       column -s : -t| sed 's/\"//g'| sort
```

Because the REST Proxy container is enabled, we can test this using the Kafka [REST API](https://docs.confluent.io/platform/current/kafka-rest/quickstart.html#produce-and-consume-json-messages) for producing and consuming messages.

Produce a message using JSON with the value '{ "foo": "bar" }' to the topic jsontest:

```
curl -X POST -H "Content-Type: application/vnd.kafka.json.v2+json" \
      --data '{"records":[{"value":{"foo":"bar"}}]}' "http://localhost:8082/topics/jsontest"
```

Create a consumer for JSON data, starting at the beginning of the topic's log.

```
curl -X POST -H "Content-Type: application/vnd.kafka.v2+json" \
      --data '{"name": "my_consumer_instance", "format": "json", "auto.offset.reset": "earliest"}' \
      http://localhost:8082/consumers/my_json_consumer
```

Subscribe to the topic.

```
curl -X POST -H "Content-Type: application/vnd.kafka.v2+json" --data '{"topics":["jsontest"]}' \
 http://localhost:8082/consumers/my_json_consumer/instances/my_consumer_instance/subscription
```

Then consume some data using the base URL in the first response.

```
curl -X GET -H "Accept: application/vnd.kafka.json.v2+json" \
      http://localhost:8082/consumers/my_json_consumer/instances/my_consumer_instance/records
```

Finally, close the consumer with a DELETE to make it leave the group and clean up its resources.

```
curl -X DELETE -H "Content-Type: application/vnd.kafka.v2+json" \
      http://localhost:8082/consumers/my_json_consumer/instances/my_consumer_instance
```

## Resources

- [Confluent demo](https://github.com/confluentinc/demo-scene/tree/master/kafka-connect-zero-to-hero)
- [Kafka Connect REST API](https://docs.confluent.io/platform/current/connect/references/restapi.html#kconnect-rest-interface)
