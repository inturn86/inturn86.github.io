---
title: (QueryDsl) 왜 QueryDsl로 Sql 함수가 안되지...
excerpt: QueryDsl에서 Sql 함수 에러 발생? JPA 방언으로 문제 해결하는 방법 ? 함수 에러에 대한 원인 분석 및 해결 방안 제시
date: 2024-03-28 21:55:00 +0800
categories:
  - JPA
  - QueryDsl
tags:
  - JPA
  - QueryDsl
  - Dialect
  - Postgresql
  - DATE
  - to_char
  - ANSI
  - Custom
  - PostgreSQL92Dialect
  - Expressions
  - stringTemplate
pin: false
img_path:
---

QueryDsl의 Expressions.stringTemplate으로 Sql 함수를 사용하면서 발생했던 문제를 확인해보자. 

## 문제 상황

주문의 처리 수량을 summary하는 쿼리를 QueryDsl로 개발했다. 
LocalDateTime의 데이터를 DateTime에서 Date로 변경하여 group by 해야해서  Expressions.stringTemplate으로 Sql 함수를 사용했다. 하지만 아래와 같은 에러가 발생한다.

> DATE(Timestamp)
> 
> 이 함수는 'Timestamp' 값을 입력받아 해당 날짜 부분만을 추출하여 'Date' 타입의 값으로 반환한다. 표준 SQL 함수로서, 다양한 DBMS에서 날짜와 시간 값을 다루는데 사용하는 함수이다.

```
org.hibernate.QueryException: No data type for node: org.hibernate.hql.internal.ast.tree.MethodNode 
 +-[METHOD_CALL] MethodNode: '('
 |  +-[METHOD_NAME] IdentNode: 'DATE' {originalText=DATE}
 |  \-[EXPR_LIST] SqlNode: 'exprList'
 |     \-[DOT] DotNode: 'orderentit0_.order_finish_dt' {propertyName=orderFinishDt,dereferenceType=PRIMITIVE,getPropertyPath=orderFinishDt,path=orderEntity.orderFinishDt,tableAlias=orderentit0_,className=com.sdc.solutions.service.wom.order.OrderEntity,classAlias=orderEntity}
 |        +-[ALIAS_REF] IdentNode: 'orderentit0_.order_id' {alias=orderEntity, className=com.sdc.solutions.service.wom.order.OrderEntity, tableAlias=orderentit0_}
 |        \-[IDENT] IdentNode: 'orderFinishDt' {originalText=orderFinishDt}
 [select DATE(orderEntity.orderFinishDt) as orderFinishDay, sum(orderEntity.execQty) as execSumQty
from com.sdc.solutions.service.wom.order.OrderEntity orderEntity
group by DATE(orderEntity.orderFinishDt)
order by DATE(orderEntity.orderFinishDt) asc]; nested exception is java.lang.IllegalArgumentException: org.hibernate.QueryException: No data type for node: org.hibernate.hql.internal.ast.tree.MethodNode 
```

**No data type for node. 'DATE' 함수을 사용한 node의 Data type이 없다는 것**으로 확인된다. 다음은 QueryDsl 구현부와 실제 수행된 쿼리이다.

```java
@Override  
public List<OrderSummaryByDayDTO> getSummaryExecQtyByDay() {  
   var orderFinishDay = Expressions.stringTemplate("DATE({0})", qOrder.orderFinishDt);  
   return from(qOrder)  
         .select(Projections.bean(OrderSummaryByDayDTO.class,  
               orderFinishDay.as("orderFinishDay")  
               , qOrder.execQty.sum().as("execSumQty")  
         ))  
         .groupBy(orderFinishDay)  
         .fetch();  
}
```

```sql
select 
	DATE(orderentit0_.order_finish_dt) as col_0_0_
	, sum(orderentit0_.exec_qty) as col_1_0_
from wom_order orderentit0_ 
	group by DATE(orderentit0_.order_finish_dt)
```

해당 쿼리를 DB에서 실행하면 아주 정상적으로 실행된다.
**함수를 사용했을 때 QueryDsl에서 이러한 문제가 발생하는 이유를 하나씩 살펴보자**.



## 문제 상황의 구성 정보 - DB, 테이블 및 Entity

- DB 정보
> PostgreSQL Server 9.4

- 테이블 erd 정보
  
