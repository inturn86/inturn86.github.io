---
title: (QueryDsl) QuerydslRepositorySupport를 확장하여 개발 생산성을 향상시키자.
excerpt: QuerydslRepositorySupport를 확장하여 생산성 향상 / 코드 재사용에 대한 효율을 향상시켜보자.
date: 2024-05-05 12:00:00 +0800
description: QuerydslRepositorySupport를 확장하여 생산성 향상 / 코드 재사용에 대한 효율을 향상시켜보자.
categories:
  - JPA
  - QueryDsl
tags:
  - JPA
  - QueryDsl
  - BooleanExpression
  - SimpleExpression
  - OrderSpecifier
  - SOrt
  - Pageable
  - QuerydslRepositorySupport
  - CustomQuerydslRepositorySupport
---

QueryDsl을 편하게 사용하기 위해 구현된 `QuerydslRepositorySupport` 를 확장하여 코드의 재사용성을 용이하게 하고 생산성을 향상 시킬수 있는 커스텀 `QuerydslRepositorySupport`를 개발해 보자.

## **QuerydslRepositorySupport**

```java
@Repository  
public abstract class QuerydslRepositorySupport
```

Querydsl 라이브러리를 사용하여 Implement Repository를 구현하기 위한 추상클래스

```
public class CategoryRepositoryDslImpl extends QuerydslRepositorySupport implements CategoryRepositoryDsl{  
  
   QCategory qCategory = QCategory.category;  
  
   public CategoryRepositoryDslImpl() {  
      super(QCategory.class);  
   }
}
```

구현 클래스에 `QuerydslRepositorySupport` 를 상속하여 사용한다.

## **QuerydslRepositorySupport 확장 포인트**

1. 조건문의 추상화 
	- eq, like 등
	- null일 수 있는 조건에 대한 예외 처리
2. 동적 Sort 기능

위 확장 포인트를 구현한 클래스의 내용은 아래와 같습니다.

```java
@Slf4j  
public abstract class PfitQuerydslRepositorySupport<C extends EntityPathBase> extends QuerydslRepositorySupport {  
  
   public PfitQuerydslRepositorySupport(Class<?> domainClass) {  
      super(domainClass);  
   }  
  
   public <T extends SimpleExpression> BooleanExpression eq(T path, Object param){  
      return ObjectUtils.isEmpty(param) ? null : path.eq(param);  
   }  
  
   public BooleanExpression like(StringPath path, String param){  
      return ObjectUtils.isEmpty(param) ? null : path.like(param);  
   }  
  
   public <T> Page<T> pagingList(JPQLQuery<T> query, Pageable pageable) {  
      return new PageImpl<>(query.fetch(), pageable, query.fetchCount());  
   }  
  
   public OrderSpecifier[] getOrderSpecifiers(C clazz, Sort sort) {  
      if(ObjectUtils.isEmpty(sort))   return new OrderSpecifier[]{};  
      List<OrderSpecifier> orders = new ArrayList<>();  
      PathBuilder orderByExpression = new PathBuilder(clazz.getClass(), clazz.toString(), PathBuilderValidator.FIELDS);  
      sort.stream().forEach(order -> {  
         Order direction = order.isAscending() ? Order.ASC : Order.DESC;  
         try {  
            orders.add(new OrderSpecifier(direction, orderByExpression.get(order.getProperty())));  
         }  
         catch (Exception e) {  
            log.error("PfitQuerydslRepositorySupport getOrderSpecifiers exception = %s".formatted(e.toString()));  
         }  
      });  
      return orders.toArray(OrderSpecifier[]::new);  
   }  
}
```

확장 포인트 2가지에 대한 구현과 장점에 대해서 알아보도록 하자.

### **조건문의 추상화**

QueryDsl을 이용한 일반적인 조회 방법입니다.

```java
public List<Category> getList(CategoryPagingRequestDTO req) {  
   return from(qCategory)  
         .where(  
               qCategory.categoryId.eq(req.getCategoryId()),  
               qCategory.categoryName.eq(req.getCategoryName()),  
               qCategory.categorySort.eq(req.getCategorySort())  
         )  
         .fetch();  
}
```

위 3가지 조건은 존재할 수도 있고 아닐 수도 있습니다. 그럼 여기서 파리미터 req의 categoryId가 없을 경우는 어떻게 될까요? 예외가 발생합니다.

