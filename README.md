# kafka-connect-example

This project starts with the [Confluent Platform quickstart](https://docs.confluent.io/platform/current/platform-quickstart.html#quick-start-for-cp). The Docker Compose includes all the Confluent images needed to start up a Confluent app. pulled from the [Github repo](https://docs.confluent.io/platform/current/platform-quickstart.html#step-1-download-and-start-cp). Also take a look at Confluent's [Zero to Hero Kafka Connect demo](https://github.com/confluentinc/demo-scene/blob/master/kafka-connect-zero-to-hero/demo_zero-to-hero-with-kafka-connect.adoc).

## Setup

### Building the custom Kafka Connect image

To install plugins like the Snowflake and Salesforce connectors, we must first extend from an existing Kafka Connect image and then install the plugins we want. To create the custom Kafka Connect image with the Snowflake connector plugin, I followed the guide for "[extending Confluent Platform images](https://docs.confluent.io/platform/current/installation/docker/development.html#extend-cp-images)". The Dockerfile contains the code for extending the Kafka Connect image and installing the Snowflake connector.

The image is then enabled and built in the Docker Compose file.

```
services:
  connect:
    image: kennethlng/kafka-connect:1.0.0
    build:
      context: .
      dockerfile: Dockerfile
```

The rest of the fields in the Docker Compose file are untouched from the original Docker Compose file in the Quickstart project.

### Install additional plugins

If you want to install additional plugins and connectors to the Kafka Connect image, open the Dockerfile and include additional `confluent-hub install` commands, like this:

```
FROM confluentinc/cp-kafka-connect:latest

RUN   confluent-hub install --no-prompt hpgrahsl/kafka-connect-mongodb:1.1.0 \
   && confluent-hub install --no-prompt microsoft/kafka-connect-iothub:0.6 \
   && confluent-hub install --no-prompt wepay/kafka-connect-bigquery:1.1.0
```

### Running the Confluent Platform quickstart app

To start the container, run:

```
docker-compose up -d
```

To verify that the Snowflake connector is available, run:

```
curl -s localhost:8083/connector-plugins

# If you have `jq` installed, you can run this command to format the results in the terminal
curl -s localhost:8083/connector-plugins|jq '.[].class'
```

### Create the Snowflake sink connector

Now that we have the plugins installed, we have to create the Snowflake connector. Similar to Airbyte, this is where we have to pass in the credentials and configurations.

The [Kafka Connect REST API](https://docs.confluent.io/platform/current/connect/references/restapi.html#kconnect-rest-interface) is the primary interface to the cluster for managing connectors.

To create the Snowflake sink connector, run:

<--------EVERYTHING BELOW THIS IS NOT DONE YET--------->

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

# With jq installed to format the results in the terminal
curl -s "http://localhost:8083/connectors?expand=info&expand=status" | \
       jq '. | to_entries[] | [ .value.info.type, .key, .value.status.connector.state,.value.status.tasks[].state,.value.info.config."connector.class"]|join(":|:")' | \
       column -s : -t| sed 's/\"//g'| sort
```

## Demo

### Stream data to Snowflake

Because the REST Proxy container is enabled, we can test this using the Kafka [REST API](https://docs.confluent.io/platform/current/kafka-rest/quickstart.html#produce-and-consume-json-messages) for producing and consuming messages. We can use the REST API to produce messages that will get streamed to Snowflake.

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

- [Confluent Platform quickstart using Docker](https://docs.confluent.io/platform/current/platform-quickstart.html#quick-start-for-cp)
- [Kafka Connect Zero to Hero demo](https://github.com/confluentinc/demo-scene/tree/master/kafka-connect-zero-to-hero)
- [Extending Confluent Platform images](https://docs.confluent.io/platform/current/installation/docker/development.html#extend-cp-images)
- [Kafka Connect REST API](https://docs.confluent.io/platform/current/connect/references/restapi.html#kconnect-rest-interface)
