---
title: (Java) 비동기 작업 결과 반환 - CompletableFuture
excerpt: 비동기 작업을 통한 작업 결과 반환에 대해 알아보자. 비동기 처리를 통한 성능 향상 방법은 무엇이 있을까?
date: 2024-04-13 13:55:00 +0800
description: 비동기 작업을 통한 작업 결과 반환에 대해 알아보자. 비동기 처리를 통한 성능 향상 방법은 무엇이 있을까?
categories:
  - Java
tags:
  - Java
  - CompletableFuture
  - Future
  - CompletionStage
  - 비동기
  - 비동기콜백
  - 성능향상
---

개발을 진행하면서 비동기로 작업을 진행하고 작업 결과를 반환받아 처리해야 하는 상황이 생겨 비동기 작업을 처리하는 기술과 해당 기술을 선정하게 된 이유, 사용 방법에 대해 알아보도록 하자.

### 개발 목표

1. 주문을 생성한다.
2. 생성된 주문을 외부 서비스로 전송하여 확정 여부를 결정한다.
3. 확정된 주문은 사용자에게 푸시 메세지로 전송한다.
4. 주문 생성을 제외한 확정 / 푸시 메세지는 비동기 처리로 성능을 향상시킨다.

위 내용과 같은 개발을 진행할 때 생성된 주문에 대한 2, 3의 처리를 순차적으로 실행하는 시간은 비동기로 실행하는 경우에 비해 당연히 느릴 것이다. 그렇다면 실행시간을 위해 비동기로 처리할 어떠한 기술을 사용해야 할까?

처음 고려한 기술은 **Future**이다. **Future**는 Java5 부터 사용되던 인터페이스로 비동기 작업의 결과 값을 받는 용도로 사용했다. 하지만 **한계점이 존재**하였다.

#### Future의 한계점
- **블로킹 코드**(get)을 통해서만 이후의 결과를 처리 할 수 있음
- 외부에서 완료시킬 수 없고, get의 타임아웃 설정으로만 완료 가능
- 여러 Future를 조합할 수 없음
- 예외를 처리할 수 없음

get을 이용하여 비동기 작업 응답을 받는데, 작업이 완료될 때까지 대기하는 블로킹 호출이므로 비동기 작업 응답에 추가 작업을 하기 적합하지 않다.

그래서 Future를 보완하여 Java8부터 사용 가능한 CompletableFutrue를 사용하기로 결정했다. 

CompletableFutrue는 Future와 CompletionStage 인터페이스를 구현하고 있다. CompletionStage는 여러 연산을 결합할 수 있고 연산이 완료되면 다음 단계의 작업을 수행하거나 값을 연산하는 비동기식 연산 단계를 제공하는 인터페이스이다.

### CompletableFuture 

CompletableFuture가 갖는 작업의 종류는 크게 다음과 같이 구분할 수 있다.

1. 비동기 작업 실행
2. 작업 콜백
3. 작업 조합
4. 예외 처리

#### 1. 비동기 작업 실행
##### runAsync(Runnable runnable)
- 반환값이 없는 경우

```java
@DisplayName("CompletableFuture 주문 생성")  
@Test  
void completableFuture_OrderCreate() throws ExecutionException, InterruptedException {  
    
   final int orderCnt = 100;
   //주문 목록 생성  
   CompletableFuture.runAsync(() -> createOrderList(orderCnt));  
   CompletableFuture<Void> future = CompletableFuture.runAsync(() -> createOrderList(orderCnt));  
  
   //CompletableFuture는 ForkJoinPool.CommonPool()를 사용하는데 여기서 생성된 Thread는 데몬스레드이기에 확인을 위한 get처리
   future.get();  
}

//주문 목록 생성
private List<OrderDTO> createOrderList(int orderCnt) {  
   List<OrderDTO> orderList = new ArrayList<>();  
   for (int i = 0; i < orderCnt; i++) {  
      String orderId = String.format("OD-%d", i);  
      orderList.add(new OrderDTO(orderId, "READY"  
            , String.format("ITEM-%s", i), String.format("addr %s", i),1));  
      log(String.format("Create Order = %s", orderId));  
   }  
   return orderList;  
}

private void log(String comment) {  
   System.out.println(String.format("[%s - %s] %s", LocalDateTime.now(), Thread.currentThread().getName(), comment));  
}
```

