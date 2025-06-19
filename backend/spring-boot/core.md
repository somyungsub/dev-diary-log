# 이슈
## 내장/외장 톰캣 Component 스캔
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