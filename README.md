# 운영환경별 환경변수 설정 연구
개발, 테스트, 운영 환경마다 다른 환경설정 파일 관리 방법  
구성요소: Spring Boot, Spring Cloud, Spring Bus  
목표: One Code Multi Use (단 한 번의 빌드로 모든 환경에 대응하기)  

## Spring Cloud Config Git Repository
Spring Cloud를 이용하기 위해서는 Git을 사용해야 한다. Git Repository를 생성하고 Spring 환경설정파일을 작성한다. (*e.g.* `master-config.yml`)

`master-config.yml` [link][0]

    config:
      servername: default
      info: This is default sever
      user: user001
      password: password001
      accessmessage: default msg - spring cloud config data
    ---

    spring:
      profiles: develop
    config:
      servername: develop
      info: This is develop sever
      user: user002
      password: password002
      accessmessage: welcome to develop server

    ---

    spring:
      profiles: production
    config:
      servername: production
      info: This is production sever
      user: user003
      password: password003
      accessmessage: welcome to production server

## Spring Profiles
Spring은 Profiles 기능으로 환경정보 설정 추상화(편리한 사용)를 지원한다. `@PropertySource` `@Value`를 이용하여 property를 손쉽게 설정할 수 있다.
##### @PropertySource [link][1]
    @Component
    @PropertySource(value = "file:C:/properties/application-test.properties", ignoreResourceNotFound = true)     // 테스트서버 환경변수
    @PropertySource(value = "file:/properties/application-production.properties", ignoreResourceNotFound = true) // 운영서버 환경변수
    public class ExternalProperty {
    }
##### @Value [link][2]
    //name에 바인딩된 데이터가 있으면 name필드에 static으로 대입되고 없으면 default name에 선언한 데이터가 대입된다.
    @Value("${name:default name}") String name;

## Spring Boot Profiles
spring boot는 profiles를 기본으로 사용한다. 그리고 `application.properties` `application.yml`에 입력한 환경변수를 자동으로 인식한다. `spring-boot-configuration-processor`를 사용하면 Java 객체 자동 매핑까지 지원한다. `WAS` 의 `VM` 옵션에 `-Dspring.profiles.active=production` 파리미터를 넘기면 yml의 `production` profiles로 프로젝트의 환경변수가 설정된다. bootRun시 log를 통해 적용된 profiles를 확인할 수 있다. `Intellij` 에서는 `File > Settings > Gradle > VM` 에 설정한다. `Run Configuration > VM` 은 정상작동하지 않는 경우가 있다. 실제 환경에서는 Tomcat이 설치된 서버에 환경변수를 설정한다.

`application.yml` [link][3]

    server:
      port: 8000
    protest:
      message: This is default profiles msg
      port: 8000

    ---

    spring:
      profiles: develop
    server:
      port: 8100
    protest:
      message: This is develop profiles data
      port: 8100

    ---

    spring:
      profiles: production
    server:
      port: 8200
    protest:
      message: This is production profiles data
      port: 8200

##### @ConfigurationProperties [link][4]
check `error` : yml의 첫 property 단계를 읽지 못하는 경우가 있다. `null` 발생.

      @ConfigurationProperties(prefix = "protest")
      public class ProfilesConfig {

          private String message;
          private String port;

          //getter and setter
      }

##### @EnableConfigurationProperties [link][5]
`application.yml` 데이터가 바인딩된 ProfilesConfig를 사용하게 해준다.

    @RestController
    @EnableConfigurationProperties(ProfilesConfig.class)
    public class ProfilesController {

        @Autowired
        private ProfilesConfig profilesConfig;

        @RequestMapping(value = "/check", method = RequestMethod.GET)
        public ProfilesConfig check() {
            return profilesConfig;
        }
    }

## Spring Cloud Server
Spring Cloud 프로젝트는 Git 저장소에 Config 데이를 Server가 바라보게 한다. Server는 Git Config 데이터가 업데이트되면 자동으로 업데이트 한다. `spring-cloud-config-server` defendency를 추가해서 사용한다.

`application.yml` [link][6]

    server:
      port: 8888
    spring:
      cloud:
        config:
          server:
            git:
              uri: https://github.com/kangyongho/spring-cloud-config

