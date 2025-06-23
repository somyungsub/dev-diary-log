# 이슈
## 1.내장/외장 톰캣 Component 스캔
내장톰캣 사용할 때, 컴포넌트 스캔이 잘 돼서 문제가 되지 않는데, 외장톰캣에서 문제가 된 케이스

```java
@Component
@ConditionalOnBean(RouterDatabaseConfigProperty.class)
public class DataSourceContextHolder {
  private static final ThreadLocal<String> context = new ThreadLocal<>();
  
  public void setRoutingKey(String lookupKey) {
    clear();
    context.set(lookupKey);
  }

  public String getRoutingKey() {
    return context.get();
  }

  public void clear() {
    context.remove();
  }
}

@Configuration
@ConditionalOnProperty(prefix = "wne.his.routing", name = "enabled", havingValue = "true")
@EnableConfigurationProperties(RouterDatabaseConfigProperty.class)
@RequiredArgsConstructor
@Slf4j
public class RouterDatabaseConfig {
  private final DataSourceContextHolder dataSourceContextHolder;
  private final RouterDatabaseConfigProperty routerDatabaseConfigProperty;

  @PostConstruct
  public void init() {
    log.info("RouterDatabaseConfig loaded");
  }

  @Bean
  @Primary
  public DataSource lazyDataSource(@Qualifier("routingDataSource") DataSource dataSource) {
    return new LazyConnectionDataSourceProxy(dataSource);
  }

  @Bean
  RoutingDataSource routingDataSource(RouterDatabaseConfigProperty props) {
    RoutingDataSource routingDataSource = new RoutingDataSource(dataSourceContextHolder, routerDatabaseConfigProperty);
    Map<Object, Object> targetDataSourceMap = targetDataSourceMap(props);
    routingDataSource.setTargetDataSources(targetDataSourceMap);

    // 기본은 DefaultKeyName 으로 설정하거나 fallback 지정
    routingDataSource.setDefaultTargetDataSource(targetDataSourceMap.get(routerDatabaseConfigProperty.getDefaultKeyName()));
    return routingDataSource;
  }

  // ... 중략
}
```
### 해결
@Bean을 @Configuration 에서 정의하여 처리
```java

@Configuraion
public class RouterDatabaseConfig {
  //...
  @Bean
  DataSourceContextHolder dataSourceContextHolder() {
    return new DataSourceContextHolder();
  }
  //...
}
```
### 결론
- 내장 톰캣
SpringApplication 이 톰캣을 포함해서 부트스트랩 → Spring Context가 무조건 먼저 초기화 → @Component / @Bean 등 모든 스프링 빈이 예상대로 동작
- 외장 톰캣(WAR 배포)
외부 WAS(Tomcat)가 먼저 동작 → web.xml → ServletContainerInitializer → SpringServletContainerInitializer → SpringBootServletInitializer → ApplicationContext 초기화
즉, Spring Context 초기화 시점이 더 복잡하고 WAS 컨테이너와 라이프사이클이 얽혀 있음
특히 @ConditionalOnBean, @ConditionalOnProperty 같은 조건부 설정은 의존 빈/속성이 컨텍스트 초기화 순서와 얽히면 쉽게 조건 불일치로 간주되어 빈 생성이 안됨


## 2.컨피그 우선순위
Spring Boot의 프로퍼티 우선순위는 공식 문서 기준:
- https://docs.spring.io/spring-boot/reference/features/external-config.html#features.external-config

```text
(높음)
1 Command Line Arguments
2 Java System Properties (-Dkey=value)
3 OS Environment Variables (export KEY=value)
4 JNDI
5 ServletContext init params
6 ServletConfig init params
7 Spring Cloud Config Server
8 application.properties / application.yml (로컬 파일)
9 Profile-specific application-xxx.yml
10 @PropertySource in code
11 @ConfigurationProperties defaults in 코드

```

| 우선순위                                                    | 이름 |
| ------------------------------------------------------- | -- |
| 1️⃣ Command Line                                        |    |
| 2️⃣ Env Vars                                            |    |
| 3️⃣ JNDI                                                |    |
| 4️⃣ **local `application-routing-db.yml`** (활성 profile) |    |
| 5️⃣ **configService\:application.yml** (remote base)    |    |
| 6️⃣ local `application.yml`                             |    |
그래서 DataSourceProperties 는:
remote application.yml 에 같은 키가 있어도
local profile yml(application-routing-db.yml) 값이 이김

> 공식 문서에도
> Profile-specific application properties take precedence over non-profile-specific files regardless of origin.

| 상황                                                                         | 우선순위                          |
| -------------------------------------------------------------------------- | ----------------------------- |
| remote `application.yml` vs. local `application-routing-db.yml`            | **local `routing-db.yml` 이김** |
| remote `application-routing-db.yml` vs. local `application-routing-db.yml` | remote 이김 (둘 다 profile일 경우)   |
| local `application.yml` vs. remote `application.yml`                       | remote 이김                     |

- Spring Boot는 항상 profile-specific YML을 base YML보다 우선 적용
- configserver가 가져온 remote YML은 base PropertySource임
  - configserver 정보가 어디에 있냐에 따라 우선순위 변경됨
- remote에 profile YML이 없으면 local profile이 무조건 이김
  - 원래대로라면 : remote profile > local profile > remote base > local base
  - remote profile 없으므로 : local profile > remote base > local base

