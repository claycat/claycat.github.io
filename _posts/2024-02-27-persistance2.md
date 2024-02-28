---
title: 영속성 핥아보기2 (Feat. 10분 테코톡)
date: 2024-02-27 12:00:00 +/-TTTT
categories:
comments: true
tags: ["JPA"]
---


## **JPA와 영속성**
JPA에 대해서 학습하다 [우아한형제들 10분 테코톡](https://www.youtube.com/watch?v=kJexMyaeHDs&t=92s)을 보게 되었습니다.

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
        Optional<Person> findPerson = personRepository.findById(person.getId());

        assertThat(findPerson).hasValue(JOHN);
    }

}
```

### **요약**
위에 작성되어있는 `Person` 클래스의 테스트 코드인  
Test1 과 Test2를 **각자** 돌렸을 때는 **성공**하지만,    
**동시**에 돌리게 되면 **실패**한다는 것이었습니다.

영상에서는 해당 테스트의 실패 이유에 대해서 정확하게 짚고 넘어가지는 않고,  
영속성 컨텍스트에 대한 설명을 하고 바로 두번째 케이스로 넘어갑니다.

해당 부분을 지적하는 몇개의 댓글들도 있어서 정리해보려고 합니다.

## Test 1 에서는 어떤일이 일어나는가
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
영속성 컨텍스트에 해당 엔티티가 존재하는지 확인하는 isNew 메소드는 true를 반환합니다.
이후 entityManager.persist 메소드 호출을 통해서 해당 엔티티는 영속성 컨텍스트에 등록됩니다 


## Test 1 이후 시작된 Test 2에서는 어떤일이 일어나는가
두번째 테스트에서는 어떤일이 일어날까요?  




## Common JPA pitfall

