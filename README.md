# kafka-connect-example

This project starts with the [Confluent Platform quickstart](https://docs.confluent.io/platform/current/platform-quickstart.html#quick-start-for-cp). The Docker Compose includes all the core Confluent images needed to start up a Confluent app (Kafka broker, Kafka Connect, schema registry, control center, etc.).

## Resources

- [Confluent Platform quickstart using Docker](https://docs.confluent.io/platform/current/platform-quickstart.html#quick-start-for-cp)
- [Kafka Connect Zero to Hero demo](https://github.com/confluentinc/demo-scene/tree/master/kafka-connect-zero-to-hero): This project is heavily based on this demo.
- [Extending Confluent Platform images to add connectors](https://docs.confluent.io/platform/current/installation/docker/development.html#extend-cp-images)
- [Installing the Snowflake Kafka connector](https://docs.snowflake.com/en/user-guide/kafka-connector-install.html)
- [Kafka Connect REST API](https://docs.confluent.io/platform/current/connect/references/restapi.html#kconnect-rest-interface)

## Setup

### Building the custom Kafka Connect image

To install plugins like the Snowflake and Salesforce connectors, we must first extend from an existing Kafka Connect image from Confluent and then install the plugins we want. This is based on the guide for "[extending Confluent Platform images](https://docs.confluent.io/platform/current/installation/docker/development.html#extend-cp-images)". See the Dockerfile for how the custom image is built.

```
FROM confluentinc/cp-kafka-connect:latest

ENV CONNECT_PLUGIN_PATH="/usr/share/java,/usr/share/confluent-hub-components"

RUN confluent-hub install --no-prompt snowflakeinc/snowflake-kafka-connector:1.8.2
```

The image is then enabled and built in the Docker Compose file.

```
services:
  connect:
    image: signal-data/kafka-connect:1.0.0
    build:
      context: .
      dockerfile: Dockerfile
```

### Install additional plugins

If you want to install additional plugins and connectors to the Kafka Connect image, edit the Dockerfile and include additional `confluent-hub install` commands, like this:

```
FROM confluentinc/cp-kafka-connect:latest

ENV CONNECT_PLUGIN_PATH="/usr/share/java,/usr/share/confluent-hub-components"

RUN   confluent-hub install --no-prompt snowflakeinc/snowflake-kafka-connector:1.8.2 \
   && confluent-hub install --no-prompt microsoft/kafka-connect-iothub:0.6 \
   && confluent-hub install --no-prompt wepay/kafka-connect-bigquery:1.1.0
```

### Run the Confluent Platform quickstart app

To start the container, run:

```
docker-compose up -d
```

![](https://github.com/kennethlng-signal/kafka-connect-example/blob/main/images/screenshot_confluent_containers.png)

### Verify the plugins are available

To verify that the Snowflake connector is available, run:

```
curl -s localhost:8083/connector-plugins
```

Or if you have `jq` installed, you can run this command to format the results in the terminal:

```
curl -s localhost:8083/connector-plugins|jq '.[].class'
```

You should see the Snowflake connector listed as one of the plugins:

```
"com.snowflake.kafka.connector.SnowflakeSinkConnector"
"org.apache.kafka.connect.mirror.MirrorCheckpointConnector"
"org.apache.kafka.connect.mirror.MirrorHeartbeatConnector"
"org.apache.kafka.connect.mirror.MirrorSourceConnector"
```

### Set up the Snowflake environemnt

> The Snowflake environment has already been set up, so you do not need to do anything in this section. You can continue reading if you are interested in knowing how it was set up.

> Our Snowflake instance is available at https://dv98916.us-central1.gcp.snowflakecomputing.com.

If you haven't done so yet, go to the official [Snowflake guide](https://docs.snowflake.com/en/user-guide/kafka-connector-install.html) for setting up a connector with Snowflake. For the section involving running Snowflake worksheet commands to create a user and role, follow this [guide from Confluent](https://docs.confluent.io/cloud/current/connectors/cc-snowflake-sink.html#creating-a-user-and-adding-the-public-key) instead. The guide will create a custom user named `confluent` with the role `kafka_connector_role`. In our Snowflake instance in the Worksheets tab, you can open the worksheet called "Kafka Connect setup" to see how this user and role was created.

If there is no database called `KAFKA_CONNECT_TEST`, use your `ACCOUNTADMIN` role and create a database called `KAFKA_CONNECT_TEST` in the Snowflake console. Creating a database automatically creates a `PUBLIC` schema in that database, which we will use to create new tables from the Kafka topics.

### Create the Snowflake sink connector

Now that our Confluent app is running and we have the Snowflake Kafka connector plugin installed, we have to create the Snowflake connector. Similar to Airbyte, this is where we have to pass in the credentials and configurations to connect the Kafka Connector with our Snowflake account.

The [Kafka Connect REST API](https://docs.confluent.io/platform/current/connect/references/restapi.html#kconnect-rest-interface) is the primary interface to the cluster for managing connectors.

To create the Snowflake sink connector, run the following command in the terminal. This creates a connector using the custom "confluent" Snowflake user that we set up previously. This connector is listening for the Kafka topic called `jsontest`.

```
curl -i -X PUT -H  "Content-Type:application/json" \
  http://localhost:8083/connectors/snowflake-sink-connector/config \
  -d '{
    "connector.class":"com.snowflake.kafka.connector.SnowflakeSinkConnector",
    "topics":"jsontest",
    "snowflake.url.name": "dv98916.us-central1.gcp.snowflakecomputing.com:443",
    "snowflake.user.name":"confluent",
    "snowflake.private.key":"MIIFHzBJBgkqhkiG9w0BBQ0wPDAbBgkqhkiG9w0BBQwwDgQIFEFvGsjGHIoCAggAMB0GCWCGSAFlAwQBKgQQpJ9WzdFUhkjXIlZdz84+CwSCBNDr43hM42VVq9MPUDyvpA1wG5ok72YbsbrKkekPYQWiU99rEciLVqAJk0TJEyFmAxtasRfSEfCW2IHWmKcDbCUfn9tdraO5vLK+1wUFSoDruWAN7/pqnBEkUuQNA+9d3WKRygA44C6eNCJqOFJmVdDwR6vFLqMRKJxDgtT/KC/gCuKYlIMeEp3/qiKLeObosTfkzPwA5q7r3Ckik7Ku3w04G0vYCB+0KJxzOOLr433VRpW8VwfNBPJo2tUBYvmnLjTXNkRilWWTTBgT5p9HmJZXWw9mKDc3eaQDcNnE1jd80SDMLezF8gXAPD/qfuwifPdqE6g0vbSCV9DFlJYrel/Xq01Hy5ccEIJi013Tdtq6RhhHnAZorrwFXUqXXrgIPCsYWeW9I7sAFrnf4Pe3CBCVWrRjVJBNYRWXMiEX2CyzhuN8KgaNoGyXkiti7Vh1Cy+7xpKHtOAi3lDytINEsecCEI84LuNyKCguRokL8jTH0Qr2AwKW5KD7+HFEQMYyNGup2xCYBV9or2j66Djk1eV7yx1T6QawH1u+TCRDcaMEnNGoiCRljpxnyOACWiBRgu8RIT1cE2adcKM6rd3Q7gS4o/1tMjgVWkYtBV4D7XkTwdKiCCHMJ+8iBXWI/Hfw76M4CW+8v/4hav4Se4KATqiOMhMZy6ZvdXEg5nLMrDmvt3WhOoXuz5WUgwzhQ8zmPxBBpq8DRoifJcOKSYeJ2+dwXh1qat3MSMq6cs50y7pF42+HE2Kt/UutGDWh1F0N/ytjaM68a5ZgM6WStb59HQsVBFjlncJJEN4kdJDXBEwTOdlay8Ytz43yyrq6KhqQDx98DKCf9947Mg1JiLjX+BQvWE4/Ar8qjBhfBjtmNGK3XwF0x5U9/SS+jOwoxfBB6P/cKyWSjIeIiwq0oqX8tHzbGNEQVDD9CyGN4oJz2c3lx2E8SMjmz+hEJ2bzkDb+5vjR2NJrKOCJ2NdsR6mDik3me3lKzjj1V4ObzwgN1BvWZE9KTUoV/4e0p2eHAeFwOF6+P5qFJETfM4mh29rGiYn8K322M2cobWaD+ibtB2iQ8Q2/cGT6lN/YIM0j44z+7dA5hdsjs0Df5ipsWUQXR33FLVmv7jOqlUyPTsyVsN7gU5U6nWz9ofjh2LobnLYk9aIIb6aB2MLH/NKJhqCsM5piKgF4hqx3lmH1Iyk0zWtAQAuw5GlYy6mL7R+/6I1dYUUxq/tTNo1paI4fAPKb66C2XbTrGfYPelJldqUxVWHLn9IxQZ6h/mqMeu3if5GA4YwlwkqLlV0TvO6Xx5l5GEOUQew5HNqlro7iKLoCqjd3mdMvkova2hdIQlxFfQ1IB7IpfkUdwlTpxjRhCkwU+7Bxoa57YXXvPKaJUp6zAFpWj4wuuDUtfHNZ+Q+rtv4vrjf4ARISOKIBnE9asYYtccq6K3giti1wI5viuW28FB9/wCZFtu+qAorL77PKFUEYNVhU1fTnchmdnFKa2wBABL+kMF24NmVB52t+djaSBX1DWYXIhnP9hxg2nqcyiw9OiQVMm0HkMIzt/vAdiTrQtxrH2TZTYe71ZrZzkTENyEO8abL22JSOeShabiOatYhFj0W2mRw2rNsWla4pww8kxvhvI8uxk9RoL4fLuZ5uPSnQbg==",
    "snowflake.private.key.passphrase":"signaldata",
    "snowflake.database.name":"KAFKA_CONNECT_TEST",
    "snowflake.schema.name":"PUBLIC",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "com.snowflake.kafka.connector.records.SnowflakeJsonConverter"
  }'
```

> The private key and passphrase are tied to the "confluent" user. Do not change these values. This user and the private keys are used for testing purposes only, so they will get revoked eventually.

Each connector you create must have a unique name or it will be overriden when you run this command. The name for this connector is called "snowflake-sink-connector", as you can see in the url. To create another connector, run this command but with a new name.

### Check the status of the Snowflake sink connector

```
curl -s "http://localhost:8083/connectors?expand=info&expand=status"
```

Or, if you have jq installed, run the following to format the results in the terminal.

```
curl -s "http://localhost:8083/connectors?expand=info&expand=status" | \
       jq '. | to_entries[] | [ .value.info.type, .key, .value.status.connector.state,.value.status.tasks[].state,.value.info.config."connector.class"]|join(":|:")' | \
       column -s : -t| sed 's/\"//g'| sort
```

## Test the connector

Now that the connector is set up, we can test if it works by producing Kafka messages.

### Stream data to Snowflake

Because the REST Proxy container is enabled, we can test the connector by using the Kafka [REST API](https://docs.confluent.io/platform/current/kafka-rest/quickstart.html#produce-and-consume-json-messages) for producing messages that will get streamed to Snowflake.

Produce a message using JSON with the value '{ "foo": "bar" }' to the topic `jsontest`:

```
curl -X POST -H "Content-Type: application/vnd.kafka.json.v2+json" \
      --data '{"records":[{"value":{"foo":"bar"}}]}' "http://localhost:8082/topics/jsontest"
```

Wait a couple minutes, and inside Snowflake you will see the topic created as a table in the `PUBLIC` schema of the `KAFKA_CONNECT_TEST` database.

![](https://github.com/kennethlng-signal/kafka-connect-example/blob/main/images/screenshot_snowflake_topic.png)

<!-- Create a consumer for JSON data, starting at the beginning of the topic's log.

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
``` -->

## Troubleshooting

### Delete the connector

If the connector has crashed or has stopped working, you can delete the connector by running:

```
curl -X DELETE http://localhost:8083/connectors/snowflake-sink-connector
```