**실행결과**
```
[2024-04-13T13:46:37.781139900 - ForkJoinPool.commonPool-worker-1] Create Order = OD-54
[2024-04-13T13:46:37.781139900 - ForkJoinPool.commonPool-worker-2] Create Order = OD-32
[2024-04-13T13:46:37.781139900 - ForkJoinPool.commonPool-worker-1] Create Order = OD-55
[2024-04-13T13:46:37.781139900 - ForkJoinPool.commonPool-worker-2] Create Order = OD-33
```

runAsync를 사용하여 주문을 생성하는 메소드를 비동기로 처리하였다. 위 실행결과를 보면 별도의 Thread로 처리된 것을 확인할 수 있다.

Test 코드에서 future.get()을 사용하였다. CompletableFutrue에서도 마찬가지로 get은 결과가 반환되기까지 블로킹 된다. 여기서 블로킹한 이유는 CompletableFuture는 ForkJoinPool.CommonPool()을 사용하여 스레드를 할당하는데 여기서 생성된 스레드는 데몬스레드이다. 따라서 JVM은 해당 스레드가 완료될 때까지 기다리지 않는다. 그렇기에 메소드가 모두 실행되게 하려면 get을 사용하여 블로킹해야 확인할 수 있다.

##### supplyAsync(Supplier supplier)
-  반환값이 있는 경우

supplyAsync는 작업 콜백에서 확인하도록 하자.

#### 2. 작업 콜백

##### thenAccept(Consumer\<? super T\> action)
- Comsumer를 인자로 받고 CompletableFuture\<Void\>를 반환  
- 작업 콜백 처리 후 반환 값이 없는 로직을 처리할 때 사용할 수 있다.  

```java
@DisplayName("CompletableFuture 주문 확정")  
@Test  
void completableFuture_OrderConfirm() throws ExecutionException, InterruptedException {  
  
   final int orderCnt = 100;
   
   List<String> confirmOrderList = new ArrayList<>();  
   //주문 목록을 전송하고 확정 처리  
   for(OrderDTO order : createOrderList(orderCnt)) {  
      CompletableFuture.supplyAsync(() -> sendOrder(order))  
            .thenAccept(res -> simpleConfirmOrder(confirmOrderList, res.getOrderId()));  
   }  
  
   //데몬스레드 종료를 위한 확정 완료 비교
   CompletableFuture<Integer> block = CompletableFuture.supplyAsync(() -> finishConfirmOrder(orderCnt, confirmOrderList));  
   log(String.valueOf(block.get()));  
}

//주문 전송 및 성공 반환
private OrderResponse sendOrder(OrderDTO order) {  
   boolean success = true;  
   log(String.format("Send Order : OrderId = %s", order.orderId()));
   return OrderResponse.builder().success(success).orderId(success ? order.orderId() : null).build();  
}

//주문 확정 처리
private String simpleConfirmOrder(List<String> confirmOrderList, String orderId) {  
   log(String.format("Confirm Complele Order : OrderId = %s", orderId));  
   confirmOrderList.add(orderId);  
   return orderId;  
}  

//데몬스레드 종료를 위한 확정 완료 비교 
private Integer finishConfirmOrder(int orderCnt, List<?> completeOrderList) {  
   while(orderCnt != completeOrderList.size()){  
      try {  
         Thread.sleep(10);  
      } catch (InterruptedException e) {  
         throw new RuntimeException(e);  
      }  
   }  
   return completeOrderList.size();  
}
```

**실행결과**

```
[2024-04-13T14:16:21.027928700 - ForkJoinPool.commonPool-worker-3] Send Order : OrderId = OD-91
[2024-04-13T14:16:21.028428300 - ForkJoinPool.commonPool-worker-3] Confirm Complele Order : OrderId = OD-91
[2024-04-13T14:16:21.027527500 - ForkJoinPool.commonPool-worker-1] Confirm Complele Order : OrderId = OD-86
[2024-04-13T14:16:21.027527500 - ForkJoinPool.commonPool-worker-2] Send Order : OrderId = OD-88
[2024-04-13T14:16:21.027161900 - ForkJoinPool.commonPool-worker-6] Confirm Complele Order : OrderId = OD-79
[2024-04-13T14:16:21.028428300 - ForkJoinPool.commonPool-worker-2] Confirm Complele Order : OrderId = OD-88
[2024-04-13T14:16:21.028428300 - ForkJoinPool.commonPool-worker-4] Send Order : OrderId = OD-97
[2024-04-13T14:16:21.028428300 - ForkJoinPool.commonPool-worker-4] Confirm Complele Order : OrderId = OD-97
[2024-04-13T14:16:21.044216900 - Test worker] 100
```

