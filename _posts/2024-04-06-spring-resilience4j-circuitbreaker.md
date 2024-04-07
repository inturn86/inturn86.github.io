---
title: (Spring) Resilience4j Circuitbreaker
date: 2024-04-06 21:55:00 +0800
categories:
  - Spring
tags:
  - resilience4j
  - circuitbreaker
  - msa
  - fallback
pin: false
img_path:
---
### Resilience4j 란?

resilience4j 는 Netflix Hystrix에서 영감을 받았지만 함수형 프로그래밍을 위해 설계된 내결함성 라이브러리이며, Resilience(회복력)과 Java가 합쳐진 이름이다.
아래 6가지 기능을 제공한다.

#### Code Modules
- resilience4j-circuitbreaker: Circuit breaking
- resilience4j-ratelimiter: Rate limiting
- resilience4j-bulkhead: Bulkheading
- resilience4j-retry: Automatic retrying (sync and async)
- resilience4j-cache: Result caching
- resilience4j-timelimiter: Timeout handling

해당 글에서는 circuitbreaker에 대해서 알아보고 테스트해보도록 하겠다.

### Circuitbreaker가 필요한 이유

애플리케이션의 각각의 도메인이나 기능을 세분하하고 분산 서버로 아키텍쳐링하는 구조가 늘어나고 있다.

이에 외부 서비스에 대한 의존도가 증가함에 따라 외부 서비스의 장애가 발생할 경우 다른 서비스에도 장애가 전파되는 것을 막기위해 circuitbreaker를 사용한다.

하나의 외부 서비스가 장애가 발생하였을 경우 Connection Time 에 대한 Latency가 증가하고 상황이 지속될 경우 Thread 반환이 지연되어 해당 서비스 또한 장애가 발생할 수 있다. 이때 circuitbreaker를 사용하여 빠른 실패 처리를 통해 Thread를 반환시키고 실패한 이력을 Fallback으로 관리하여 대응할 수 있도록 한다.

### Circuitbreaker 란?

circuitbreaker를 직역하면 회로차단기 이다. 과전류가 발생할 경우 회로를 차단 시켜 이후에 발생할 수 있는 문제를 방지해준다.

시스템의 관점에서 외부 서비스의 장애를 감지하고 더 이상 요청을 보내지 않도록 차단하여, 장애가 퍼지지 않도록 격리시킨다.

circuitbreaker는 아래 세 가지의 정상상태(CLOSED, OPEN, HALF_OPEN)과 두 가지 특수 상태(DISABLED, FORCE_OPEN)로 관리된다.

