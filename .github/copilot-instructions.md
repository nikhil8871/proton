# VProfile Project - Copilot Instructions

## Project Overview
VProfile is a Spring MVC web application (Maven WAR) for user account management with multi-layer infrastructure integration. The app combines traditional Spring configuration with modern Jakarta EE and integrates external services (MySQL, RabbitMQ, Memcached, Elasticsearch).

## Architecture

### Layered Structure
- **Controller Layer** ([src/main/java/com/visualpathit/account/controller/](src/main/java/com/visualpathit/account/controller/)): Spring MVC controllers handling HTTP requests; user, file upload, RabbitMQ, and Elasticsearch endpoints
- **Service Layer** ([src/main/java/com/visualpathit/account/service/](src/main/java/com/visualpathit/account/service/)): Business logic (UserServiceImpl, SecurityServiceImpl, ProducerServiceImpl/ConsumerServiceImpl for message handling)
- **Repository Layer** ([src/main/java/com/visualpathit/account/repository/](src/main/java/com/visualpathit/account/repository/)): Spring Data JPA repositories for User and Role entities
- **Model Layer** ([src/main/java/com/visualpathit/account/model/](src/main/java/com/visualpathit/account/model/)): JPA entities (User, Role) mapped to MySQL tables via Hibernate
- **Utilities** ([src/main/java/com/visualpathit/account/utils/](src/main/java/com/visualpathit/account/utils/)): Infrastructure integration helpers (RabbitMqUtil, MemcachedUtils, ElasticsearchUtil)

### Configuration
Spring uses XML-based bean configuration ([src/main/webapp/WEB-INF/](src/main/webapp/WEB-INF/)) organized by concern:
- `appconfig-root.xml`: Root context, imports all configs, enables component scanning for `com.visualpathit.account.*`
- `appconfig-mvc.xml`: DispatcherServlet, JSP view resolver (`/WEB-INF/views/`), multipart file upload
- `appconfig-data.xml`: JPA/Hibernate config, MySQL datasource (BasicDataSource), transaction management
- `appconfig-rabbitmq.xml`: RabbitMQ connection factory, annotation-driven AMQP, listener container (3-10 concurrent consumers)
- `appconfig-security.xml`: Spring Security with BCrypt encoding (strength=11), form-based login, role-based access control

## Key Patterns & Conventions

### Spring Configuration Discovery
- XML bean definitions create beans referenced by `@Autowired` fields in services/controllers
- Properties from `application.properties` injected via `${property.name}` placeholders
- **Database**: MySQL 8 configured at `jdbc:mysql://db01:3306/accounts`
- **Caching**: Two memcached nodes (active: mc01:11211, standby: 127.0.0.2:11211)
- **Messaging**: RabbitMQ on rmq01:5672 with Producer/Consumer services
- **Elasticsearch**: vprofilenode cluster on localhost:9300

### Service Injection Pattern
```java
@Service
public class UserServiceImpl {
    @Autowired private UserRepository userRepository;
    @Autowired private RoleRepository roleRepository;
    @Autowired private BCryptPasswordEncoder bCryptPasswordEncoder;
    // Implementation uses injected dependencies
}
```

### Password Encoding
BCryptPasswordEncoder configured with strength=11 in security config; always encode passwords in save() methods before persistence.

### Error Handling
- Global exception handler: [GlobalExceptionHandler.java](src/main/java/com/visualpathit/account/GlobalExceptionHandler.java) with `@ControllerAdvice`
- Custom exception: [UserNotFoundException.java](src/main/java/com/visualpathit/account/UserNotFoundException.java)
- 500 error mapped to [/WEB-INF/views/error/500.jsp](src/main/webapp/WEB-INF/views/error/500.jsp)

## Build & Test Workflow

### Build Command
```bash
mvn clean install -DskipTests  # Build without tests (creates .war in target/)
mvn clean install              # Full build with unit tests
```

### Test Commands (from Jenkinsfile stages)
```bash
mvn test                    # Unit tests only
mvn verify -DskipUnitTests  # Integration tests only
mvn checkstyle:checkstyle   # Code analysis
```

### Build Output
WAR file: `target/vprofile-v2.war` (version from pom.xml: `<version>v2</version>`)

### CI/CD
Jenkinsfile defines pipeline stages: BUILD → UNIT TEST → INTEGRATION TEST → CODE ANALYSIS → PUBLISH TO NEXUS. Jenkins tools configured: MAVEN3, JDK17.

## Database & Schema

### Database Location
- MySQL database: `accounts` on host `db01`
- Credentials: admin/admin123 (from application.properties)

### Schema Import
```bash
mysql -u admin -p accounts < src/main/resources/db_backup.sql
```

### Entity Mapping
- `User` entity maps to `user` table
- `Role` entity maps to `role` table
- Repositories extend Spring Data JPA interfaces (findAll(), findById(), save(), etc. auto-generated)

## External Service Integration

### RabbitMQ Message Flow
- ProducerServiceImpl: Publishes messages via RabbitMqUtil
- ConsumerServiceImpl: Consumes messages (runs in listener container with configurable concurrency)
- Rabbitmq Controller endpoints: Message management UI

### Elasticsearch Indexing
- ElasticsearchUtil service handles document indexing/searching
- ElasticSearchController provides REST endpoints
- Results displayed in [elasticeSearchRes.jsp](src/main/webapp/WEB-INF/views/elasticeSearchRes.jsp)

### Memcached Caching
- MemcachedUtils service for distributed caching
- Failover between active (mc01) and standby nodes

## View Layer
JSPs located in [/WEB-INF/views/](src/main/webapp/WEB-INF/views/):
- `login.jsp`, `registration.jsp`: Authentication flows
- `user.jsp`, `userList.jsp`, `userUpdate.jsp`: User management
- `upload.jsp`: File upload
- `rabbitmq.jsp`, `rabbitmq-error.jsp`: Message queue UI
- `index_home.jsp`, `welcome.jsp`: Home pages

Static resources ([/resources/](src/main/webapp/resources/)): CSS (bootstrap, custom), JS, fonts, images.

## Dependencies to Know
- Spring 6.0.11, Spring Security 6.1.2, Spring Data JPA 3.1.2
- Hibernate 7.0.0.Alpha3, MySQL Connector 8.0.33
- Spring AMQP 3.1.6, Elasticsearch client 7.10.2
- Spymemcached 2.12.3 (memcached client)
- Jakarta EE 10.0.0 (Java 17 compatible)
- JUnit 4.13.2, Mockito 5.5.0 (testing)

## Development Notes
- **Java Version**: JDK 17 (maven.compiler.source/target)
- **JSP View Resolution**: Prefix `/WEB-INF/views/`, suffix `.jsp`
- **File Upload Limit**: 128KB max (via spring.servlet.multipart properties)
- **Security**: CSRF disabled; use form-based login only
- **Logging**: SLF4J via Logback; Spring Security set to DEBUG level
- **Deployment**: War packaged as `target/vprofile-v2.war`, deployed to Tomcat