주문을 일괄 생성하고 supplyAsync를 통해 sendOrder 메소드를 비동기로 처리한다. 그리고 sendOrder의 반환값을 작업 콜백 thenAccept에서 받아 simpleConfirmOrder 메소드가 실행된다.
위 결과를 보면 별도의 스레드를 통해 처리되기 때문에 처리에 대한 순서도 순차적이지 않다.


##### thenApply(Function\<? super T,? extends U\> fn)
-  Function을 인자로 받고 Function의 반환값인 U의 CompletableFuture\<U\>를 반환  
 - 작업 콜백 처리 시 인자를 받아 반환 값이 있는 로직을 처리할 때 사용할 수 있다.  

```java
@DisplayName("CompletableFuture 주문 확정에 따른 Push Message 전송")  
@Test  
void completableFuture_OrderConfirmAndSendPushMessage() throws InterruptedException {  
  
   List<PushMessageResultDTO> failPushMessageResultDTOList = new ArrayList<>();  
   //주문 목록에 따른 confirm 처리  
   for(OrderDTO order : createOrderList(orderCnt)) {  
      CompletableFuture.supplyAsync(() -> confirmOrder(order))  
            .thenApply(orderId -> sendPushMessageTask(orderId))  
            .whenComplete((result, e) -> {  
               if(!result.getSuccess())    failPushMessageResultDTOList.add(result);  
            });  
   }  
  
   //CompleteFuture의 데몬 스레드를 유지하기 위한 테스트용 Sleep   
   Thread.sleep(10000);  
   //푸시 메세지 전송에 실패한 주문 목록.  
   failPushMessageResultDTOList.stream().forEach(o -> System.out.println(o.getOrderId()));  
}

private String confirmOrder(OrderDTO order) {  
   var res = this.sendOrder(order);  
   log(String.format("confirmOrder thread name success = %s", res.getSuccess().toString()));  
   //성공 여부에 따른 confirm 처리.  
   if(ObjectUtils.isNotEmpty(res) && StringUtils.equals(order.orderId(), res.getOrderId())) {  
      return res.getOrderId();  
   };  
   return StringUtils.EMPTY;  
}

//푸시 메세지 전송 Task
private PushMessageResultDTO sendPushMessageTask(String orderId) {  
   boolean result = false;  
   AtomicInteger retry = new AtomicInteger();  
   //푸시 메세지를 3회 전송에 대한 결과를 전송.  
   while (!result) {  
      int retryCnt = retry.getAndIncrement();  
      //푸시 메세지 전송 및 결과 반환  
      result = sendPushMessage(orderId);  
      log(String.format("sendPushMessage thread name orderId = %s, result = %b", orderId, result));  
      if(retryCnt >= 3) {  
         break;  
      }  
      try {  
         Thread.sleep(100);  
      } catch (InterruptedException e) {  
         throw new RuntimeException(e);  
      }  
   }  
   return PushMessageResultDTO.builder().orderId(orderId).success(result).build();  
}

//푸시 메세지 전송
private boolean sendPushMessage(String orderId) {  
   boolean result = RandomUtils.nextBoolean();  
   log(String.format("sendPushMessage result = %b", result));  
   return result;  
  
}
```
주문을 일괄 생성하고 supplyAsync로 confirmOrder 메소드를 비동기 처리한다. confirmOrder가 완료되어 반환되면 thenApply를 통해 주문 ID가 sendPushMessageTask로 전달되고 이때 반복문을 통해 sendPushMessage가 실행되는데 전송에 실패할 경우 최대 3회 시도 후 반환값을 whenComplete로 전달하여 비동기 처리를 마무리한다.  


##### thenRun(Runnable action)
- Runnable을 인자로 받고 CompletableFuture\<Void\>를 반환
- 작업 콜백 처리 시 인자를 받지 않고 반환 값이 없는 로직을 처리할 때 사용. 



