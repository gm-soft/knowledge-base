# Deploy a Kafka broker in a Docker container

[Source](https://www.kaaproject.org/blog/kafka-docker)

---

This tutorial provides a step-by-step instruction on how to deploy a Kafka broker with Docker containers when a Kafka producer and consumer sit on different networks.

As a part of our recent Kaa enhancement we needed to deploy one of our newly created Kaa services together with a Kafka server in Docker containers and test it from a host machine. The service played the role of a Kafka producer and tests on the host machine were a Kafka consumer. Therefore, both the producer and the Kafka broker sit inside the Docker network while the consumer was on the outside. It looks like this:

![](/technologies/2020-12-09-kafka-in-docker/use-case.png)

This is how we did it: The first thing to do is go to the Docker Hub and find a Kafka Docker image. We used Wurstmeister Kafka and ZooKeeper images. Define the first draft of the Docker Compose file:

```yml
version: '2'

services:

  zookeeper:
    image: wurstmeister/zookeeper:3.4.6
    expose:
    - "2181"

  kafka:
    image: wurstmeister/kafka:2.11-2.0.0
    depends_on:
    - zookeeper
    ports:
    - "9092:9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181

```

Bootstrap the above Compose file and use kafka-console-producer.sh and kafka-console-consumer.sh utilities from the Quickstart section of the Apache Kafka site.

The result of running the producer from the Docker host machine:

```bash
andrew@host$ bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
>Hi there!
>It is a test message.
```

The result of running the consumer:

```bash
andrew@host$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
Hi there!
It's a test message.
```

As you can see, everything works just fine when both producer and consumer are in one network. Pay attention to the `KAFKA_ADVERTISED_LISTENERS` environment variable from the above Compose file. Kafka sends the value of this variable to clients during their connection. After receiving that value, the clients use it for sending/consuming records to/from the Kafka broker.

Since we defined the variable as `PLAINTEXT://localhost:9092`, both producer and consumer received it on the initial connection and used it for further communication with the broker through the forwarded 9092 port.

![](/technologies/2020-12-09-kafka-in-docker/client-and-kafka-in-docker.png)

he key takeaway here is that clients use the specified Kafka address (the values of `--bootstrap-server` and `--broker-list`) only for the initial connection. Kafka then redirects them to the value specified in the `KAFKA_ADVERTISED_LISTENERS` variable, which the clients then use for producing/consuming records. Let's run the producer inside an arbitrary Docker container within the same Docker network where the Kafka container is running.

```bash
root@869f83f2f265:/kafka# bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
>Hi there!
[2018-10-10 14:37:40,397] WARN [Producer clientId=console-producer] Connection to node -1 could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
```

No way - we immediately receive the error message about the failed operation. What happened here is the client received the value of the `KAFKA_ADVERTISED_LISTENERS` environment variable (`PLAINTEXT://localhost:9092`), then tried to connect to it and failed since there is no such resolvable address inside the client's Docker container. It is clear that from the client perspective the Kafka is reachable on kafka:9092 address. Therefore, in order for the client to be able to communicate with the broker, the `KAFKA_ADVERTISED_LISTENERS` value must be set to `PLAINTEXT://kafka:9092`. Back to our Docker Compose file:

```yml
  kafka:
    image: wurstmeister/kafka:2.11-2.0.0
    depends_on:
    - zookeeper
    ports:
    - "9092:9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
```

```bash
root@7dfb9eaa81dc:/kafka# bin/kafka-console-producer.sh --broker-list kafka:9092 --topic test
>Hi there!
```


Cool! Now we managed to publish records to the Kafka inside the Docker container.

> Note: the container with the client must be in the same Docker network as the Kafka broker. Otherwise, kafka:9092 won't be resolvable.

Let's try to consume the published message by the consumer from the Docker host machine:

```bash
andrew@host$ bin/kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic test --from-beginning
[2018-10-10 23:57:06,827] WARN Removing server kafka:9092 from bootstrap.servers as DNS resolution failed for kafka (org.apache.kafka.clients.ClientUtils)
```

As expected, the consumer has failed to connect to the broker since `kafka:9092` is not resolvable from the host machine. We need to make some changes to the above Compose file.

```yml
kafka:
    image: wurstmeister/kafka:2.11-2.0.0
    depends_on:
    - zookeeper
    ports:
    - "9092:9092"
    expose:
    - "9093"
    environment:
      KAFKA_ADVERTISED_LISTENERS: INSIDE://kafka:9093,OUTSIDE://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INSIDE:PLAINTEXT,OUTSIDE:PLAINTEXT
      KAFKA_LISTENERS: INSIDE://0.0.0.0:9093,OUTSIDE://0.0.0.0:9092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_INTER_BROKER_LISTENER_NAME: INSIDE
```

Let's explore some of the above environment variables:

- `KAFKA_ADVERTISED_LISTENERS` - the list of available addresses that points to the Kafka broker. Kafka will send them to clients on their initial connection.

- `KAFKA_LISTENERS` - the list of addresses (0.0.0.0:9093, 0.0.0.0:9092) and listener names (INSIDE, OUTSIDE) on which Kafka broker will listen on for incoming connections.

- `KAFKA_LISTENER_SECURITY_PROTOCOL_MAP` - maps the defined above listener names (INSIDE, OUTSIDE) to the PLAINTEXT Kafka protocol.

- `KAFKA_INTER_BROKER_LISTENER_NAME` - points to a listener name that will be used for cross-broker communication.

Here we defined two listeners (`INSIDE://0.0.0.0:9093, OUTSIDE://0.0.0.0:9092`) for internal traffic within the Docker network and external traffic from the Docker host machine respectively. We specified the INSIDE listener for the cross-broker communication. By means of KAFKA_ADVERTISED_LISTENERS and KAFKA_LISTENER_SECURITY_PROTOCOL_MAP we instructed Kafka to send PLAINTEXT://kafka:9093 to those clients that come to it on kafka:9093 and PLAINTEXT://localhost:9092 to those who come on localhost:9092.

In short, we defined the two types of Kafka clients - external and internal - and configured Kafka to send them different addresses on their initial connections.

The producer and the Kafka broker are inside the Docker network.

![](/technologies/2020-12-09-kafka-in-docker/client-and-kafka-in-docker.png)

The consumer is outside, the Kafka broker is inside the Docker network.

![](/technologies/2020-12-09-kafka-in-docker/client-on-host-kafka-in-docker.png)