- 정확한 이유
- PropertySource 우선순위 표 자체는 맞는데,
- Spring Boot의 profile 계층 합산 규칙 때문에
  - remote base
  - local profile

이렇게 source가 섞여 있으면,
profile이 base를 덮는 것이 최종 머지 단계에서 반영됩니다.

요약
- PropertySource 우선순위는 remote가 먼저이나,
- Spring Boot의 프로파일 계층 머지 규칙이 Local profile > Remote base 를 보장


```java
@Configuration
@ConditionalOnProperty(prefix = "wne.his.routing", name = "enabled", havingValue = "false")
@RequiredArgsConstructor
@Slf4j
public class DefaultDataSourceConfig {
  private final DataSourceProperties properties;

  @Bean
  @ConfigurationProperties(prefix = "spring.datasource.hikari")
  public HikariConfig hikariConfig() {
    return new HikariConfig();
  }

  @Bean
  @Primary
  public DataSource defaultDataSource(HikariConfig hikariConfig) {
    if (StringUtils.isNotEmpty(properties.getJndiName())) {
      return new JndiDataSourceLookup().getDataSource(properties.getJndiName());
    }

    return createDataSource(hikariConfig);
  }

  private DataSource createDataSource(HikariConfig hikariConfig) {
    HikariDataSource dataSource = properties.initializeDataSourceBuilder()
            .type(HikariDataSource.class)
            .build();

    dataSource.setPoolName(CommonUtil.defaultValue(hikariConfig.getPoolName(), "his-pool"));
    dataSource.setSchema(CommonUtil.defaultValue(hikariConfig.getSchema(), "his"));
    dataSource.setMaximumPoolSize(CommonUtil.defaultValue(hikariConfig.getMaximumPoolSize(), 10));
    dataSource.setConnectionTimeout(CommonUtil.defaultValue(hikariConfig.getConnectionTimeout(), 10000L));
    return dataSource;
  }

}
```

```yaml
# application-routing-db.yml
spring:
  config:
    activate:
      on-profile: local
  datasource:
    url: jdbc:postgresql://localhost:5432/test?currentSchema=his
    username: postgres
    password: password
    driver-class-name: org.postgresql.Driver
    jndi-name: jdbc/mydb3
    hikari:
      schema: his
      maximum-pool-size: 10
      pool-name: designer-local-pool
      connection-timeout: 10000
```
- spring.datasource.url -> 키 중복시, 우선순위 중요
- DataSourceProperties 값 세팅되는 정보가 달라짐

### Config Server 를 사용할 경우
config server가 어디에서 import 하느냐에 따라 우선선위가 달라질 수 있음.
동일한 키(spring.datasource.url 등)가 존재시, 경우에 따라 config server 값이 달라 질 수 있음

### 결론
- 키값 중복없이 따로 정리 되도록 설계가 필요
- local 용 config 파일도 필요한 상태
- DataSourceProperties 값이 채워지는 순서가 중요
  - Config Server 값 -> 어디에 import?
  - application.yml (base?)
  - application-특정.yml (특정 프로파일 활성?)
  - 우선순위 특정 > base
  - 따라서, 현재상태에서 configserver의 import 된 키를 따로 오버라이드 해서 처리하려면, 특정 profile 설정후 설정하거나, 키값 따로 주거나 
- 원래대로라면 Spring Cloud Config Server > Local 특정 > Local Base
  - bootstrap.yml은 ApplicationContext 생성 전에 먼저 로드됨

1.bootstrap.yml 사용 시 (예전 방식)
```yaml
# bootstrap.yml
spring:
  cloud:
    config:
      uri: http://configserver:8888
```

2.application.yml + spring.config.import (Spring Boot 2.4+)
```yaml
# application.yml
spring:
  config:
    import: optional:configserver:http://configserver:8888
```
import는 application.yml의 일부이므로, application.yml 처리 흐름에 따라 병합됨.
import는 PropertySourceLoader에서 처리 → PropertySource로 붙음.
import는 Local YML보다 우선 등록됨 → 그래서 Config Server > Local YML 구조는 유지됨.

3.spring.config.import에 on-profile 사용
```yaml
spring:
  config:
    activate:
      on-profile: local
    import: optional:configserver:http://...
```
on-profile 조건이 만족해야 import가 작동함.
해당 profile이 아닐 땐 아예 Config Server PropertySource가 등록되지 않음.
즉, profile별로 Config Server 우선순위를 on/off 할 수 있음.

| 설정 위치                                                       | 동작                                         |
| ----------------------------------------------------------- | ------------------------------------------ |
| `bootstrap.yml`                                             | 항상 가장 먼저 → 무조건 최우선                         |
| `application.yml` `import:`                                 | Local YML보다 먼저 붙음 → Config Server 우선 유지    |
| `import` + `on-profile`                                     | profile 조건 만족 시에만 붙음 → 조건 안 맞으면 아예 없음      |
| Remote `application.yml` vs Local `application-profile.yml` | **Local profile이 base를 덮음** (profile rule) |

Config Server PropertySource는 보통 Local YML보다 우선.
단, Local profile YML은 Remote base보다 항상 우선.
import나 bootstrap.yml 위치/조건에 따라 붙는 타이밍으로 우선순위를 제어 가능!
- 경우에 따라 달라 질 수 있다? 