#### 3. 작업 조합
##### thenCompose(Function\<? super T, ? extends CompletionStage\<U\>> fn)
- thenCompose 를 호출한 CompletableFuture가 비동기 처리되고 결과를 전달하여 매개변수로 받은 CompletableFuture가 비동기로 처리된다.  

```java
@DisplayName("CompletableFuture Compose 주문 생성 및 주문 확정")  
@Test  
void completableFuture_ComposeCreateOrderAndConfirmOrder() throws ExecutionException, InterruptedException {  
  
   var future = CompletableFuture.supplyAsync(() -> createOrder())  
         .thenCompose(order -> CompletableFuture.supplyAsync(() -> confirmOrder(order)));  
  
   log(future.get());  
}

private OrderDTO createOrder() {  
   String orderId = String.format("OD-%d", RandomUtils.nextInt());  
   log(String.format("Create Order = %s", orderId));  
   return new OrderDTO(orderId, "READY", "ITEM", "ADDRESS", 1);  
}
```

##### thenCombine(CompletionStage\<? extends U\> other, BiFunction\<? super T,? super U,? extends V\> fn)
- 2개의 CompletableFuture로 각각 비동기 처리되며, 모두 완료되었을 경우 BiFunction에서 결과를 조합하여 사용  

```java
@DisplayName("CompletableFuture 주문 제품 및 배송지 조회")  
@Test  
void completableFuture_CombineOrderItemAndShippingInfo() throws ExecutionException, InterruptedException {  
  
   OrderDTO order = createOrder();  
   //주문의 제품 및 배송지 조회
   var future = CompletableFuture.supplyAsync(() -> getItemInfo(order.itemId()))  
            .thenCombine(CompletableFuture.supplyAsync(() -> getShippingInfo(order.address())), (s1, s2) -> completeOrder(s1, s2));  
  
   log(future.get());  
}

private String getItemInfo(String itemId) {  
   try {  
      Thread.sleep(1000);  
   } catch (InterruptedException e) {  
      throw new RuntimeException(e);  
   }  
   log(String.format("getItemInfo = %s ", itemId));  
   return itemId;  
}  
  
private String getShippingInfo(String address) {  
   try {  
      Thread.sleep(50);  
   } catch (InterruptedException e) {  
      throw new RuntimeException(e);  
   }  
   log(String.format("getShippingInfo = %s ", address));  
   return address;  
}  
  
private String completeOrder(String itemId, String address) {  
   String completeOrder = String.format("item = %s, address = %s", itemId, address);  
   log(completeOrder);  
   return completeOrder;  
}
```

##### allOf(CompletableFuture\<?>... cfs)
- 여러개의 CompletableFuture를 비동기로 일괄 실행하고 모든 작업이 완료된 경우 결과를 일괄 반환.

```java
@DisplayName("CompletableFuture 주문 목록 일괄 확정")  
@Test  
void completableFuture_AllOfOrderConfirm() throws ExecutionException, InterruptedException {  
  
   final int confirmTime = 100;  
   List<CompletableFuture<String>> futureList = new ArrayList<>();  
     
   for(OrderDTO order : createOrderList(orderCnt)) {  
      CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> confirmOrder(order.orderId(), confirmTime));  
      futureList.add(future);  
   }  
   CompletableFuture<String>[] futureArr = new CompletableFuture[orderCnt];
   //주문의 일괄 확정 저리
   CompletableFuture<List<String>> allOfFuture = CompletableFuture.allOf(futureList.toArray(futureArr))  
         .thenApply(o -> futureList.stream().map(CompletableFuture::join).collect(Collectors.toList())  
   );
   System.out.println(allOfFuture.get());  
}

private String confirmOrder(String orderId, int confirmTime) {  
   try {  
      Thread.sleep(confirmTime);  
   } catch (InterruptedException e) {  
      throw new RuntimeException(e);  
   }  
   log(String.format("confirmOrder orderId = %s", orderId));  
   return orderId;  
}
```
  
##### anyOf(CompletableFuture\<?>... cfs)
- 여러개의 CompletableFuture를 비동기로 일괄 실행하고 가장 빨리 끝난 1개의 작업 결과를 반환.  