![Pasted image 20240327231302](https://github.com/inturn86/inturn86.github.io/assets/110794550/f869d62f-4224-4f7f-9518-4fe8d79b8b7f)

- 간단한 단일 품목 주문 테이블이다. 해당 테이블에서 **주문 완료 일시(order_finish_dt)를 기준으로 일자별 실행수량(exec_qty)의 sum을 계산하는 것이 쿼리의 목적**이다.


- Entity 클래스

```
@Getter @Setter  
@Entity  
@Table(name = "wom_order")
public class Order {  
   // 주문ID.  
   @Id @DTOKey("WO")  
   protected String orderId;  
   
   // 품목ID.  
   @Column(nullable = false)  
   protected String itemId;  
   
   // 주문수량.  
   @Column(nullable = false)  
   protected Integer orderQty;  
  
   // 처리수량.  
   @Column(nullable = false)  
   protected Integer execQty;  
  
   // 주문 완료일시.  
   @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")  
   @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")  
   @Column(nullable = false)  
   protected LocalDateTime orderFinishDt;  
  
   // 등록ID  
   @Column(nullable = false, updatable = false)  
   protected String regId;  
  
   // 등록일시  
   @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")  
   @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")  
   @Column(nullable = false, updatable = false)  
   protected LocalDateTime regDt;  
  
}
```

- 반환 DTO
```
@Getter  
@Setter  
public class OrderSummaryByDayDTO  {  
  
   private Integer execSumQty;  
  
   private String orderFinishDay;  
}
```

자 이제 모든 정보가 공개되었다. 문제 해결을 위해 하나씩 개선해보자.

## JPA 데이터베이스 방언(Dialect)

위에서 설명한 것처럼 쿼리는 정상동작한다. 그렇다면 **No data type for node**는 어떤 이유로 발생했는지 살펴봐야한다.

JPA의 동작 원리를 생각해보자 데이터베이스를 접근 및 관리하는 경우 애플리케이션이 직접 JDBC 레벨에서 SQL을 작성하는 것이 아니라 **JPA가 직접 SQL을 작성하고 실행하는 원리**이다.
그렇다면 다양한 RDBMS에 대해 **어떤식으로 JPA가 SQL을 작성할 수 있을까?** 
ANSI SQL을 표준으로 하고 RDMBS 별로 자신만의 독자적인 기능을 가지는 방언(Dialect) 을 제공하여 처리한다.

> ANSI SQL
> 
> ANSI, American National Standards Institute (미국 표준 협회)가 각기 다른 DBMS에서 공통적으로 사용할 수 있도록 고안한 표준 SQL 문서 작성방법.

JPA 구현체들은 이런 문제를 해결하기 위해 다양한 데이터베이스 방언 클래스를 제공한다. 따라서 스프링 JPA가 제공하는 표준 문법에 맞추어 개발하면 특정 데이터베이스에 의존적인 SQL은 설정한 데이터베이스 방언이 처리해줍니다.

**방언 클래스를 사용하므로써, 개발자는 데이터베이스에 의존적인 개발을 지양할 수 있으며 데이터베이스 변경에도 유연하게 대처할 수 있다**.

application.yml에 아래와 같이 설정하면 PostgreSQL의 방언이 PostgreSQL92Dialect 클래스로 설정된다.

```
spring:  
  jpa:  
    properties:
      hibernate:  
	    dialect: org.hibernate.dialect.PostgreSQL92Dialect
```

그렇다면 우리가 사용한 DATE라는 함수가 위 방언에 등록되어 있는지 확인해보자. 위 방언 클래스를 상속하고 있는 Dialect 클래스에 sqlFunctions라는 HashMap에 사용할 수 있는 함수가 저장되어 있다.

![Pasted image 20240328223539](https://github.com/inturn86/inturn86.github.io/assets/110794550/13f673f9-4be9-4082-9097-7836cba762d6)

그리고 자세히 내용을 보면 HashMap의 value에 registeredType이라는 반환값이 설정되어있다. 

![Pasted image 20240328223642](https://github.com/inturn86/inturn86.github.io/assets/110794550/3f4db5b4-0370-4e62-9453-13835575b803)

'to_char'는 StringType의 반환값을 가지고 있다. 해당 내용을 찾아보면 'to_char'는 PostgreSQL81Dialect 클래스의 생성자에서 registerFunction 메소드로 등록한다.
![Pasted image 20240328224052](https://github.com/inturn86/inturn86.github.io/assets/110794550/d5888c7a-0b84-41ee-a7be-9869b6503ecf)

'DATE' 는 등록되어 있지 않기 때문에 위와 같은 문제가 발생했다. 그럼 'to_char'로 변경하여 summary 쿼리를 조회해보자.

> to_char(Timestamp, '날짜포맷')
> 
> 날짜타입의 데이터를 '날짜포맷'에 따라 문자열로 변환

- QueryDsl 구현부
```java
@Override  
public List<OrderSummaryByDayDTO> getSummaryExecQtyByDay() {  
   var orderFinishDay = Expressions.stringTemplate("DATE({0})", qOrder.orderFinishDt);  
   return from(qOrder)  
         .select(Projections.bean(OrderSummaryByDayDTO.class,  
               //orderFinishDay.as("orderFinishDay")  
               Expressions.stringTemplate("to_char({0}, 'YYYY-MM-DD')", qOrder.orderFinishDt.min()).as("orderFinishDay")  
               , qOrder.execQty.sum().as("execSumQty")  
         ))  
         .groupBy(orderFinishDay)  
         .fetch();  
}
```

- SQL
```sql
select 
	to_char(min(orderentit0_.order_finish_dt), 'YYYY-MM-DD') as col_0_0_
	, sum(orderentit0_.exec_qty) as col_1_0_ 
from wom_order orderentit0_ 
	group by DATE(orderentit0_.order_finish_dt)
```

- 결과


  ![Pasted image 20240328230928](https://github.com/inturn86/inturn86.github.io/assets/110794550/9b98ccfc-2a12-4b6a-8a45-d5fca8f9f4cc)

'to_char' 로 변환 시 정상동작한다. 하지만 위 QueryDsl 구현부를 보면 group by는 그대로 'DATE'를 사용하고 있다. 그럼에도 불구하고 정상동작하는 이유는 뭘까? 어떻게 동작하는지 추가적으로 확인해보자.

## org.hibernate.QueryException: No data type for node

해당 QueryException이 발생한 곳을 찾아보자.

![Pasted image 20240328232438](https://github.com/inturn86/inturn86.github.io/assets/110794550/10f39b76-52fe-4cfc-a1bd-68c570c9c120)

SelectClause 클래스의 initializeExplicitSelectClause 메소드에서 해당 QueryException이 발생했다.

![Pasted image 20240328232613](https://github.com/inturn86/inturn86.github.io/assets/110794550/5c25aa63-3ffb-4ec1-9b19-13f47fe09364)

위 이미지에서 보는바와 같이 'DATE' 라는 methodName을 사용하는데 반환하는 dataType은 null이다. 

![Pasted image 20240328232752](https://github.com/inturn86/inturn86.github.io/assets/110794550/bb19344c-8c4d-4df8-ad44-8a03902fdf95)

그래서 QueryException이 발생하는 것이다. 즉 해당 함수는 반환하는 값이 없기 때문에 Select 절에서 에러가 발생하는 것이다. 따라서 GROUP BY 절에 있는 함수는 데이터베이스 서버에서 처리되기 때문에 방언에 등록할 필요 없이 처리된다. 하지만 SELECT 절에서 사용되는 함수는 방언에 등록하여야 해당 함수에 대한 dataType을 인식하고 정상적으로 해당 타입으로 반환 받을 수 있다.

## DIalect Custom(방언 커스텀)

그렇다면 'DATE' 함수를 사용할 수 있도록 방언을 커스텀 해보자.

![Pasted image 20240328233818](https://github.com/inturn86/inturn86.github.io/assets/110794550/be0062b8-da9f-4395-8e3a-340e2aeb987c)

1. 현재 사용중인 방언을 상속받는 커스텀 방언 클래스를 생성한다.
2. 커스텀 방언 클래스의 기본 생성자를 만들고 registerFunction으로 사용할 함수를 등록한다.
3. StandardSQLFunction 생성자에 사용할 함수명과 반환타입을 명시한다.
4. 생성한 커스텀 방언을 application.yml에 설정한다. 

![Pasted image 20240328234124](https://github.com/inturn86/inturn86.github.io/assets/110794550/b3c49179-b0d7-4cf3-884d-447f91082119)

```
spring:  
  jpa:  
    properties:
      hibernate:  
	    dialect: com.sdc.supports.PostgreSQL92DialectCustom
```

- 최종 QueryDsl 구현부
```
@Override  
public List<OrderSummaryByDayDTO> getSummaryExecQtyByDay() {  
   var orderFinishDay = Expressions.stringTemplate("DATE({0})", qOrder.orderFinishDt);  
   return from(qOrder)  
         .select(Projections.bean(OrderSummaryByDayDTO.class,  
               orderFinishDay.as("orderFinishDay")  
               , qOrder.execQty.sum().as("execSumQty")  
         ))  
         .groupBy(orderFinishDay)  
         .fetch();  
}
```

이렇게 QueryDsl 에서 Expredssions.stringTemplate으로 Sql 함수를 사용할 때 발생하는 문제의 원인과 해결 방안에 대해서 알아보았다. RDBMS 마다 방언에 차이가 있고 또한 함수에도 차이가 있음을 알게 되었다. 방언은 기본적으로 표준 SQL인 ANSI SQL을 사용하기에 우선 표준 SQL에서 사용 가능한 함수를 찾고 없을 경우 각 RDBMS 고유의 함수를 사용하도록 하자. **방언에 함수가 포함되어있지 않을 경우는 방언을 커스텀하여 사용할 함수를 등록하여 문제를 해결**하자.


>참고 자료
>
>[[Spring JPA ] 데이터베이스 방언(Dialect) 이란? (velog.io)](https://velog.io/@choidongkuen/Spring-JPA-%EB%8D%B0%EC%9D%B4%ED%84%B0%EB%B2%A0%EC%9D%B4%EC%8A%A4-%EB%B0%A9%EC%96%B8Dialect-%EC%9D%B4%EB%9E%80)
> 
>[(MySQL)JPA-QueryDSL만 사용해서 인근 위치 데이터 검색해보기 (velog.io)](https://velog.io/@sdsd0908/MySQLJPA-QueryDSL%EB%A1%9C-%EC%9D%B8%EA%B7%BC-%EC%9C%84%EC%B9%98-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EA%B2%80%EC%83%89%ED%95%B4%EB%B3%B4%EA%B8%B0)