##### @EnableConfigServer [link][7]
spring cloud server로 동작하게 한다.

    @SpringBootApplication
    @EnableConfigServer
    public class SpringCloudApplication {
    	public static void main(String[] args) {
    		SpringApplication.run(SpringCloudApplication.class, args);
    	}
    }


## Spring Cloud Client
Spring Cloud Client는 Server를 바라본다. 따라서 여러개의 분산 서비스 애플리케이션을 한 번에 관리할 수 있다. `micro architecture` `microservice` 가 가능하다. Git Config가 업데이트되면 `refresh` 명령으로 서버의 재시작 없이도 환경변수를 초기화 할 수 있다. `refresh` 는 `POST` 로 해야한다. 그리고 추가적으로 Spring `spring-boot-starter-actuator` defendency를 classpath에 넣어줘야 한다.

`bootstrap.yml` [link][8]  
Spring Cloud에 환경설정을 할 때는 `application.*` 보다 `bootstrap.yml`을 사용하는것이 확실한 설정이 가능하다. 인식을 못하는 경우가 있었다. Spring Cloud Server 경로와 환경변수 파일명을 `uri` `name` 에 입력한다.

    spring:
      cloud:
        config:
          uri: http://mirlang2.ddns.net/config-server
          name: master-config

    management:
      security:
        enabled: false

`management.security.enabled: false` 설정은 Spring Cloud의 보안이 default로 설정되어 있기 때문에 false로 변경해서 `refresh` 명령을 내렸을 때 인증거부가 나지 않도록 한다. `production` 환경에서 실제로 서비스하기 위해서는 Spring Security 등 인증 체계를 반드시 사용해야 한다.  

*i.e.* `error message` with out `management.security.enabled: false` on POST MAN

    {
      "timestamp": 1492934517385,
      "status": 401,
      "error": "Unauthorized",
      "message": "Full authentication is required to access this resource.",
      "path": "/refresh"
    }

*i.e.* `error message` with out `management.security.enabled: false` on Tomcat

    s.b.a.e.m.MvcEndpointSecurityInterceptor : Full authentication is required to access actuator endpoints. Consider adding Spring Security or set 'management.security.enabled' to false.

`refresh`  
크롬 확장도구 POST MAN을 이용해서 POST `refresh` 요청을 한다. `localhost:8080/refresh` 호출 후 Tomcat log를 보면 재설정 로그를 확인할 수 있다.

## Spring Cloud Bus
Spring Cloud Bus를 사용하려면 `spring-cloud-starter-bus-amqp` defendency를 추가하고 RabbitMQ 설치가 필요하다. Spring Profiles, Spring Cloud를 이용하면 환경설정용 Config Server를 상단에 두고 `microservice` `distributed` 환경에 간편히 설정정보를 업데이트하고 관리할 수 있다. 그러나 업데이트를 위해서는 수 맣은 Client에게 `refresh` 명령을 수동으로 내려야 한다. 물론 스크립트나 간단한 메서드를 정해두고 사용할 수도 있지만 부지런함은 다른 곳에 사용하자.  
Spring Cloud Bus는 AMQP 프로토콜을 지원하는 RabbitMQ 메시징 오픈소스 서비스를 이용하여 단 한번의 `refresh` 로 같은 서버를 바라보는 Client에게 환경정보 업데이트를 가능하게 지원한다.

##### All Client 업데이트
    localhost:8080/bus/refresh
    아래 'refresh' log 시간을 보면 Client01, Client02가 동시에 업데이트 된 것을 볼 수 있다.

