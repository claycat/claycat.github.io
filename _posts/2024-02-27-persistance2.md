---
title: 영속성 까보기 (feat. 10분 테코톡)
date: 2024-02-27 12:00:00 +/-TTTT
categories:
comments: true
tags: ["JPA"]
---


## **JPA와 영속성**

JPA 영속성 관련 실험 중 이상한 케이스를 발견했습니다.  
[우아한형제들 테코톡](https://www.youtube.com/watch?v=kJexMyaeHDs&t=92s)의 문제상황 1에서도 비슷한 예제가 있습니다.  
영상에서는 정확히 이유를 설명하지는 않고 영속성 설명으로 넘어가버려서
정확한 이유를 정리해보려고 합니다.

```java
@Getter
@Entity
@AllArgsConstructor
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Person {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    public Person(String name) {
        this.name = name;
    }
}
```


```java
@DataJpaTest
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class PersonRepositoryTest {
    @Autowired
    private PersonRepository personRepository;

    private static final Person JOHN = new Person("John");


    @Test // Test 1
    @Order(1)
    void saveAndCompare() {
        Person person = personRepository.save(JOHN);
        assertThat(person).isEqualTo(JOHN);
    }

    @Test // Test 2
    @Order(2)
    void find() {
        Person person = personRepository.save(JOHN);
        assertThat(person).isEqualTo(JOHN);
    }

}
```

### **결과**
위에 작성되어있는 `Person` 클래스의 테스트 코드인  
Test1 과 Test2를 **각자** 돌렸을 때는 **성공**하지만,    
**동시**에 돌리게 되면 **실패**합니다.

일반적으로 `@DataJpaTest`에는 `@Transactional`이 달려있기 때문에,  
하위 테스트들 모두에 `@Transactional`이 적용됩니다  
@Transactional이 걸린 테스트들은 테스트 마지막에 rollback이 된다는 사실도 알고 있죠.  
트랜잭션이 롤백되면 영속성 컨텍스트도 초기화되어 직관적으로는  
두개 다 성공하거나 두개 다 실패해야 할 꺼라고 생각되는데,  
**왜 둘 중 하나만 성공하는걸까요?**

*영속성 컨텍스트가 동기화가 되지 않은걸까요?*  
하지만 `em.flush()` 와 `em.clear()`을 Test1 마지막에 추가하더라도   
결과가 동일한것을 볼 수 있을 것입니다.

왜 실패하는지 그 이유에 대해 단계별로 쪼개서 알아보도록 합시다.

## **Test 1 에서는 어떤일이 일어나는가**
`Person person = personRepository.save(JOHN);`   
단 한줄입니다

save 메소드의 내부 코드를 간단하게 살펴보면 다음과 같습니다 (분량상 몇줄 생략)

```java
@Transactional
public <S extends T> S save(S entity) {
    if (entityInformation.isNew(entity)) {
        entityManager.persist(entity);
        return entity;
    } else {
        return entityManager.merge(entity);
    }
}
```

아무것도 없는 상태에서 테스트 1을 수행했기 때문에,   
영속성 컨텍스트에 해당 엔티티가 존재하는지 확인하는 isNew 의 결과에 따라서 분기가 갈립니다.

간단하게 isNew메소드의 구현을 보면 아래와 같습니다

```java
public boolean isNew(T entity) {
    ID id = getId(entity);
    Class<ID> idType = getIdType();
    
    if (!idType.isPrimitive()) {
        return id == null;
    }
    
    if (id instanceof Number) {
        return ((Number) id).longValue() == 0L;
    }
}
```

id의 타입이 원시타입이 아니라면 null일 때 true 입니다.  
따라서 isNew는 true를 반환합니다   
(**Long은 wrapper 클래스이기때문에 원시타입이 아닙니다.**)


이후 entityManager.persist 메소드 호출을 통해서 해당 엔티티는 영속성 컨텍스트에 등록됩니다.
인자로 넘긴 값을 그대로 반환하기 때문에 당연히 첫 `assert`는 성공합니다


## **영속성 컨텍스트와 롤백**

첫번째 테스트가 종료되었습니다.
첫번째 테스트가 롤백되면, 데이터베이스에 저장되어있는 `JOHN` 엔티티는 사라질 것입니다.
마찬가지로, 영속성 컨텍스트 또한 초기화되면서 내부의 엔티티들은 `detached` 상태로 전환됩니다
> transaction rollback causes all pre-existing managed instances and removed instances [31] to become **detached**  
[JSR-000317 Persistence Specification for Eval 2.0 Eval]

간단하게 `em.contains(JOHN)` 과 `personRepository.find(JOHN)`을 통해서 확인할 수 있고,  
예상대로 둘다 `false`를 반환합니다.

하지만 id는 어떨까요? 
Test2의 첫번째 줄에 JOHN.getId()를 하면 null 이 아닌 1이 출력되는것을 확인할 수 있습니다.  
**트랜잭션이 롤백되더라도 in-memory로 올라가있는 static 객체인 JOHN에 추가된 id는 회수되지 않는것입니다.**  
**그리고 이게 두번째 테스트가 실패하는 원인입니다.**  

다시 save로 돌아가봅시다.
마찬가지로 isNew 까지 들어가는데, 이번에는 id 가 null이 아니기 때문에 em.merge()를 호출합니다.  
em.merge는 반환할 때 새로운 영속상태의 엔티티를 생성해서 반환합니다.  
  
*(주의:전달된 인자가 영속상태로 관리되는것이 아니라, 영속상태로 관리되는 새로운 객체를 반환)*
*(em.contains(JOHN)을 해본다면 false가 출력됩니다)*

즉, JOHN과는 전혀 다른, 영속상태로 관리되는 객체를 의미하는것입니다.  
따라서 assert문은 실패하게 됩니다.

테스트간 엔티티를 공유하는것이 문제가 될 수 있는 시나리오였습니다.  
이상입니다. 


### **부록 - 왜 id는 트랜잭션이 롤백되더라도 회수되지 않는가?**
id는 왜 롤백되더라도 원래 상태로 복구되지 않습니다.  
즉 id에 대한 값 부여는 트랜잭션과 별도로 일어나는 작업입니다.  
왜 그럴까요?  
만약 진행중인 트랜잭션이 끝나지 않고 5분동안 락을 소유하게 된다면
해당 테이블에 값을 레코드를 추가하고 싶은 다른 트랜잭션들의 작업이 모두 막히기 때문입니다.
> from [StackOverflow](https://stackoverflow.com/a/67401366), Vlad Mihalcea
