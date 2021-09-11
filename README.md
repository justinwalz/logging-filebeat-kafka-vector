# readme

Log and event pipeline built on filebeat, kafka, vector.

## tldr

docker logs -> filebeat -> kafka -> vector -> loki and stdout

## overview

The [vector kubernetes logs](https://vector.dev/docs/reference/configuration/sources/kubernetes_logs/) source is technically marked as stable however I've had some issues with desync's and dropped logs. See [this RFC](https://github.com/timberio/vector/issues/7527) for more.

The [filebeat docker input](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-input-docker.html) has been around a long time, and has support for enriching event metadata in cloud, docker, and kubernetes environments.

So we ship directly to kafka to keep the log collection side as simple as possible.

We then aggregate with vector to filter, process, and send logs and events to their final destination. In this repo I set that to console stdout, but something like Loki, Elasticsearch, Postgres, S3, etc. would all work just fine.

## getting started

### hello world kafka

From the [quickstart](https://kafka.apache.org/quickstart).

Pane 1: Download kafka, start zookeeper

```bash
wget https://dlcdn.apache.org/kafka/2.8.0/kafka_2.13-2.8.0.tgz
tar -xzf kafka_2.13-2.8.0.tgz
cd kafka_2.13-2.8.0
bin/zookeeper-server-start.sh config/zookeeper.properties
```

Pane 2: start server

```bash
cd kafka_2.13-2.8.0
bin/kafka-server-start.sh config/server.properties
```

Pane 3: create topic, start producer

```bash
cd kafka_2.13-2.8.0
bin/kafka-topics.sh --create --topic quickstart-events --bootstrap-server localhost:9092
bin/kafka-console-producer.sh --topic quickstart-events --bootstrap-server localhost:9092
```

Pane 4: start consumer

```bash
cd kafka_2.13-2.8.0
bin/kafka-console-consumer.sh --topic quickstart-events --from-beginning --bootstrap-server localhost:9092\n
```

### adding filebeat

Publish log messages to kafka. [Install docker](https://docs.docker.com/get-docker) here if you do not already have it.

```bash
docker run -d \
  --name=filebeat \
  --user=root \
  --volume="$(pwd)/filebeat.docker.yml:/usr/share/filebeat/filebeat.yml:ro" \
  --volume="/var/lib/docker/containers:/var/lib/docker/containers:ro" \
  --volume="/var/run/docker.sock:/var/run/docker.sock:ro" \
  docker.elastic.co/beats/filebeat:7.14.1 filebeat -e -strict.perms=false
```

Optional: Consume logs with kafka console consumer

```bash
bin/kafka-console-consumer.sh --topic filebeat  --from-beginning --bootstrap-server localhost:9092 | jq -rc .message
```

### adding vector

Consume logs with vector, to stdout. [Install vector](https://vector.dev/docs/setup/quickstart/) here if you do not already have it.

```
vector --config vector.yml
```

## production

Use a production kafka setup if you have it. Otherwise check out a variety of hosted solutions, one of which being [MSK](https://aws.amazon.com/msk/).

Deploy filebeat on kubernetes with tanka and helm by following [my tutorial here (TBD)](https://github.com/justinwalz/).

Deploy vector on kubernetes in aggregator mode by following the [deployment guide here](https://vector.dev/docs/setup/deployment/roles/#aggregator).

Ship finalized events to the datastore of your choosing, internal, hosted, or public. I recently like [Loki](https://grafana.com/oss/loki/).
