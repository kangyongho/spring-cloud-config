# 운영환경별 환경변수 설정 연구
개발, 테스트, 운영 환경마다 다른 환경설정 파일 관리 방법  
구성요소: Spring Boot, Spring Cloud, Spring Bus  
목표: One Code Multi Use (단 한 번의 빌드로 모든 환경에 대응하기)

## Spring Cloud Config Git Repository
Spring Cloud를 이용하기 위해서는 Git을 사용해야 한다. Git Repository를 생성하고 Spring 환경설정파일을 작성한다. (*e.g.* `application.properties` or `application.yml`)

## Spring Profiles
Spring은 [Profiles][1] 기능으로 환경정보 설정 추상화(편리한 사용)를 지원한다. `@PropertySource`, `@Value`를 이용하여 property를 손쉽게 설정할 수 있다.
###### @PropertySource
    @Component
    @PropertySource(value = "file:C:/properties/application-test.properties", ignoreResourceNotFound = true)     // 테스트서버 환경변수
    @PropertySource(value = "file:/properties/application-production.properties", ignoreResourceNotFound = true) // 운영서버 환경변수
    public class ExternalProperty {
    }
###### @Value
    //name에 바인딩된 데이터 있으면 name필드에 static으로 대입되고, 없으면 default name에 선언한 데이터가 대입된다.
    @Value("${name:default name}") String name;

## Spring Boot Profiles
spring boot에서는 profiles를 기본으로 사용한다. 그리고 `application.properties` `application.yml`에 입력한 환경변수를 자동으로 인식한다. `spring-boot-configuration-processor`를 사용하면 Java 객체에 자동 매핑까지 지원한다. `-Dspring.profiles.active=production` 으로 `WAS` 에 `VM` 옵션으로 파리미터를 넘기면 yml의 production profiles로 프로젝트의 환경변수가 설정된다. bootRun시 log를 통해 적용된 profiles를 확인할 수 있다. `Intellij` 에서는 `File > Settings > Gradle > VM` 에 설정한다. `Run Configuration > VM` 은 정상작동하지 않는 경우가 있다.

`application.yml`  

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

###### @ConfigurationProperties

      @ConfigurationProperties(prefix = "protest")
      public class ProfilesConfig {

          private String message;
          private String port;

          //getter and setter
      }

###### @ConfigurationProperties

    @RestController
    @EnableConfigurationProperties(ProfilesConfig.class)
    public class ProfilesController {

        @Autowired
        private ProfilesConfig profilesConfig;

        @RequestMapping(value = "/check")
        public String check() {
            return profilesConfig.getMessage();
        }
    }

## Spring Cloud Server
Spring Cloud 프로젝트는 Git 저장소에 Config 데이를 Server가 바라보게 한다. Server는 Git Config 데이터가 업데이트되면 자동으로 업데이트 한다. `spring-cloud-config-server` defendency를 추가해서 사용한다.

`application.yml`

    server:
      port: 8888
    spring:
      cloud:
        config:
          server:
            git:
              uri: https://github.com/kangyongho/spring-cloud-config

###### @EnableConfigServer
    @SpringBootApplication
    @EnableConfigServer //spring cloud server로 동작하게 한다.
    public class SpringCloudApplication {
    	public static void main(String[] args) {
    		SpringApplication.run(SpringCloudApplication.class, args);
    	}
    }


## Spring Cloud Client
Spring Cloud Client는 Server를 바라본다. 따라서 여러개의 분산 서비스 애플리케이션을 한 번에 관리할 수 있다. `micro architecture` `microservice` 가 가능하다. Git Config가 업데이트되면 `refresh` 명령으로 서버의 재시작 없이도 환경변수를 초기화 할 수 있다. `refresh` 는 `POST` 로 해야한다. 그리고 추가적으로 Spring `spring-boot-starter-actuator` defendency를 classpath에 넣어줘야 한다.

`bootstrap.yml`  
Spring Cloud에 환경설정을 할 때는 `application.*` 보다 `bootstrap.yml`을 사용하는것이 확실한 설정이 가능한 듯 하다. 인식을 못하는 경우가 있었다.

    spring:
      cloud:
        config:
          uri: http://mirlang2.ddns.net/config-server
          name: master-config

    management:
      security:
        enabled: false

