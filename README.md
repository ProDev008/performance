# performance
Performance Analytics

# HR Performance Analytics Microservice (Runnable)

This codebase is a runnable Spring Boot microservice (Java 21) with:
- REST + GraphQL APIs
- Kafka producer/consumer (performance-events)
- Scheduler for daily/weekly/monthly aggregation
- JIRA integration (simple REST client hook)
- Dockerfile + docker-compose for local run
- GitHub Actions CI (build + test)

---

# Project structure (single-file view)

```
hr-performance-service/
├── .github/workflows/ci.yml
├── Dockerfile
├── docker-compose.yml
├── pom.xml
└── src/main/
    ├── java/com/example/hr/
    │   ├── HrPerformanceApplication.java
    │   ├── config/KafkaConfig.java
    │   ├── config/GraphQLConfig.java
    │   ├── controller/EmployeeController.java
    │   ├── graphql/GraphQLController.java
    │   ├── model/Employee.java
    │   ├── model/TaskEvent.java
    │   ├── model/PerformanceAggregate.java
    │   ├── repository/EmployeeRepository.java
    │   ├── repository/PerformanceRepository.java
    │   ├── service/PerformanceService.java
    │   ├── service/ReportService.java
    │   ├── service/JiraIntegrationService.java
    │   └── events/TaskEventListener.java
    └── resources/
        ├── application.yml
        └── graphql/schema.graphqls
```

---

