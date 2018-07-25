# Using Kafka

## Features
Kafka is a popular publish-subscribe messaging system. JHipster has an optional support for Kafka, that will:

* Configure Spring Cloud Stream with JHipster.
* Add the necessary configuration in the application-*.yml files to have a sample topic-jhipster topic, and to have an healthcheck monitor for Kafka (which will be available in the health administration screen).
* Generate a Docker Compose configuration file, with the sample topic-jhipster topic, so Kafka is usable by simply typing docker-compose -f src/main/docker/kafka.yml up -d.
* Provide support for Kafka in a microservice environment, when using Docker. The Docker Compose sub-generator will generate a specific Kafka configuration, if one microservice or one gateway uses Kafka. All microservices and gateways will then use that Kafka broker for all their messages. The broker is common for all applications, as it is typically used as a message broker between applications.

## Tutorial on using Kafka with Spring Cloud Stream in a JHipster application
### Prerequisite
Generate a new application and make sure to select Asynchronous messages using Apache Kafka when prompted for technologies you would like to use. A Docker Compose configuration file is generated and you can start Kafka with the command:

```bash
docker-compose -f src/main/docker/kafka.yml up -d
```

### Model
Create a simple model to represent the messages we would be sending through a Kafka topic.

```bash
public class Greeting {
    private String message;

    public Greeting() {
    }

    public String getMessage() {
        return message;
    }

    public Greeting setMessage(String message) {
        this.message = message;
        return this;
    }
}
```

### Message Channels
Spring Cloud Stream introduced an abstraction layer called message channels. Producers sends messages to output channels, and consumers subscribes to input channels for messages. This gives you the flexibility to work with different messaging systems (called binders) without writing a lot of platform-specific code.

Let’s create our output and input channels.

Output channel
```bash
public interface ProducerChannel {

    String CHANNEL = "messageChannel";

    @Output
    MessageChannel messageChannel();
}
```
Input channel
```bash
public interface ConsumerChannel {

    String CHANNEL = "subscribableChannel";

    @Input
    SubscribableChannel subscribableChannel();
}
```

## Configuration
We need to tell Spring Cloud Stream about our channels in the configuration class generated by Jhipster.
```java
@EnableBinding(value = {Source.class, ProducerChannel.class, ConsumerChannel.class})
public class MessagingConfiguration {

}
```
We also need to configure our application to talk to Kafka.
```yaml
spring:
    cloud:
      stream:
        bindings:
            messageChannel:
                destination: greetings
                content-type: application/json
            subscribableChannel:
                destination: greetings
```

This corresponds to:

spring.cloud.stream.bindings.<channelName>.destination.<topic>

### Producer and Consumer
**Producer Resource**

Let’s create a simple REST endpoint that we can invoke to send messages to the Kafka topic, greetings.

```java
@RestController
@RequestMapping("/api")
public class ProducerResource{

    private MessageChannel channel;

    public ProducerResource(ProducerChannel channel) {
        this.channel = channel.messageChannel();
    }

    @GetMapping("/greetings/{count}")
    @Timed
    public void produce(@PathVariable int count) {
        while(count > 0) {
            channel.send(MessageBuilder.withPayload(new Greeting().setMessage("Hello world!: " + count)).build());
            count--;
        }
    }

}
```

**Consumer Service**

We can consume the messages using StreamListener for message mapping and automatic type conversion.

```java
@Service
public class ConsumerService {

    private final Logger log = LoggerFactory.getLogger(ConsumerService.class);


    @StreamListener(ConsumerChannel.CHANNEL)
    public void consume(Greeting greeting) {
        log.info("Received message: {}.", greeting.getMessage());
    }
}
```

### Running the app
Allow access to the endpoint in SecurityConfiguration.java:
```bash
.antMatchers("/api/greetings/**").permitAll()
```
If you invoke the endpoint http://localhost:8080/api/greetings/5, you should see the messages logged to the console.

You can find the full source code on [eosimosu/jhipster-kafka](https://github.com/eosimosu/jhipster-kafka).
