---
title: 영속성 핥아보기
date: 2024-01-24 12:00:00 +/-TTTT
categories: ["JPA"]
comments: true
tags: 
---

# 영속성이란
* 영원히 계속되는 성질이나 능력
* 영속화 : 어플리케이션의 상태와 상관없이 물리적인 저장소에 데이터를 저장하는 행위
    * 데이터를 **어떤** 공간에 **어떤** 형태로 저장할 것인가?
    * RDMBS에 데이터를 저장하는 방법은 SQL이라고 볼 수 있다

### 자바의 경우 
* 자바 어플리케이션에서는 JDBC 인터페이스를 사용
    * 각 데이터베이스 제조사들은 JDBC 인터페이스를 구현하는 클래스(드라이버) 들을 제공
    * 순수 JDBC를 사용할 경우
        * 데이터베이스 접속, 질의, 유저 객체로의 변환 절차가 필요
        * 모두 동일한 파일에서 관리해야 한다
            * 칼럼의 이름명이 변경 등 사소한 테이블의 변경이 sql 문자열에 전파됨

### 객체와 테이블의 불일치
* 객체지향 어플리케이션과 RDBMS의 불일치를 맞추는데 많은 리소스 소모
* 이를 해결해주는것이 Persistence Framework
    * SQL Mapping (MyBatis - 실제 SQL 코드와 비즈니스 로직의 분리)
        * .xml 형태로 외부로 빼내준다
    * OR Mapping (객체와 관계형 데이터베이 사이의 매핑, SQL을 생성)

# JPA
* JPA를 통해 객체를 영속화 하려고 사용하는것이 EntityManager이고, EntityManagerFactory를 통해 얻음
* EntityManager는 영속성 컨텍스트를 통해 영속 객체를 관리
    * 영속 객체 = 실제로 데이터가 영속적으로 저장되기 위한 객체
        * Customer c = new Customer()가 entityManager를 통해 영속성 컨텍스트에 등록되고  
          트랜잭션 내에서 관리되다가 끝났을 때 데이터베이스 안으로 등록된다
* 사용자는 EntityManager 인스턴스 객체의 메소드를 이용해 영속 객체를 관리
* 근데 왜 실제로 EntityManager EntityManagerFactory 실제 코드에서는 없음 ? 
    * 이런 작업들은 Spring Framework에 위임할 수 있음


## 영속성 컨텍스트 - INSERT AND SELECT
* 고유 ID를 갖는 모든 영속 객체 인스턴스에 대한 집합

```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory();
EntityManager em = emf.createEntityManager();
EntityTransaction tx = em.getTransaction();

Customer customer = new Customer(); //아직 일반객체
em.persist(customer); //이제 등록됨 (1차 캐시) 아직 INSERT는 X
em.persist(customer); //캐시에 이미 해당 객체 있어서 do nothing

Customer c1 = em.find(Customer.class, "ID0001"); // SELECT X, 1차 캐시에 있는 데이터를 받아온다

tx.commit() //이순간 INSERT 발생 
```
* **모든건 트랜잭션 안에서 일어난다** 
    * transaction 내부에서 persist가 일어나야 1차 캐시에 들어가있는것

* **괜히 캐시라고 부르는게 아니다**
    * `entityManger.find` 의 경우, Persistence Context에 있다면 그녀석을 가져오고
    * 만약 없다면 db에 select문을 날려서 가져온다
        * **select 문을 통해 가져온 객체는 다시 persistence context에 보관된다**

* **SQL 저장소** 
    * persist 메소드가 호출될 경우, SQL 저장소(쓰기 지연)에 SQL 저장됨
        * flush / transaction commit 시점에 저장되어있는 SQL들을 실행

## 영속성 컨텍스트 - UPDATES
* 객체의 상태 변경
    * UPDATE 쿼리들은 마찬가지로 SQL 저장소에 보관 후 커밋 시점에 반영
    * `foundCustomer.setName("Kim")` 할경우, 현재 객체와 snapshot의 상태와 비교하고,  
        만약 다를 경우 UPDATE 쿼리를 SQL 저장소에 저장함
        
## Flush
* 영속성 컨텍스트의 내용을 데이터베이스와 동기화 하는것
* flush 방식
    * `EntityManager.flush()` 직접 호출
    * 트랜잭션의 커밋 
    * JPQL 쿼리 실행
* flush를 실행하더라도 영속성 컨텍스트 내용은 유지됨
* flush가 실행되더라도, 데이터베이스 내용이 완전히 변경된것은 아님 
    * transaction이 완성되어서 롤백이 안되는 시점까지 간 것이 아니다
    * 커밋하기 이전, 쿼리가 "반영된 시점"까지를 의미함

```java
tx.begin()
try {
    Customer customer = new Customer("ID001", "KIM");
    em.persist(customer);
    // 이 시점에는 우리의 java application에서는 persistence context와 db 동기화
    // 하지만 다른 어플리케이션 (h2 console) 에서는 tx.commit()을 하지 않는이상 반영 X

    tx.commit();
    // 이 시점 이후로는 반영 완료
} catch(Exception e) {
    tx.rollaback();
}
```