`refresh` Client 01

    2017-05-01 14:18:13.648  INFO 11116 --- [DKLSoTuEGeEvQ-1] c.c.c.ConfigServicePropertySourceLocator : Fetching config from server at: http://mirlang2.ddns.net/config-server
    2017-05-01 14:18:14.455  INFO 11116 --- [DKLSoTuEGeEvQ-1] c.c.c.ConfigServicePropertySourceLocator : Located environment: name=master-config, profiles=[develop], label=null, version=null, state=null
    2017-05-01 14:18:14.456  INFO 11116 --- [DKLSoTuEGeEvQ-1] b.c.PropertySourceBootstrapConfiguration : Located property source: CompositePropertySource [name='configService', propertySources=[MapPropertySource [name='https://github.com/kangyongho/springcloudconfig/master-config.yml#develop'], MapPropertySource [name='https://github.com/kangyongho/springcloudconfig/master-config.yml']]]
    2017-05-01 14:18:14.458  INFO 11116 --- [DKLSoTuEGeEvQ-1] o.s.boot.SpringApplication               : The following profiles are active: develop
    2017-05-01 14:18:14.460  INFO 11116 --- [DKLSoTuEGeEvQ-1] s.c.a.AnnotationConfigApplicationContext : Refreshing org.springframework.context.annotation.AnnotationConfigApplicationContext@14cd3edc: startup date [Mon May 01 14:18:14 KST 2017]; parent: org.springframework.context.annotation.AnnotationConfigApplicationContext@44e065cb
    2017-05-01 14:18:14.483  INFO 11116 --- [DKLSoTuEGeEvQ-1] o.s.boot.SpringApplication               : Started application in 4.536 seconds (JVM running for 320.182)
    2017-05-01 14:18:14.484  INFO 11116 --- [DKLSoTuEGeEvQ-1] s.c.a.AnnotationConfigApplicationContext : Closing org.springframework.context.annotation.AnnotationConfigApplicationContext@14cd3edc: startup date [Mon May 01 14:18:14 KST 2017]; parent: org.springframework.context.annotation.AnnotationConfigApplicationContext@44e065cb
    2017-05-01 14:18:14.815  INFO 11116 --- [DKLSoTuEGeEvQ-1] o.s.cloud.bus.event.RefreshListener      : Received remote refresh request. Keys refreshed [config.accessmessage]

`refresh` Client 02

    2017-05-01 14:18:13.828  INFO 10824 --- [io-9020-exec-10] c.c.c.ConfigServicePropertySourceLocator : Fetching config from server at: http://mirlang2.ddns.net/config-server
    2017-05-01 14:18:15.082  INFO 10824 --- [io-9020-exec-10] c.c.c.ConfigServicePropertySourceLocator : Located environment: name=master-config, profiles=[production], label=null, version=null, state=null
    2017-05-01 14:18:15.082  INFO 10824 --- [io-9020-exec-10] b.c.PropertySourceBootstrapConfiguration : Located property source: CompositePropertySource [name='configService', propertySources=[MapPropertySource [name='https://github.com/kangyongho/springcloudconfig/master-config.yml#production'], MapPropertySource [name='https://github.com/kangyongho/springcloudconfig/master-config.yml']]]
    2017-05-01 14:18:15.092  INFO 10824 --- [io-9020-exec-10] o.s.boot.SpringApplication               : The following profiles are active: production
    2017-05-01 14:18:15.096  INFO 10824 --- [io-9020-exec-10] s.c.a.AnnotationConfigApplicationContext : Refreshing org.springframework.context.annotation.AnnotationConfigApplicationContext@5165e373: startup date [Mon May 01 14:18:15 KST 2017]; parent: org.springframework.context.annotation.AnnotationConfigApplicationContext@3cc7cd64
    2017-05-01 14:18:15.120  INFO 10824 --- [io-9020-exec-10] o.s.boot.SpringApplication               : Started application in 5.271 seconds (JVM running for 229.849)
    2017-05-01 14:18:15.121  INFO 10824 --- [io-9020-exec-10] s.c.a.AnnotationConfigApplicationContext : Closing org.springframework.context.annotation.AnnotationConfigApplicationContext@5165e373: startup date [Mon May 01 14:18:15 KST 2017]; parent: org.springframework.context.annotation.AnnotationConfigApplicationContext@3cc7cd64
    2017-05-01 14:18:15.746  INFO 10824 --- [io-9020-exec-10] o.s.cloud.bus.event.RefreshListener      : Received remote refresh request. Keys refreshed [config.accessmessage]

##### RabbitMQ 설치
RabbitMQ는 Erlang 언어 설치를 전제조건으로 한다. RabbitMQ 홈페이지를 참고해서 Erlang 부터 설치하고 RabbitMQ 프로그램을 설치하면 된다. RabbitMQ가 없거나 접속이 불가능하면 Spring Cloud Bus Tomcat error log를 보면 다음과 같다.  

