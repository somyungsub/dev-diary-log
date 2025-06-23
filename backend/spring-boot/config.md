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

```java
@Configuration
@ConditionalOnProperty(prefix = "wne.his.routing", name = "enabled", havingValue = "false")
@RequiredArgsConstructor
@Slf4j
public class DefaultDataSourceConfig {

  private final DataSourceProperties properties;

  // TODO 정리, 임시
  @Value("${spring.datasource.jndi-name:#{null}}")
  private String jndiName;

  @PostConstruct
  void init() {
    log.info("Initializing default datasource : {}", properties);
  }

  @Bean
  @Primary
  @ConfigurationProperties(prefix = "spring.datasource")
  public DataSource defaultDataSource() {
    if (StringUtils.isNotEmpty(jndiName)) {
      return new JndiDataSourceLookup().getDataSource(jndiName);
    }
    return DataSourceBuilder.create()
            .type(HikariDataSource.class)
            .build();
  }
}
```

### config server 를 사용할 경우
config server가 로컬 파일보다 무조건 우선이기 때문에
동일한 키(spring.datasource.url 등)가 있으면 config server 값이 항상 이김

### 결론
- 키값 중복없이 따로 정리 되도록 설계가 필요
- local 용 config 파일도 필요한 상태
- 덮어쓰기가 안되니, bean 설정에서 따로 사용할 값들을 잘 구분해서 세팅? 좋은 방향같진 않음
- 차선책으로 `@Value`를 임시적으로 사용 고려. (이방법도 좋은 방향 같진 않음)