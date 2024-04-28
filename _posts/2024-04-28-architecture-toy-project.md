---
title: (Architecture) 토이 프로젝트 패키지 구조
date: 2024-04-28 14:00:00 +0800
description: 일반적인 패키지 구조의 유형과 프로젝트의 규모/기능을 통해 패키지 구조를 결정해보자.
excerpt: 일반적인 패키지 구조의 유형과 프로젝트의 규모/기능을 통해 패키지 구조를 결정해보자.
categories:
  - Architecture
tags:
  - 토이프로젝트
  - Architecture
  - Package
  - 패키지구조
---

토이 프로젝트의 기능 구현에 앞서 패키지 구조의 유형에 대해 알아보고 어떠한 패키지 구조로 진행할지 결정하는 시간을 갖기로 했다.

패키지의  구조는 크게 레이어 계층형, 도메인형 크게 2가지 유형으로 구분된다. 각 유형에 대해 간단히 설명하고 어떠한 방향으로 개발해 나갈지에 대한 기준을 잡아보도록 하자.

## **패키지 구조 유형**

### 계층형
```
com  
└── example  
    └── demo  
        ├── DemoApplication.java  
        ├── config  
        ├── controller  
        ├── dao  
        ├── domain  
        ├── exception  
        └── service
```

계층형 구조는 각 계층을 대표하는 요소들이 패키지로 구성된다.

#### **계층형 구조의 특징**

##### **장점**
- 프로젝트의 이해가 상대적으로 낮아도 전체적인 구조를 빠르게 파악할 수 있다.

##### **단점**
- 규모가 커지면 하나의 패키지에 많은 클래스들이 생성되어 구분이 힘들다.
- 도메인별 응집도가 낮아 도메인 관련 내용을 확인하려면 여러 패키지에 접근 해야한다.

### 도메인형

```
com  
└── example  
    └── demo  
        ├── DemoApplication.java  
        ├── trade  
        │   ├── controller  
        │   ├── domain  
        │   ├── exception  
        │   ├── repository  
        │   └── service  
        ├── user  
        │   ├── controller  
        │   ├── domain  
        │   ├── exception  
        │   ├── repository  
        │   └── service  
        └── item  
            ├── controller  
            ├── domain  
            ├── exception  
            ├── repository  
            └── service

```

도메인을 기준으로 패키지를 구성하고 도메인 내에 계층에 대한 패키지가 존재한다,

#### **도메인형 구조의 특징**
##### **장점**
- 도메인별 응집도가 높아져 도메인에 대한 정보를 하나의 패키지에서 확인 가능하다.

##### **단점**
- 프로젝트의 이해가 낮다면 전체적인 흐름 파악이 힘들다.
- 도메인의 구조를 기초로 하기때문에 애매한 클래스들이 발생한다.

계층형, 도메인형 패키지는 장/단점이 상반되는 구조라고 생각된다. 그렇기에 프로젝트의 규모 및 상황에 따라 장점을 비교하여 선택해야 한다.

### **최종 프로젝트 패키지 구조**

패키지 구조에 대한 유형을 알아봤다. 이제 진행할 프로젝트의 관점에서 어떠한 구조로 가는 것이 이점이 있을지 생각해보자.
개발 하려는 프로젝트의 구성 상 도메인형 구조가 더 적합하다. 개발할 프로젝트는 일반 사용자 / 관리자 / 작업자 로 구별하여 각 사용자 유형에 맞는 도메인 기능을 사용한다. 
만약 도메인별 응집도가 낮은 계층형 구조를 사용한다면 계층 패키지 안에서 추가 패키지를 생성하여 사용자 유형을 구별하는 방식으로 구조를 설계할 것이다. 계층 안에 추가 패키지가 생성되면 이는 개발자의 프로젝트 구조 파악에도 좋지 않을 것이라 생각한다.
그렇기에 도메인형 패키지 구조로 선택하였다.

그럼 도메인형 패키지 구조의 단점 중 하나 인 애매한 클래스들을 어떠한 방식으로 해결할지 고민해보자.

```
com  
└── example  
    └── demo  
        ├── DemoApplication.java  
        ├── domain
		│   ├── user
		|	|   ├── controller
		|	|   ├── dto
		|	|   ├── define
		|	|   ├── entity
		|	|   ├── repository
		|	|   ├── service
		|	|   └── usecase
		│   └── trade
        ├── global
        │   ├── common
		|   ├── utils
		|   ├── exception
        │   └── config
		|       └── SecurityConfig.java
	    └── infra
            ├── alarm
            └── sms
			    └── SmsClient.java
```


#### **domain**
domain 패키지는 기존 도메인형 패키지를 담당한다.
- dto - request, response 및 부가적인 dto가 위치한 package
- define - enum과 const 클래스가 위치한 package
- repository - dao의 개념으로 JpaRepository를 구현한 클래스와 QueryDsl 클래스가 위치한 package
- service - 해당 도메인에 한정된 service 클래스가 위치한 package
- usecase - 해당 도메인과 외부 도메인을 연결해주는 역할을 담당하는 service 클래스가 위치한 package

#### **global**
global은 설정과 공통 클래스를 관리한다.
- common - 공통적 요소에 대한 클래스가 위치한 package
- utils - 유틸리티 클래스가 위치한 package
- config - 스프링 설정을 관관리하는 package
- exception - 예외 핸들링 과 커스텀 예외를 관리하는 package

#### **infra**
infra는 외부 서비스에 대한 클래스를 관리하는 패키지이다.

## **마치며** 
토이 프로젝트의 패키지 구조를 설계해보았다. 계층형, 도메인형을 시작으로 프로젝트에 적합한 그리고 추가 해야 할 요소들을 정리하고 보완하여 최종 결론을 도출하였다. 패키지 구조는 큰틀에서 벗어나지 않지만 프로젝트의 규모 / 기능에 대해 고려하고 필요한 패키지를 추가하며 구조화 해볼 수 있는 유익한 시간이였다.. 