`management.security.enabled: false` 설정은 Spring Cloud의 보안이 default로 설정되어 있기 때문에 false로 변경해서 `refresh` 명렬을 내렸을 때 인증거부가 나지 않도록 한다. `production` 환경에서 실제로 서비스하기 위해서는 Spring Security 등 인증 체계를 반드시 사용해야 한다.  

*i.e.* `error message`

    {
      "timestamp": 1492934517385,
      "status": 401,
      "error": "Unauthorized",
      "message": "Full authentication is required to access this resource.",
      "path": "/refresh"
    }

`refresh`  
크롬 확장도구 POST MAN을 이용해서 POST `refresh` 요청을 한다. `localhost:8080/refresh` 호출 후 Tomcat log를 보면 재설정 로그를 확인할 수 있다.

## Spring Cloud Bus
Spring Profiles, Spring Cloud를 이용하면 환경설정용 Config Server를 상단에 두고 `microservice` `distributed` 환경에 간편히 설정정보를 업데이트하고 관리할 수 있다. 그러나 업데이트를 위해서는 수 맣은 Client에게 `refresh` 명령을 수동으로 내려야 한다. 물론 스크립트나 간단한 메서드를 정해두고 사용할 수도 있지만 부지런함은 다른 곳에 사용하자.
Spring Cloud Bus는 AMQP 프로토콜을 지원하는 RabbitMQ 메시징 오픈소스 서비스를 이용하여 단 한번의 `refresh` 로도 같은 서버를 바라보는 Client에게 환경정보 업데이트를 가능하게 지원한다. `spring-cloud-starter-bus-amqp` defendency를 추가하기만 하면 된다. 단 RabbitMQ 설치를 전제로 한다.

###### RabbitMQ 설치
RabbitMQ는 Erlang 언어 설치를 전제조건으로 한다. RabbitMQ 홈페이지를 참고해서 Erlang 부터 설치하고 RabbitMQ 프로그램을 설치하면 된다. RabbitMQ Tutorial을 모두 진행해보고 Spring Cloud Bus Tomcat log를 보면 다음과 같다.  

`RabbitMQ` 설치 전 `Tomcat error log`

    2017-04-26 19:40:20.510  WARN 4688 --- [           main] o.s.amqp.rabbit.core.RabbitAdmin         : Failed to declare exchange: Exchange [name=springCloudBus, type=topic, durable=true, autoDelete=false, internal=false, arguments={}], continuing... org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused: connect

    2017-04-26 19:40:21.653  WARN 4688 --- [           main] o.s.amqp.rabbit.core.RabbitAdmin         : Failed to declare exchange: Exchange [name=springCloudBus, type=topic, durable=true, autoDelete=false, internal=false, arguments={}], continuing... org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused: connect

    2017-04-26 19:40:22.655  WARN 4688 --- [           main] o.s.amqp.rabbit.core.RabbitAdmin         : Failed to declare queue: Queue [name=springCloudBus.anonymous.uGMF6uS8QRK1R5vwX_4n3w, durable=false, autoDelete=true, exclusive=true, arguments={}], continuing... org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused: connect

    2017-04-26 19:40:23.666  WARN 4688 --- [           main] o.s.amqp.rabbit.core.RabbitAdmin         : Failed to declare binding: Binding [destination=springCloudBus.anonymous.uGMF6uS8QRK1R5vwX_4n3w, exchange=springCloudBus, routingKey=#], continuing... org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused: connect

    2017-04-26 19:40:29.743  WARN 4688 --- [RK1R5vwX_4n3w-1] o.s.a.r.l.SimpleMessageListenerContainer : Consumer raised exception, processing can restart if the connection factory supports it. Exception summary: org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused: connect

    2017-04-26 19:40:35.764  WARN 4688 --- [RK1R5vwX_4n3w-2] o.s.a.r.l.SimpleMessageListenerContainer : Consumer raised exception, processing can restart if the connection factory supports it. Exception summary: org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused: connect

    2017-04-26 19:40:41.779  WARN 4688 --- [RK1R5vwX_4n3w-3] o.s.a.r.l.SimpleMessageListenerContainer : Consumer raised exception, processing can restart if the connection factory supports it. Exception summary: org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused: connect

[1]: http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#beans-environment