![426](https://files.readme.io/39cdd54-state_machine.jpg "state_machine.jpg")

|    상태     | 설명                                                                                                                   |
| :-------: | :------------------------------------------------------------------------------------------------------------------- |
|  CLOSED   | 회로 차단기가 동작하지 않는 상태(외부 서비스 호출)<br>임계값 제한을 초과하면 회로 차단기가 작동하며 OPEN 상태로 전환된다.                                            |
|   OPEN    | 회로 차단기가 동작하는 상태(외부 서비스가 호출되지 않음)<br>실행시도 없이 오류(fallback)와 함께 반환된다.                                                   |
| HALF_OPEN | OPEN 상태에서 일정 시간(설정시간)이 지난 후 HALF_OPEN 상태로 전환<br>해당 상태에서 일정 횟수만큼 외부 서비스를 호출하고 임계값 제한을 초과하면 OPEN 그렇지 않으면 CLOSED로 전환된다. |

### Circuitbreaker 적용

개념을 알아보았으니 실제 적용을 해보자.

#### Dependency 참조

```
<dependency>  
   <groupId>org.springframework.cloud</groupId>  
   <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>  
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

#### application.yml 설정

```yml
resilience4j:  
  circuitbreaker:  
    #circuit-breaker-aspect-order: 1  resilience4j 기능에 대한 우선 순위, 다른 기능을 함께 사용할 것이 아니면 사용하지 않아도됨.  
    configs:  
      default:  
        registerHealthIndicator: true # actuator 정보 노출을 위한 설정  
        slowCallRateThreshold: 80
        slowCallDurationThreshold: 60s
        slidingWindowType: COUNT_BASED  
        slidingWindowSize: 10   
        permittedNumberOfCallsInHalfOpenState: 5  
        waitDurationInOpenState: 10s
        failureRateThreshold: 50
        minimumNumberOfCalls: 10

management:  
  endpoints:  
    web:  
      exposure:  
        include:  
          - "*" # 테스트를 위해 actuator 전체 노출  
  endpoint:  
    health:  
      show-details: always  
  health:  
    circuitbreakers:  
      enabled: true # circuitbreakers 정보 노출

```


| properties                            | 설명                                                                 |
| ------------------------------------- | ------------------------------------------------------------------ |
| slidingWindowType                     | 개수 기반 슬라이딩 윈도우(COUNT_BASED) / 시간 기반 슬라이딩 윈도우(TIME_BASED)           |
| slidingWindowSize<br>                 | 슬라이딩 윈도우의 사이즈                                                      |
| slowCallDurationThreshold             | Slow Call로 인식할 시간                                                  |
| slowCallRateThreshold                 | Slow Call 발생에 대한 임계값 해당 이 값을 초과하면 circuitbreaker가 OPEN으로 전환        |
| permittedNumberOfCallsInHalfOpenState | circuit이 HALF_OPEN 상태일 때 허용되는 call 수이며 실패율에 따라서 CLOSE또는 OPEN으로 변경. |
| waitDurationInOpenState               | OPEN 상태를 유지하는 시간, 해당 시간이후 HALF OPEN 상태로 변경                         |
| failureRateThreshold                  | 실패한 호출에 대한 임계값(백분율)으로 이 값을 초과하면 circuit이 OPEN 상태로 전환               |
| minimumNumberOfCalls                  | circuit을 동작시키기 위한 최소한의 call 수                                      |

> - 개수 기반 슬라이딩 윈도우
    circuitbreaker 에서 사용되는 메트릭 수집 방법 중 하나로 일정 개수(slidingWindowSize)의 요청을 추적하고, 해당 요청들 중 실패한 요청의 비율을 계산하여 임계값과 비교하여 회로 차단 여부를 결정한다.

> ![](https://velog.velcdn.com/images/akfls221/post/9db6474a-606b-4172-8bc8-d8b2df75cbb2/image.png)

> - 시간 기반 슬라이딩 윈도우
    시간을 슬라이딩 윈도우로 사용하고 일정 시간동안의 실패율을 계산하여 회로 차단 여부를 결정한다.

#### 송신 FeignClient 설정

```java
@Component  
@FeignClient(url = "${interface.order-service.url}", name = "externalClient")  
public interface ExternalClient {  
  
   @GetMapping(value ="/getOrderData/{orderId}",  produces = { MediaType.APPLICATION_JSON_VALUE})  
   public ExternalApiOrderResponseDTO getCircuitOrderData(@PathVariable String orderId);  
}
```

> FeignClient를 사용하여 외부 서비스를 호출하도록 설정한다. 

#### Service 설정 

```java
@Component  
@Slf4j  
@RequiredArgsConstructor  
public class ExternalClientWrapperService {  
  
   private final ExternalClient externalClient;  
  
  
   @CircuitBreaker(name = "getCircuitOrderData", fallbackMethod = "fallbackOrderData")  
   public ExternalApiOrderResponseDTO getCircuitOrderData(String orderId) {  
      return externalClient.getCircuitOrderData(orderId);  
   }  
  
   private ExternalApiOrderResponseDTO fallbackOrderData(String orderId, Throwable e) {  
      log.info("===== fallback Throwable ==== ");  
      ExternalApiOrderResponseDTO.builder().orderId(orderId).build();  
      return ExternalApiOrderResponseDTO.builder().build();  
   }  
  
   private ExternalApiOrderResponseDTO fallbackOrderData(String orderId, CallNotPermittedException e) {  
      log.info("===== fallback CallNotPermittedException ==== ");  
      ExternalApiOrderResponseDTO.builder().orderId(orderId).build();  
      return ExternalApiOrderResponseDTO.builder().build();  
   }  
}
```

> 외부 서비스를 호출하는 FeignClient 를 감싼 service를 만들어 @CircuitBreaker 어노테이션을 통해 circuitbreaker 설정
> 
> fallbackMethod는 외부 서비스 호출에 실패할 경우 호출되며 예외의 종류에 따라 별도의 메소드가 호출되도록 구성할 수 있다. 반환형은 @CircuitBreaker를 선언한 메소드와 일치해야하며 매개변수는 기존 메소드의 매개변수에 발생되는 예외에 대한 변수를 추가로 선언하면 된다.
> 
> CallNotPermittedException은 circuitbreaker가 OPEN 상태로 전환될 경우 외부 서비스를 호출하지 않고 해당 Exception을 매개변수로 설정한 메소드가 실행된다.

#### 수신 RestController

```java
@Slf4j  
@RestController  
@RequestMapping("/api/order")  
public class OrderController {  
  
   @GetMapping("/getOrderData/{orderId}")  
   public ExternalApiOrderResponseDTO getOrderData(@PathVariable String orderId){  
      Random r1 = new Random();  
      int randomNumber = r1.nextInt(10) + 1;  
      log.info("get Data random data = " + randomNumber);  
      if(randomNumber >= 5) {  
         throw new RuntimeException();  
      }  
  
      return ExternalApiOrderResponseDTO.builder()  
            .orderId(orderId)  
            .qty(randomNumber)  
            .itemId(String.format("ITEM%d", randomNumber)).build();  
   }  
}
```

> 외부 서비스의 컨트롤러에 랜덤 변수로 가변적인 RuntimeException을 발생시킨다.

#### Circuitbreaker 동작 확인

![Pasted image 20240406225743](https://github.com/inturn86/inturn86.github.io/assets/110794550/9c705598-0108-4bd8-aca1-12eb5b81066d)

```
CircuitBreaker 'getCircuitOrderData' is OPEN and does not permit further calls
```

> 외부 서비스를 호출하여 임계값을 초과한 경우 위와같이 circuitbreaker가 OPEN 되어 해당 call을 허가 할 수 없다는 log를 확인할 수 있다.

#### Spring Actuator Endpoint를 통한 Circuitbreaker 모니터링

> Postman을 사용하여 현재 circuitbreaker가 어떻게 동작하는지 확인해보자.
> 컨텍스트 패스에 **'/actuator/health/circuitBreakers'** 를 추가하고 GET 방식으로 호출해보자.  

##### CLOSED 상태

![Pasted image 20240406230120](https://github.com/inturn86/inturn86.github.io/assets/110794550/a490ec91-20f5-4280-be62-32cc8e5b1d7e)
> circuitbreaker의 상태이다. bufferedCalls는 현재 call 된 숫자이고, 그 중 failedCalls는 실패한 숫자를 가르킨다.
> circuitbreaker를 동작시킬 최소한의 call수(minimumNumberOfCalls)를 넘지않으면 failureRate는 측정되지 않는다. 

##### OPEN 상태

![Pasted image 20240406230140](https://github.com/inturn86/inturn86.github.io/assets/110794550/087be2de-d04a-4c6f-bd36-83329900c92b)
> 10회 실행 중 5회 실패에 따라 임계값 50%보다 커지게 되어 circuitbreaker가 OPEN 상태로 전환되었다. 
> circuitbreaker가 OPEN 되어 waitDurationInOpenState에 설정한 10초의 시간동안 외부 서비스를 call하지 않고 빠른 실패로 반환처리 한다. 

##### HALF_OPEN 상태
![Pasted image 20240406230149](https://github.com/inturn86/inturn86.github.io/assets/110794550/322d7341-f581-4290-95da-42f34069cc92)

> HALF_OPEN 상태일 때 다시 외부 서비스를 call한다.
> permittedNumberOfCallsInHalfOpenState에 설정한 call의 수 만큼 호출한 후 실패율이 임계값보다 클 경우 OPEN 상태, 반대일 경우 CLOSED 상태로 전환한다.


#### FallbackMethod

```
2024-04-06T23:17:29.930+09:00  INFO 59816 --- [   scheduling-1] c.s.r.f.ExternalClientWrapperService     : ===== fallback Throwable ==== 
2024-04-06T23:17:32.940+09:00  INFO 59816 --- [   scheduling-1] c.s.r.f.ExternalClientWrapperService     : ===== fallback CallNotPermittedException ==== 
```

> 외부 서비스 호출에 실패할 경우와 circuitbreaker에서 차단시키는 경우에 대한 예외를 구분할 수 있다.
> 이를 통하여 circuitbreaker의 장애 전파를 막음과 동시에 장애가 발생한 요청을 후처리 할 수 있다.

### 마치며

오늘은 resilience4j의 circuitbreaker를 활용하여 장애 전파를 막고 장애가 발생한 요청에 대한 후처리를 할 수 있는 기능까지 알아보았다. resilience4j는 MSA에 많이 활용되는 기술이지만 외부 서비스와 인터페이스가 빈번한 서비스에도 활용하기 좋은 기술이라고 생각한다.

### Github
- https://github.com/inturn86/msa/tree/aa13eefcf695b476214d870b6a9697f4a80302c2/resilience-project


### 참고 자료
-  https://resilience4j.readme.io/