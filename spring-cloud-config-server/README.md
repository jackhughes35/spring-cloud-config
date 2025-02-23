# Spring Cloud Config Server with Kafka Bus Integration

## Description

This project demonstrates the integration of Spring Cloud Config Server with Spring Cloud Bus using Kafka. It enables centralized configuration management for distributed systems, allowing dynamic property updates across applications without requiring redeployment.

## Usage

### 1. Spring Cloud Config Repository
The configuration repository is a Git-based repository that stores the application's configuration properties.

#### Repository Structure:
The repository follows the structure:
```plaintext
oms-configuration-repo/
  |-- {application-name}/
      |-- {application-name}-{env}.yaml
```

Example:
```plaintext
oms-configuration-repo/
  |-- oms-cancellation-manager/
      |-- oms-cancellation-manager-dev.yaml
      |-- oms-cancellation-manager-prod.yaml
```

- The repository is hosted at: [https://gitlab.com/HnBI/fulfilment/shared/oms-configuration-repo](https://gitlab.com/HnBI/fulfilment/shared/oms-configuration-repo).
- It runs on the `main` branch only.
- Authentication to the GitLab repository is done using a username (`oauth`) and a password (access token):
    - Ensure these credentials are present as environment variables 
  
      ```bash
      export GIT_USERNAME=oauth
      export GIT_PASSWORD=<YOUR_ACCESS_TOKEN>
      ```

    - You can see these referenced in the configuration application.yaml:
      ```yaml
      spring:
        cloud:
          config:
            server:
              git:
                uri: https://gitlab.com/HnBI/fulfilment/shared/oms-configuration-repo
                username: ${GIT_USERNAME}
                password: ${GIT_PASSWORD}
      ```

---

### 2. Spring Cloud Config Server
The Config Server provides configuration properties to client applications and integrates with Kafka for dynamic property updates.

#### Authentication Mechanism:
The Config Server accesses the GitLab repository using a username (`oauth`) and a password (access token). These credentials are injected as environment variables, ensuring secure and seamless integration with the Git repository. The token has the `read_repository` scope and should be rotated periodically for security.

---

### 3. Spring Cloud Bus with Kafka
Spring Cloud Bus is integrated with Kafka to propagate configuration changes dynamically.

#### Kafka Configuration:
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
  cloud:
    bus:
      enabled: true
      destination: oms-cloud-config-topic
```

---

### 4. Client Application

The client application fetches configuration properties from the Config Server and listens for updates via the Kafka bus. To enable this, ensure to add the below to the client application;

#### Dependencies:
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-bus-kafka</artifactId>
</dependency>
```

#### Application Configuration:
```yaml
spring:
  cloud:
    config:
      enabled: true
    stream:
      bindings:
        springCloudBusInput:
          group: ${spring.application.name}-${HOSTNAME}
      default:
        group: ${spring.application.name}
    bus:
      enabled: false
      trace:
        enabled: true
      destination: oms-cloud-config-topic
      ack:
        destination-service: oms-spring-cloud-config-server

management:
  endpoints:
    web:
      exposure:
        include: "*"
```

#### Kubernetes Environment Configuration:
```yaml
SPRING_PROFILES_ACTIVE: dev
SPRING_CLOUD_BUS_ENABLED: true
SPRING_CLOUD_STREAM_BINDINGS_SPRINGCLOUDBUSINPUT_GROUP: ${spring.application.name}-${HOSTNAME}
```

#### Annotating Classes for Refresh:
Mark any bean that should be refreshed dynamically with `@RefreshScope`:

```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@RefreshScope
@Configuration
@ConfigurationProperties
public class RefreshableConfigurationClass {
    @Value("${my.config.property}")
    private String configProperty;
}
```

---

### Integration Tests
To test Kafka integration, include the Kafka bootstrap servers dynamically in the test configuration:
```java
public static class DependencyInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {

    @Override
    public void initialize(ConfigurableApplicationContext configurableApplicationContext) {

        TestPropertyValues.of(
            "spring.kafka.bootstrap-servers=" + KAFKA_CONTAINER.getBootstrapServers(),
            "spring.cloud.stream.kafka.binder.brokers=" + KAFKA_CONTAINER.getBootstrapServers(),
            ).applyTo(configurableApplicationContext.getEnvironment());
    }
}
```

---

### Spring Cloud Config Bus with Kafka
When a change is made to the configuration repository, the following sequence occurs:

1. A REST request is sent to the Config Server's `/actuator/busrefresh` endpoint, specifying the target application:
   ```bash
   POST https://oms-spring-cloud-config-server-internal-eks.eu-west-1.dev.hbi.systems/actuator/busrefresh/oms-cancellation-manager
   ```

2. The Config Server emits a `RefreshRemoteApplicationEvent`, which is sent via Kafka:
   ```json
   {
     "type": "RefreshRemoteApplicationEvent",
     "timestamp": 1736034693845,
     "originService": "oms-cloud-config-server",
     "destinationService": "oms-cancellation-manager:**",
     "id": "668fc804-dbe4-4474-8913-c22282278c17"
   }
   ```

    - **`destinationService`**: Specifies the target application (e.g., `oms-cancellation-manager`). The `**` wildcard indicates that all instances of the application should receive the event.

3. Each instance of the client application receives the event and processes it. After processing, the client application sends an acknowledgment back to the Config Server:
   ```json
   {
     "type": "AckRemoteApplicationEvent",
     "timestamp": 1736034704068,
     "originService": "oms-cancellation-manager:dev:8080:a11bbc6c74f8b07b88d1a1975ea0f433",
     "destinationService": "oms-cloud-config-server",
     "id": "d86ffc56-40e0-4c5f-8332-e5f2a4639db0",
     "ackId": "668fc804-dbe4-4474-8913-c22282278c17",
     "ackDestinationService": "oms-cancellation-manager:**",
     "event": "org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent"
   }
   ```

    - **`destinationService`** in the acknowledgment refers back to the Config Server.
    - **`ackId`** corresponds to the ID of the `RefreshRemoteApplicationEvent`.

This mechanism ensures that all instances of the client application are updated dynamically and can confirm the updates back to the Config Server.

---

## Authors
- [Jack Hughes / OMS Squad)

## Acknowledgments
- [Spring Cloud Documentation](https://spring.io/projects/spring-cloud)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Spring Guides](https://spring.io/guides)

Feel free to contribute or raise issues for further improvements on slack in the #oms-support channel!