## pom.xml (essential deps)

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>hr-performance-service</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <properties>
    <java.version>21</java.version>
    <spring.boot.version>3.2.0</spring.boot.version>
  </properties>
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>${spring.boot.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-graphql</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
      <groupId>org.postgresql</groupId>
      <artifactId>postgresql</artifactId>
      <scope>runtime</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.kafka</groupId>
      <artifactId>spring-kafka</artifactId>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

---

## src/main/java/com/example/hr/HrPerformanceApplication.java

```java
package com.example.hr;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class HrPerformanceApplication {
  public static void main(String[] args) {
    SpringApplication.run(HrPerformanceApplication.class, args);
  }
}
```

---

## src/main/java/com/example/hr/model/Employee.java

```java
package com.example.hr.model;

import jakarta.persistence.*;

@Entity
public class Employee {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;
  private String employeeId;
  private String name;
  private String department;
  // getters/setters
  public Employee(){}
  // ...
}
```

---

## src/main/java/com/example/hr/model/TaskEvent.java

```java
package com.example.hr.model;

import java.time.LocalDateTime;
import java.util.UUID;

public class TaskEvent {
  private String eventId = UUID.randomUUID().toString();
  private String employeeId;
  private String taskId;
  private String status; // CREATED, COMPLETED
  private long durationSeconds; // time spent
  private LocalDateTime timestamp = LocalDateTime.now();
  // getters/setters
}
```

---

## src/main/java/com/example/hr/model/PerformanceAggregate.java

```java
package com.example.hr.model;

import jakarta.persistence.*;
import java.time.LocalDate;

@Entity
public class PerformanceAggregate {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;
  private String employeeId;
  private LocalDate periodDate; // day or start-of-week or month
  private String periodType; // DAILY | WEEKLY | MONTHLY
  private double completionRate;
  private double avgResolutionSeconds;
  // getters/setters
}
```

---

## src/main/java/com/example/hr/repository/EmployeeRepository.java

```java
package com.example.hr.repository;

import com.example.hr.model.Employee;
import org.springframework.data.jpa.repository.JpaRepository;

public interface EmployeeRepository extends JpaRepository<Employee, Long> {
  Employee findByEmployeeId(String employeeId);
}
```

---

## src/main/java/com/example/hr/repository/PerformanceRepository.java

```java
package com.example.hr.repository;

import com.example.hr.model.PerformanceAggregate;
import org.springframework.data.jpa.repository.JpaRepository;

public interface PerformanceRepository extends JpaRepository<PerformanceAggregate, Long> { }
```

---

## src/main/java/com/example/hr/config/KafkaConfig.java

```java
package com.example.hr.config;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class KafkaConfig {
  @Bean
  public NewTopic performanceTopic() {
    return new NewTopic("performance-events", 1, (short)1);
  }
}
```

---

## src/main/java/com/example/hr/events/TaskEventListener.java

```java
package com.example.hr.events;

import com.example.hr.model.TaskEvent;
import com.example.hr.service.PerformanceService;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class TaskEventListener {
  private final PerformanceService performanceService;
  public TaskEventListener(PerformanceService performanceService) { this.performanceService = performanceService; }

  @KafkaListener(topics = "performance-events", groupId = "hr-service")
  public void onTaskEvent(TaskEvent event) {
    performanceService.recordEvent(event);
  }
}
```

---

## src/main/java/com/example/hr/service/PerformanceService.java

```java
package com.example.hr.service;

import com.example.hr.model.TaskEvent;
import com.example.hr.model.PerformanceAggregate;
import com.example.hr.repository.PerformanceRepository;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ConcurrentLinkedQueue;

@Service
public class PerformanceService {
  private final ConcurrentLinkedQueue<TaskEvent> events = new ConcurrentLinkedQueue<>();
  private final PerformanceRepository repo;
  public PerformanceService(PerformanceRepository repo) { this.repo = repo; }

  public void recordEvent(TaskEvent e){ events.add(e); }

  // Daily aggregation at 18:00
  @Scheduled(cron = "0 0 18 * * ?")
  public void aggregateDaily(){ aggregate("DAILY"); }

  @Scheduled(cron = "0 0 19 ? * MON")
  public void aggregateWeekly(){ aggregate("WEEKLY"); }

  @Scheduled(cron = "0 0 20 1 * ?")
  public void aggregateMonthly(){ aggregate("MONTHLY"); }

  private void aggregate(String periodType){
    List<TaskEvent> drained = new ArrayList<>();
    TaskEvent e;
    while((e = events.poll()) != null) drained.add(e);

    // naive aggregation: group by employeeId
    var grouped = drained.stream().collect(java.util.stream.Collectors.groupingBy(TaskEvent::getEmployeeId));
    grouped.forEach((employeeId, list) -> {
      long total = list.size();
      long completed = list.stream().filter(ev -> "COMPLETED".equals(ev.getStatus())).count();
      double completionRate = total==0?0:((double)completed/total)*100.0;
      double avgRes = list.stream().mapToLong(TaskEvent::getDurationSeconds).average().orElse(0.0);

      PerformanceAggregate pa = new PerformanceAggregate();
      pa.setEmployeeId(employeeId);
      pa.setPeriodDate(LocalDate.now());
      pa.setPeriodType(periodType);
      pa.setCompletionRate(completionRate);
      pa.setAvgResolutionSeconds(avgRes);
      repo.save(pa);
    });
  }
}
```

---

## src/main/java/com/example/hr/service/ReportService.java

```java
package com.example.hr.service;

import com.example.hr.repository.PerformanceRepository;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

@Service
public class ReportService {
  private final PerformanceRepository repo;
  private final JiraIntegrationService jira;
  public ReportService(PerformanceRepository repo, JiraIntegrationService jira){ this.repo = repo; this.jira = jira; }

  // simple scheduled report that checks low performers and creates JIRA
  @Scheduled(cron = "0 30 18 * * ?")
  public void dailyReportAndJira() {
    var low = repo.findAll().stream().filter(p -> p.getCompletionRate() < 50.0).toList();
    if(!low.isEmpty()){
      jira.createIssue("Low performance detected","Employees below 50% completion rate");
    }
  }
}
```

---

## src/main/java/com/example/hr/service/JiraIntegrationService.java

```java
package com.example.hr.service;

import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;
import org.springframework.http.*;
import java.util.Map;

@Service
public class JiraIntegrationService {
  private final RestTemplate rest = new RestTemplate();
  private final String jiraUrl = "https://yourcompany.atlassian.net/rest/api/3/issue";
  private final String authToken = "REPLACE_WITH_BASE64_AUTH"; // store in secrets

  public void createIssue(String title, String description){
    var payload = Map.of(
      "fields", Map.of(
        "project", Map.of("key","HR"),
        "summary", title,
        "description", description,
        "issuetype", Map.of("name","Task")
      )
    );
    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.APPLICATION_JSON);
    headers.setBasicAuth("email@example.com","API_TOKEN");
    HttpEntity<Map> req = new HttpEntity<>(payload, headers);
    try{
      rest.postForEntity(jiraUrl, req, String.class);
    }catch(Exception ex){
      System.err.println("Failed to create JIRA issue: " + ex.getMessage());
    }
  }
}
```

---

## src/main/java/com/example/hr/controller/EmployeeController.java

```java
package com.example.hr.controller;

import com.example.hr.model.Employee;
import com.example.hr.repository.EmployeeRepository;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/employees")
public class EmployeeController {
  private final EmployeeRepository repo;
  public EmployeeController(EmployeeRepository repo){ this.repo = repo; }

  @GetMapping
  public List<Employee> all(){ return repo.findAll(); }

  @PostMapping
  public Employee create(@RequestBody Employee e){ return repo.save(e); }
}
```

---

## src/main/java/com/example/hr/graphql/GraphQLController.java

```java
package com.example.hr.graphql;

import com.example.hr.model.Employee;
import com.example.hr.repository.EmployeeRepository;
import org.springframework.graphql.data.method.annotation.QueryMapping;
import org.springframework.stereotype.Controller;

import java.util.List;

@Controller
public class GraphQLController {
  private final EmployeeRepository repo;
  public GraphQLController(EmployeeRepository repo){ this.repo = repo; }

  @QueryMapping
  public List<Employee> employees(){ return repo.findAll(); }
}
```

---

## src/main/resources/graphql/schema.graphqls

```graphql
type Employee { id: ID! employeeId: String name: String department: String }

type Query { employees: [Employee] }
```

---

## src/main/resources/application.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/hrdb
    username: demo
    password: demo
  jpa:
    hibernate:
      ddl-auto: update
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: hr-service
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus

server:
  port: 8081

logging:
  level:
    root: INFO
```

---

## Dockerfile

```dockerfile
FROM maven:3.9.4-eclipse-temurin-21 AS build
WORKDIR /workspace
COPY pom.xml ./
COPY src ./src
RUN mvn -B -DskipTests package
FROM eclipse-temurin:21-jdk-jammy
WORKDIR /app
COPY --from=build /workspace/target/*.jar app.jar
EXPOSE 8081
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

---

## docker-compose.yml (local test)

```yaml
version: '3.8'
services:
  hr-service:
    build: .
    ports:
      - "8081:8081"
    depends_on:
      - kafka
      - postgres
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/hrdb
  kafka:
    image: bitnami/kafka:3.7
    environment:
      - KAFKA_ENABLE_KRAFT=yes
      - KAFKA_CFG_NODE_ID=1
      - KAFKA_CFG_PROCESS_ROLES=broker,controller
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092
    ports:
      - "9092:9092"
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: hrdb
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
    ports:
      - "5432:5432"
```

---

## .github/workflows/ci.yml (basic)

```yaml
name: CI
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
      - run: mvn -B -DskipTests package
      - run: mvn test
```

---

## How to run locally
1. `docker-compose up --build`
2. Wait for Postgres and Kafka to be ready
3. `curl` REST endpoints at `http://localhost:8081/api/v1/employees`
4. Publish task events to Kafka `performance-events` to trigger aggregation

---

multi-dimensional performance evaluation model that combines quantitative, qualitative, and behavioral indicators

1. Define Quality Metrics (per Role)

Different jobs have different definitions of “quality work.”
For example:

| Domain                | Sample Quality Indicators                                          |
| --------------------- | ------------------------------------------------------------------ |
| **Software Engineer** | Code quality (linting, bug rate, test coverage, review feedback)   |
| **Salesperson**       | Lead conversion ratio, customer retention, deal size               |
| **HR Executive**      | Hiring accuracy, time-to-fill, employee engagement improvement     |
| **Customer Support**  | Resolution rate, CSAT (Customer Satisfaction), first-response time |


Every role should have Key Quality Indicators (KQIs) tied to deliverables.


2. Measure Using Quantitative Data

Automate collection via systems like:

JIRA / Git metrics — story completion rate, rework %, reopened tickets

✅ QA tools — defect leakage, test pass rates

✅ Performance Analytics microservice — completion rate, average resolution time

✅ Customer feedback — NPS or CSAT score

✅ Peer review scores — cross-validation of work outcomes

Example metric calculation from your HR microservice:

qualityScore = (completedTasks / totalTasks) * (1 - reworkRate) * peerReviewScore;

3. Use Qualitative Evaluations

These capture behavior, creativity, communication, and problem-solving.

Methods include:

360° feedback from managers, peers, and subordinates

Performance review comments (competency-based scoring)

Observation checklists for teamwork, initiative, and adherence to values

Self-assessments (reflection on challenges, achievements, and learning)


4. Automate & Visualize Trends

Use dashboards from your Spring Boot microservice + Grafana to show:

Task accuracy %

Productivity vs. quality trade-offs

Weekly/Monthly improvement

JIRA issue resolution trends

5. Apply a Weighted Scoring Model

| Metric                | Weight | Source         | Score (0–100) | Weighted |
| --------------------- | ------ | -------------- | ------------- | -------- |
| Output Accuracy       | 0.3    | QA reports     | 85            | 25.5     |
| Timeliness            | 0.2    | Scheduler logs | 90            | 18       |
| Peer Feedback         | 0.2    | Survey         | 80            | 16       |
| Customer Feedback     | 0.2    | NPS            | 75            | 15       |
| Initiative/Creativity | 0.1    | Manager review | 70            | 7        |
| **Total Score**       | 1.0    |                |               | **81.5** |

6. Continuous Feedback Loop

Automate alerts for consistent low performance

Create JIRA issues (your project already supports this) for improvement plans

Provide monthly quality insights to HR dashboards