`RabbitMQ` 설치 전 `Tomcat error log`

    2017-04-26 19:40:20.510  WARN 4688 --- [           main] o.s.amqp.rabbit.core.RabbitAdmin         : Failed to declare exchange: Exchange [name=springCloudBus, type=topic, durable=true, autoDelete=false, internal=false, arguments={}], continuing... org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused: connect

    2017-04-26 19:40:21.653  WARN 4688 --- [           main] o.s.amqp.rabbit.core.RabbitAdmin         : Failed to declare exchange: Exchange [name=springCloudBus, type=topic, durable=true, autoDelete=false, internal=false, arguments={}], continuing... org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused: connect

    2017-04-26 19:40:22.655  WARN 4688 --- [           main] o.s.amqp.rabbit.core.RabbitAdmin         : Failed to declare queue: Queue [name=springCloudBus.anonymous.uGMF6uS8QRK1R5vwX_4n3w, durable=false, autoDelete=true, exclusive=true, arguments={}], continuing... org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused: connect

    2017-04-26 19:40:23.666  WARN 4688 --- [           main] o.s.amqp.rabbit.core.RabbitAdmin         : Failed to declare binding: Binding [destination=springCloudBus.anonymous.uGMF6uS8QRK1R5vwX_4n3w, exchange=springCloudBus, routingKey=#], continuing... org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused: connect

    2017-04-26 19:40:29.743  WARN 4688 --- [RK1R5vwX_4n3w-1] o.s.a.r.l.SimpleMessageListenerContainer : Consumer raised exception, processing can restart if the connection factory supports it. Exception summary: org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused: connect

    2017-04-26 19:40:35.764  WARN 4688 --- [RK1R5vwX_4n3w-2] o.s.a.r.l.SimpleMessageListenerContainer : Consumer raised exception, processing can restart if the connection factory supports it. Exception summary: org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused: connect

    2017-04-26 19:40:41.779  WARN 4688 --- [RK1R5vwX_4n3w-3] o.s.a.r.l.SimpleMessageListenerContainer : Consumer raised exception, processing can restart if the connection factory supports it. Exception summary: org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused: connect

##### RabbitMQ SpringCloudBus Exchange
RabbitMQ는 AMQP 프로토콜을 사용하는 메시징 broker다. 핵심 개념으로는 producer, exchange, queue, consumer가 있으며 `exchange`가 가장 중요하다. `exchange` 종류에는 `direct` `topic` `header` `fanout` 이 있다. Spring Cloud Bus에서는 `topic` 을 사용한다. Spring Cloud Bus bootRun을 하고 RabbitMQ 정보를 조회해보면 `exchange` Name이 `SpringCloudBus` 로 Type이 `topic` 으로 생성된 것을 확인할 수 있다.

##### Others
Spring Cloud를 이용하면 환경정보를 Config 서버를 이용해서 집중관리하고 빠른 적용이 가능하다. 하지만 Client 서버 Tomcat마다 `spring.profiles.active` 환경변수를 설정해줘야 `profiles` 적용이 가능한 문제가 남아있다.

# 프로젝트 링크
* [spring-property][9]
* [spring-profiles][10]
* [spring-cloud-config-server][11]
* [spring-cloud-config-client][12]
* [spring-cloud-bus-client][13]
* [spring-cloud-config][14]

[0]: https://github.com/kangyongho/spring-cloud-config/blob/master/master-config.yml
[1]: https://github.com/kangyongho/spring-property/blob/master/src/main/java/com/example/config/ExternalProperty.java
[2]: https://github.com/kangyongho/spring-property/blob/master/src/main/java/com/example/controller/BasicController.java
[3]: https://github.com/kangyongho/spring-profiles/blob/master/src/main/resources/application.yml
[4]: https://github.com/kangyongho/spring-profiles/blob/master/src/main/java/com/example/ProfilesConfig.java
[5]: https://github.com/kangyongho/spring-profiles/blob/master/src/main/java/com/example/ProfilesController.java
[6]: https://github.com/kangyongho/spring-cloud-config-server/blob/master/src/main/resources/application.yml
[7]: https://github.com/kangyongho/spring-cloud-config-server/blob/master/src/main/java/com/example/SpringCloudApplication.java
[8]: https://github.com/kangyongho/spring-cloud-bus-client/blob/master/src/main/resources/bootstrap.yml
[9]: https://github.com/kangyongho/spring-property
[10]: https://github.com/kangyongho/spring-profiles
[11]: https://github.com/kangyongho/spring-cloud-config-server
[12]: https://github.com/kangyongho/spring-cloud-config-client
[13]: https://github.com/kangyongho/spring-cloud-bus-client
[14]: https://github.com/kangyongho/spring-cloud-config
