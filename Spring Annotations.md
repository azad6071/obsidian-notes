@Component: It's a spring managed bean. Single Instance by default
@Repository: @Component with extra exception handling at DB Level

Without @Repository:
	SQLException
	HibernateException

With @Repository:
	DataIntegrityViolationException
	DuplicateKeyException
	DataAccessResourceFailureException

@Controller: @Component, But if you use @RequestMapping inside a class, it would work only when it's annotated with @controller.

@Bean: @Bean can be used to create beans when written inside @Configuration.

We may want to create beans using this strategy in following conditions.

	3rd Party Classes
	Custom Construction Logic 
	(May be based on value of env variable we want diffrent beans)

```
@Component
public class AppConfig {

    @Bean
    public ThirdPartyService myService() {
        return new MyService();
    }

    @Bean
    public MyController myController() {
        return new MyController(myService());
    }
}

###
*
*
This may lead to creation of two beans. Because when you have @Bean inside @Component. Then it interactes with spring proxy only during creation. But when it's written inside @Configuration, spring proxy gets hit for each invocation of mySerivce ensuring single instance of ThirdPartyService.
*
*
###

```

@Transaction: 
Opens up a new transaction and commit's/rollback after invocation of the method. 

Cann't use with private methods.  
If you annotate both methods of the same class, one calling the other one, then it will run only in one transaction. 2nd method invocation won't trigger a new transaction. This is because a transaction can only be triggered by spring proxy not by internal method calls.

We can specify transaction isolation level.
Default value depends on the database, READ_COMMITTED  is default in postgres, it is REPEATABLE_READ in mysql.

@PostConstruct: When exactly does it run? (timeline)
Here’s the actual order for a bean:

1.  Bean instance is created (constructor runs)
2. Dependencies are injected (@Autowired, constructor injection, etc.)
3. @PostConstruct method is called 
4. Bean is ready for use
5. Application context continues starting
6. App becomes “started” / “ready”

@PreDestroy: When exactly does it run? (timeline)

For a managed Spring bean:
1. Application receives shutdown signal (SIGTERM, context close, etc.)
2. Spring starts shutting down the ApplicationContext
3. Bean destruction phase begins
4. `@PreDestroy` method is invoked 
5. Bean is destroyed
6. JVM exits

@Scheduled: If a bean is created AND scheduling is enabled (@EnableSchedulling), then any method annotated with `@Scheduled` will run repeatedly based on its configuration

@Async: If a bean is created, @EnableAsync is enabled, and an @Async-annotated method is invoked through a Spring proxy, then that method will run asynchronously

```
Caller
   │
   ▼
Proxy
   │
   ├── Normal method
   │      ▼
   │   invoke directly
   │
   └── @Async method
          ▼
TaskExecutor.submit(...)
          ▼
Worker Thread
          ▼
Actual method
```

@Data is Lombok annotation and a combination of 
	@Getter
	@Setter
	@ToString
	@EqualsAndHashCode
	@RequiredArgsConstructor

```
@Component
public class MyService {

    @Async
    public void asyncMethod() { }

    public void caller() {
        asyncMethod(); // self-invocation ❌
    }
}

###
 Here Async won't work. When caller method is invoked.
###
```


Spring Context: Running spring container that has information of all the beans

RestTemplate: Blocking, under maintenance only mode - no future development, non-reactive 
(1 Request = 1 Thread)
WebClient: non-blocking, Better Error handling, reactive
FeignCleint: blocking HTTP client ideal for internal service communication.



