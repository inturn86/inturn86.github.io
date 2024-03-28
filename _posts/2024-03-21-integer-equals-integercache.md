---
title: (Java) Integer 동등 연산(==) 결과에 대한 문제
excerpt: Integer 동등 연산(==)의 결과는 ? 128, 127의 차이는?
date: 2024-03-21 21:55:00 +0800
categories:
  - Java
tags:
  - Java
  - Integer
  - IntegerCache
pin: false
img_path:
---
Integer는 개발 시에 흔히 사용되는 래퍼 클래스이다. 오늘은 Integer를 동등 연산( == ) 할 때 발생하는 이슈에 대해서 알아보도록 하자.

## Integer 동등 연산 문제 상황

```java
@Test
void integerEqualsTest() {
	Integer comp1 = 128;
	Integer comp2 = 128;
	System.out.println("128 equals result = " + comp1 == comp2); 

	Integer comp3 = 127;
	Integer comp4 = 127;
	System.out.println("127 equals result = " + comp3 == comp4); 
}
```
위 테스트 코드를 실행 결과를 예측해보자.

실행 결과는 아래와 같다.
```
128 equals result = false
127 equals result = true
```

Integer는 **래퍼 클래스(Wrapper Class)로 동등 연산자로 비교할 경우 참조 주소값으로 비교하기에 모두 false가 나와야 할 것 같지만, 127로 비교했을 때는 true**를 반환했다. 이유를 하나씩 알아보자.

## 래퍼 클래스

래퍼 클래스는 기본 자료형을 객체로 다루기 위해 사용하는 클래스이다. 기본 자료형을 래퍼 클래스로 리터럴하면 자바의 컴파일러가 자동으로 오토박싱(Auto Boxing)이 처리된다. 반대로 래퍼 클래스를 기본 자료형으로 리터럴하면 오토언박싱(Auto Unboxing)이 된다.
```java
Integer comp3 = 127;

//기본 자료형을 래퍼 클래스로 Auto Boxing 처리
Integer comp3 = Integer.valueOf(127);

int comp4 = comp3;

//래퍼 클래스를 기본 자료형으로 Auto Unboxing 처리
int comp4 = comp3.intValue();
```

일반적으로 **래퍼 클래스는 객체이기에 생성할 때 마다 변수에 참조 주소값이 저장**된다. 따라서 동등 연산자로 비교할 경우 **참조 주소값을 비교하기에 false가 나오는 것이 맞다**.

그렇다면 **Inter.valueOf(127)가 어떤식으로 객체를 생성**하는지 살펴보자.

## IntegerCache

```java
@IntrinsicCandidate  
public static Integer valueOf(int i) {  
    if (i >= IntegerCache.low && i <= IntegerCache.high)  
        return IntegerCache.cache[i + (-IntegerCache.low)];  
    return new Integer(i);  
}
```

valueOf 메소드에는 IntegerCache라는 static 클래스를 사용하고 있는 걸 확인할 수 있다. valueOf의 매개변수 i가 low(-128)와 high(127)의 사이이면 **새로운 객체를 생성하지 않고 IntegerCache.cache[]를 반환**한다.

```java
private static class IntegerCache {  
    static final int low = -128;  
	static final int high;  
	static final Integer[] cache;  
	static Integer[] archivedCache;

	static {
		...
		int size = (high - low) + 1;  
		// Use the archived cache if it exists and is large enough  
		if (archivedCache == null || size > archivedCache.length) {  
		    Integer[] c = new Integer[size];  
		    int j = low;  
		    for(int i = 0; i < c.length; i++) {  
		        c[i] = new Integer(j++);  
		    }  
		    archivedCache = c;  
		}  
		cache = archivedCache;
	}
}    
```

그렇다면 **IntegerCache는 언제 초기화**가 될까? IntegerCache는 **static block에 의해 해당 클래스가 메모리에 로드되는 시점에 초기화**된다. 이때 cache라는 배열에 -128에서 127 범위의 Integer 객체를 캐싱한다.
이렇게 해당 범위의 정수 객체를 캐싱처리하는 이유는 **개발할 때 빈번히 사용되기 때문**이다.

IntegerCache의 최대값은 JVM 옵션을 통하여 조정할 수 있다.

```
-XX:AutoBoxCacheMax
```

또한 다른 래퍼 클래스(Byte, Short, Long, Character)도 위와 같은 Cache 기능을 가지고 있다. 하지만 Integer만 최대값을 조정할 수 있도록 되어있다.

## Integer 생성자

다음으로 new Integer(int) 생성자에 대해서 추가로 알아보자.

```java
//이 생성자는 
Integer comp5 = new Integer(127);


@Deprecated(since="9", forRemoval = true)  
public Integer(int value) {  
    this.value = value;  
}
```

위 Integer 생성자는 JDK 9  이후로 deprecated 되었다. 대신 Integer.valueOf(int)로 사용하길 권장하하고 있다. 이유는 **'저장 공간 및 시간 성능이 향상될 가능성이 높다'**  라고 설명한다. 이 내용을 IntegerCache와 연관지어 생각한다면 **빈번히 사용되는 정수값을 래퍼 클래스로 생셩할 때 IntegerCache를 통해 생성되지 않아도 될 객체를 선별하여 Heap 영역의 메모리를 줄일 수 있기 때문이다**.

Integer의 동등 연산 결과에 대해 발생할 수 있는 이슈를 설명했다. 가장 중요한 것은 **동등성(참조하는 객체가 같음)과 동일성(객체의 참조 값이 같음)을 명확히 구별**해야한다. 래퍼 클래스의 동등성을 비교하기 위해 동등 연산( == )을 사용하는 경우가 있지만 **대부분 동일성 비교를 위해 잘못 사용한다**. 따라서 **래퍼 클래스의 동등성을 비교할 때에는 equals 메소드**를 사용하자.

