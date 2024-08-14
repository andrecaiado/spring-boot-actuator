# Spring Boot Actuator project

Spring Boot application with actuator configuration and integration with external monitoring systems

This README file will focus on the actuator features implementation. For more information about the other project features, please refer to the project template:  [spring-boot-template](https://github.com/andrecaiado/spring-boot-template). 

## Contents

- [Dependencies](#dependencies)
- [Exposing Actuator Endpoints](#exposing-actuator-endpoints)
- [Info Endpoint](#info-endpoint)
- [Health Endpoint](#health-endpoint)
- [Metrics Endpoints](#metrics-endpoints)

## Dependencies

The following dependency is required to enable the actuator features in the Spring Boot application.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

## Exposing Actuator Endpoints

The actuators endpoints are available at [http://localhost:8080/actuator](http://localhost:8080/actuator). 

By default, only the `health` and `info` endpoints are exposed over HTTP. To expose all the endpoints, the following configuration was added to the `application.yaml` file.

```properties
management:
  endpoints:
    web:
      exposure:
        include: "*" # Expose all endpoints
```
The shutdown endpoint is not exposed because it is not enabled (it is disabled by default). 

## Info Endpoint

The `info` endpoint is available at http://localhost:8080/actuator/info and is used to display information about the application. 

The information to be displayed must be added to the `application.yaml`.

```yaml
info:
  app:
    name: @project.name@
    description: @project.description@
    version: @project.version@
    encoding: @project.build.sourceEncoding@
    java:
      version: @java.version@
```

We also have to add the following property to the `application.yaml` so the info contents are displayed in the actuator `info` endpoint.

```yaml
management:
  info:
    env:
      enabled: true
```

## Health Endpoint

By default, the health endpoint only returns the status of the application. The status can be `UP`, `DOWN`, or `UNKNOWN`.

```json
{
    "status": "UP"
}
```
If we want to see more details about the application health, we can add the following configuration to the `application.yaml` file.

```yaml
management:
  endpoint:
    health:
      show-details: always
```

Now, the health endpoint will return more details about the application health. Because we have a PostgreSQL database configured in the application, the health endpoint will return the status of the database connection.

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL",
        "validationQuery": "isValid()"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 494384795648,
        "free": 422728798208,
        "threshold": 10485760,
        "path": "/Users/a.caiadodasilva/code/training/spring-boot-actuator/.",
        "exists": true
      }
    },
    "ping": {
      "status": "UP"
    }
  }
}
```

## Metrics Endpoints

The metrics endpoint provides information about the application performance. The metrics are divided into three categories: `system`, `jvm`, and `process`.

The metrics endpoint is available at [http://localhost:8080/actuator/metrics](http://localhost:8080/actuator/metrics).

### Custom Metrics

We can create custom metrics to monitor specific parts of the application.

To learn more about metric types, please refer to the [Prometheus Metric Types page](https://prometheus.io/docs/concepts/metric_types/).

In this project, we created a custom `gauge` metric to monitor the number of employees in the database. This metric was implemented in the `EmployeeService` class as shown below.

```java
@Service
@Slf4j
public class EmployeeService {

    private final EmployeeRepository employeeRepository;

    private final MeterRegistry meterRegistry;

    public EmployeeService(EmployeeRepository employeeRepository, MeterRegistry meterRegistry) {
        this.employeeRepository = employeeRepository;
        this.meterRegistry = meterRegistry;

        Gauge.builder("employees_count", employeeRepository::count)
                .description("The current number of employees in the database")
                .register(meterRegistry);
    }
    ...
}
```

This metric will be available at [http://localhost:8080/actuator/metrics/employees_count](http://localhost:8080/actuator/metrics/employees_count).

```json
{
  "name": "employees_count",
  "description": "The current number of employees in the database",
  "measurements": [
    {
      "statistic": "VALUE",
      "value": 20
    }
  ],
  "availableTags": []
}
```

## Securing Actuator Endpoints with Spring Security