```java
@DisplayName("CompletableFuture 주문 목록 빠른 확정")  
@Test  
void completableFuture_AnyOfOrderConfirm() throws ExecutionException, InterruptedException {  
  
   final int confirmTime = 100;  
  
   List<CompletableFuture<String>> futureList = new ArrayList<>();  
   //주문 목록에 따른 confirm 처리  
   for(OrderDTO order : createOrderList(orderCnt)) {  
      CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> confirmOrder(order.orderId(), confirmTime));  
      futureList.add(future);  
   }  
   CompletableFuture<String>[] futureArr = new CompletableFuture[orderCnt];  
   CompletableFuture<Void> anyOfFuture = CompletableFuture.anyOf(futureList.toArray(futureArr))  
         .thenAccept(o -> System.out.println(o));  
   anyOfFuture.get();  
}
```

#### 4. 예외 처리

##### handle(BiFunction\<? super T, Throwable, ? extends U> fn)
- 결과, 에러를 받아 에러가 발생할 경우와 아닌 경우에 대한 처리 가능  
- 함수형 인터페이스 BiFunction을 매개변수로 받음

```java
@DisplayName("CompletableFuture 에러 헨들링")  
@ParameterizedTest  
@ValueSource(booleans = {true, false})  
void completableFuture_ErrorHandle(boolean param) throws ExecutionException, InterruptedException{  
  
   var future = CompletableFuture.supplyAsync(() -> createOrder()).thenApply(order -> { 
      if(param) {  
         throw new IllegalArgumentException("throw Illegal Exception");  
      }  
      return order.orderId();  
   }).handle((result, e) -> e == null ? result : e.toString());  
   log(future.get());  
}
```

**실행결과(에러)** 
```
[2024-04-13T15:19:35.475793200 - ForkJoinPool.commonPool-worker-1] Create Order = OD-265027152
[2024-04-13T15:19:35.475793200 - Test worker] java.util.concurrent.CompletionException: java.lang.IllegalArgumentException: throw Illegal Exception

```

**실행결과(성공)** 
```
[2024-04-13T15:19:35.475793200 - ForkJoinPool.commonPool-worker-1] Create Order = OD-147284801
[2024-04-13T15:19:35.475793200 - Test worker] OD-147284801
```

##### exceptionally(Function<Throwable, ? extends T> fn)
- 작업 처리 중 에러가 발생할 경우 예외 처리 진행  
- 함수형 인터페이스 Function을 매개변수로 받음

```java
@DisplayName("CompletableFuture 에러 예외 헨들링")  
@ParameterizedTest  
@ValueSource(booleans = {true, false})  
void completableFuture_ErrorExceptionally(boolean param) throws ExecutionException, InterruptedException{  
  
   var future = CompletableFuture.supplyAsync(() -> createOrder()).thenApply(order -> { 
      if(param) {  
         throw new IllegalArgumentException("throw Illegal Exception");  
      }  
      return order.orderId();  
   }).exceptionally(e -> e.toString());  
   log(future.get());  
}
```
#### 비동기메소드
CompletableFuture 클래스가 제공하는 메소드를 보면 async라는 단어가 붙은 메소드를 확인할 수 있다. 

> thenApply - thenApplyAsync
>
> thenAccept - thenAcceptAsync

Async가 붙은 메소드는 다른 스레드를 이용하여 비동기 작업을 처리하고, 아닌 경우는 현재 스레드로 작업을 처리한다.

```java
  
ExecutorService executorService = Executors.newCachedThreadPool();

@DisplayName("CompletableFuture Compose 주문 생성 및 주문 확정")  
@Test  
void completableFuture_ComposeCreateOrderAndConfirmOrder() throws ExecutionException, InterruptedException {  
  
   var future = CompletableFuture.supplyAsync(() -> createOrder())  
         .thenCompose(order -> CompletableFuture.supplyAsync(() -> confirmOrder(order)));  
  
   var futureThenComposeAsync = CompletableFuture.supplyAsync(() -> createOrder())  
         .thenComposeAsync(order -> CompletableFuture.supplyAsync(() -> confirmOrder(order), executorService));  
  
   log(future.get());  
   log(futureThenComposeAsync.get());  
}

```