그럼 어떻게 처리할 수 있을까요? BooleanBuilder를 사용해 해당 데이터의 유무를 비교하여 처리할 수 있습니다. 하지만 하나의 컬럼 별로 일일이 작업을 해야 하기 떄문에 생산성에 영향을 주는 요소입니다. 그리고 실수로 잘못작성하는 경우 예외를 유발할 수도 있습니다.

그렇다면 위와 같은 이슈를 어떠한 방식으로 정리할 수 있을까요?
제가 선택한 방식은 `QuerydslRepositorySupport` 확장하여 아래 메소드를 추가하였습니다.

```java
public <T extends SimpleExpression> BooleanExpression eq(T path, Object param){  
  return ObjectUtils.isEmpty(param) ? null : path.eq(param);  
}
public BooleanExpression like(StringPath path, String param){  
   return ObjectUtils.isEmpty(param) ? null : path.like(param);  
}
```

```java
public List<Category> getList(CategoryPagingRequestDTO req) {  
   return from(qCategory)  
         .where(  
               eq(qCategory.categoryId, req.getCategoryId()),  
               like(qCategory.categoryName, req.getCategoryName()),  
               eq(qCategory.categorySort, req.getCategorySort())
         )  
         .fetch();  
}
```

이렇게 작업할 경우 별도의 BooleanBuilder를 추가할 필요가 없고 실수를 방지하며 생산성을 높여줄 수 있습니다.

### **동적 Sort 기능**

QueryDsl을 이용한 Sort 방법입니다.

```
public List<Category> getList(CategoryPagingRequestDTO req, Sort sort) {  
   return from(qCategory)  
         .where(  
               qCategory.categoryId.eq(req.getCategoryId()),  
               qCategory.categoryName.eq(req.getCategoryName()),  
               qCategory.categorySort.eq(req.getCategorySort())  
         )  
         .orderBy(qCategory.categorySort.asc())   
         .fetch();  
}
```

categorySort라는 필드를 asc로 정렬하여 반환요청을 하고 있습니다.

그렇다면 클라이언트의 요청에 따라 Sort의 기준이 달라야 한다면 어떻게 처리해야 할까요? Sort 기준 필드를 클라이언트의 요청으로 주입받아 사용하는 것이 가장 생산성이 우수할 것입니다.

위와 같은 문제로 생산성을 높이기 위해 `QuerydslRepositorySupport` 확장하여 아래 메소드를 추가하였습니다.

```java
public OrderSpecifier[] getOrderSpecifiers(C clazz, Sort sort) {  
  if(ObjectUtils.isEmpty(sort))   return new OrderSpecifier[]{};  
  List<OrderSpecifier> orders = new ArrayList<>();  
  PathBuilder orderByExpression = new PathBuilder(clazz.getClass(), clazz.toString(), PathBuilderValidator.FIELDS);  
  sort.stream().forEach(order -> {  
	 Order direction = order.isAscending() ? Order.ASC : Order.DESC;  
	 try {  
		orders.add(new OrderSpecifier(direction, orderByExpression.get(order.getProperty())));  
	 }  
	 catch (Exception e) {  
		log.error("PfitQuerydslRepositorySupport getOrderSpecifiers exception = %s".formatted(e.toString()));  
	 }  
  });  
  return orders.toArray(OrderSpecifier[]::new);  
}  
```

```java
public List<Category> getList(CategoryPagingRequestDTO req, Sort sort) {  
   return from(qCategory)  
         .where(  
               qCategory.categoryId.eq(req.getCategoryId()),  
               qCategory.categoryName.eq(req.getCategoryName()),  
               qCategory.categorySort.eq(req.getCategorySort())  
         )  
         .orderBy(getOrderSpecifiers(qCategory, sort))  
         .fetch();  
}
```

getOrderSpecifiers 메소드를 통해 클라이언트의 요청 또는 내부적 메소드에서 동적으로 Sort를 변경 수 있습니다.

## **마치며**
조건문의 추상화는 각 유형의 타입과 기능에 따라 확장하여 사용할 수 있으며 Sort의 경우는 메소드 하나로 모든 정렬을 담당할 수 있습니다. `QuerydslRepositorySupport` 를 확장하여 생산성 향상과 코드의 재활용성을 높이고 기능상의 이점을 취할 수 있었습니다. 