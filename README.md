# Spring Boot Actuator project

Spring Boot application with actuator configuration and integration with external monitoring systems.

This README file will focus on the actuator features implementation. For more information about the other project features, please refer to the project template:  [spring-boot-template](https://github.com/andrecaiado/spring-boot-template). 

# Contents

- [Dependencies and requirements](#dependencies-and-requirements)
- [Exposing Actuator Endpoints](#exposing-actuator-endpoints)
- [Info Endpoint](#info-endpoint)
- [Health Endpoint](#health-endpoint)
- [Metrics Endpoints](#metrics-endpoints)
  - [Custom Metrics](#custom-metrics)
- [Securing Actuator Endpoints with Spring Security](#securing-actuator-endpoints-with-spring-security)
- [Monitoring Systems Integration](#monitoring-systems-integration)
  - [Prometheus](#prometheus)
    - [Integration with the Spring Boot application](#integration-with-the-spring-boot-application)
    - [Prometheus server](#prometheus-server)
  - [Grafana](#grafana)
    - [Setting up the Prometheus data source in Grafana](#setting-up-the-prometheus-data-source-in-grafana)
    - [Importing a Grafana dashboard](#importing-a-grafana-dashboard)

# Dependencies and requirements

The following dependency are required to implement this project features:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

To run the external services (Postgres, Prometheus and Grafana) in Docker containers, the following requirements are needed:

- Docker
- Docker Compose

# Exposing Actuator Endpoints

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

# Enabling the Shutdown Endpoint

The shutdown endpoint is used to shutdown the application and so, it is disabled by default. 
To enable this endpoint, the following configuration must be added to the `application.yaml` file.

```yaml
management:
  endpoint:
    shutdown:
      enabled: true
```

# Info Endpoint

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

# Health Endpoint

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

# Metrics Endpoints

The metrics endpoint provides information about the application performance. The metrics are divided into three categories: `system`, `jvm`, and `process`.

The metrics endpoint is available at [http://localhost:8080/actuator/metrics](http://localhost:8080/actuator/metrics).

## Custom Metrics

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

# Securing Actuator Endpoints with Spring Security

To secure the actuator endpoints, we can use Spring Security.

The following dependency was added to enable Spring Security in the Spring Boot application.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

A configuration was added to secure the `shutdown` actuator endpoint with HTTP Basic Authentication.

The configuration also includes an inMemory user with credentials to access the `shutdown` endpoint.

The configuration can be found in the [SecurityConfig.java](src%2Fmain%2Fjava%2Fcom%2Fexample%2Fspringboottemplate%2Fconfig%2FSecurityConfig.java) class.

# Monitoring Systems Integration

The Spring Boot application was integrated with two monitoring systems: Prometheus and Grafana.

## Prometheus

### Integration with the Spring Boot application

The Prometheus monitoring system was integrated with the Spring Boot application to collect metrics from the application.

To integrate Prometheus with the Spring Boot application, we used the `micrometer-registry-prometheus` dependency. To learn more about Micrometer, please refer to the [Micrometer documentation](https://micrometer.io/docs).

The following dependency was added to the `pom.xml` file.

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

After adding the dependency, Spring Boot will automatically configure a `PrometheusMeterRegistry` and a `CollectorRegistry` to collect and export metrics in a format that can be scraped by Prometheus.

The metrics can then be accessed at [http://localhost:8080/actuator/prometheus](http://localhost:8080/actuator/prometheus).

### Prometheus server

A Prometheus server service was added to the `docker-compose.yml` file to start the Prometheus server in a Docker container.  

The configuration for the Prometheus server can be found in the [prometheus.yml](prometheus.yml) file.

To start the Prometheus server, run the following command:

```shell
docker compose up -d prometheus
```

The Prometheus server will be available at [http://localhost:9090](http://localhost:9090).

## Grafana

The Grafana monitoring system was integrated with the Spring Boot application to visualize the metrics collected by Prometheus.

Although Prometheus provides dashboards to visualize the metrics, Grafana is a more powerful tool for this purpose.

A Grafana server service was added to the `docker-compose.yml` file to start the Grafana server in a Docker container.

To start the Grafana server, run the following command:

```shell
docker compose up -d grafana
```

The Grafana server will be available at [http://localhost:3000](http://localhost:3000).

The default credentials to access Grafana are:

- Username: admin
- Password: admin

Change the password after the first login or skip the password change by clicking on the `Skip` button.

### Setting up the Prometheus data source in Grafana

After logging in, add a new data source to Grafana:
- Data Source type: Prometheus
- Prometheus server URL: http://localhost:9090 
  - It will probably not work since we are running the services in Docker containers. Use `host.docker.internal` instead of `localhost`.
- Authentication: None
- Click "Save & Test"

### Importing a Grafana dashboard

Instead of creating a new dashboard from scratch, we can import a pre-built dashboard from the Grafana dashboard library.

Visit the [Grafana dashboard library](https://grafana.com/grafana/dashboards) and search for a dashboard that fits your needs.

To learn about Importing Grafana dashboards, please refer to the [Grafana dashboards documentation](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/import-dashboards/).

**Example of an imported dashboard applied on this project:**

Source: [Spring Boot Statistics & Endpoint Metrics](https://grafana.com/grafana/dashboards/14430-spring-boot-statistics-endpoint-metrics/)

![grafana-dashboard.png](src%2Fmain%2Fresources%2Fgrafana-dashboard.png)