**실행결과**
```
[2024-04-13T15:40:03.760876800 - ForkJoinPool.commonPool-worker-2] Create Order = OD-205396118
[2024-04-13T15:40:03.760876800 - ForkJoinPool.commonPool-worker-1] Create Order = OD-1721895502
[2024-04-13T15:40:03.761388200 - ForkJoinPool.commonPool-worker-3] Send Order : OrderId = OD-1721895502
[2024-04-13T15:40:03.762161500 - pool-1-thread-1] Send Order : OrderId = OD-205396118
[2024-04-13T15:40:03.762735300 - ForkJoinPool.commonPool-worker-3] confirmOrder thread name success = true
[2024-04-13T15:40:03.762735300 - pool-1-thread-1] confirmOrder thread name success = true
[2024-04-13T15:40:03.765883700 - Test worker] OD-1721895502
[2024-04-13T15:40:03.765883700 - Test worker] OD-205396118

```
위와 같이 기존 사용되던 ForkJoinPool이 아닌 다른 스레드로 실행된 것을 확인할 수 있다.


#### 처리 속도

순차 처리와 비동기 처리에 대해 처리 시간을 비교해보았다.

```java
@DisplayName("CompletableFuture 주문 확정 - Processing Time 측정")  
@Test  
void completableFutureOrderConfirm_Checklatency() throws ExecutionException, InterruptedException {  

   final int orderCnt = 100;
  
   List<String> confirmOrderList = new ArrayList<>();  
   log("START 순차 처리");  
   //주문 목록에 따른 confirm 처리  
   for(OrderDTO order : createOrderList(orderCnt)) {  
      CompletableFuture.supplyAsync(() -> sendOrderDelay(order))  
            .thenAccept(res -> simpleConfirmOrder(confirmOrderList, res.getOrderId()))  
            .get();  
   }  
   log("End 순차 처리");  
   confirmOrderList.clear();  
  
   log("START 비동기 처리");  
   for(OrderDTO order : createOrderList(orderCnt)) {  
      CompletableFuture.supplyAsync(() -> sendOrderDelay(order))  
            .thenAccept(res -> simpleConfirmOrder(confirmOrderList, res.getOrderId())); 
   }  
  
   //모든 주문이 확정될 경우 마무리.
   CompletableFuture<Integer> block = CompletableFuture.supplyAsync(() -> finishConfirmOrder(orderCnt, confirmOrderList));  
   log(String.valueOf(block.get()));  
   log("END 비동기 처리");  
}

//외부 시스템으로 주문 전송
private OrderResponse sendOrderDelay(OrderDTO order) {  
   boolean success = true;  
   log(String.format("Send Order : OrderId = %s", order.orderId()));  
   //외부 시스템의 처리시간을 sleep으로 구현  
   try {  
      Thread.sleep(50);  
   } catch (InterruptedException e) {  
      throw new RuntimeException(e);  
   }  
   return OrderResponse.builder().success(success).orderId(success ? order.orderId() : null).build();  
}
```

**실행결과**
```
[2024-04-13T15:57:44.881285500 - Test worker] START 순차 처리
[2024-04-13T15:57:51.013331200 - Test worker] End 순차 처리

[2024-04-13T15:57:51.013831200 - Test worker] START 비동기 처리
[2024-04-13T15:57:51.931857 - Test worker] END 비동기 처리
```

 - 조건 : 주문 100EA
 - 시나리오
	 - sendOrderDelay - 외부 서비스로 주문을 전송(처리시간 50 msec)
	 - confirmOrder - sendOrderDelay 의 반환값으로 주문 확정 처리.

- 순차 처리 : 6,132 msec
- 비동기 처리 : 918 msec

### 마무리
개발 목표를 위한 사용 가능한 기술을 알아보고 선정하는 시간을 가졌다. CompletableFuture를 사용한 비동기 작업 결과 반환 테스트 및 주요 기능에 대해서 알아보았는데 비동기 처리에 대한 개념과 활용 방안에 대해 자세히 알게되는 좋은 계기가 되었다.
부족한 부분이 있다면 아래 코멘트를 남겨주시면 감사하겠습니다.  

### Github
<https://github.com/inturn86/async-completable>

### **참고자료**
[Java CompletableFuture로 비동기 적용하기 | 11번가 TechBlog — 11번가 기술블로그 (11st-tech.github.io)](https://11st-tech.github.io/2024/01/04/completablefuture/#CompletableFuture)

<https://www.baeldung.com/java-completablefuture>

[https://mangkyu.tistory.com/263](https://mangkyu.tistory.com/263) [MangKyu's Diary:티스토리]