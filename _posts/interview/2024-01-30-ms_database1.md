---
title: CS 면접대비 - 데이터베이스
date: 2024-01-24 12:00:00 +/-TTTT
categories: [Interview]
comments: true
tags:
---

<details><summary>데이터베이스 키에 대해서 설명해주세요</summary><div markdown="1">
1. Candidate Key
   1. Unique 식별
   2. 최소 속성
2. Foreign Key
   1. 다른 테이블의 Primary Key
3. Primary Key
   1. Candidate Key중 선택된 Main Key
4. Alternate Key
   1. Candidate Key - Primary Key
5. Super Key
   1. Unique 식별하지만 최소 속성은 만족 못하는 키
</div></details>

<details><summary>데이터베이스를 왜 사용할까요?</summary><div markdown="1">
* 데이터베이스 이전에는 파일시스템을 사용하여 데이터를 식별하였습니다.  
디스크 IO는 느리고, 실제 어플리케이션은 랜덤 IO를 많이 발생시킵니다.  
이것을 해결하기 위해서 랜덤 IO를 순차 IO로 바꿔서 실행하는걸 통해 더 성능적 이득이 있습니다.  
</div></details>

<details><summary>트랜잭션</summary><div markdown="1">
* ***트랜잭션은 무엇인가요? 왜 필요한가요?***
  * 데이터베이스에서 논리적인 작업의 단위입니다.

- 트랜잭션 특징

  - 원자성(Atomicity) : all or nothing
  - 일관성(Consistancy) : 항상 동일한 결과
  - 독립성(Isolation) : 다른 트랜잭션이 끼어들 수 없다
  - 지속성(Durability) : 트랜잭션의 결과는 영구적으로 반영

- 트랜잭션 격리수준
  - **_트랜잭션 격리수준이 왜 필요한가요? _**
    - 트랜잭션은 ACID 원칙을 지켜야 하지만, 완벽하게 지키는 형태는 동시적인 요청에 대한 처리량이 떨어지게 됩니다.
      따라서 데이터베이스는 애플리케이션 개발자가 본인들의 환경에 맞게 트레이드오프를 고려하여 선택할 수 있도록 격리레벨을 만들어두었습니다.
      - **_어떤 트레이드오프인가요?_**
        - 동시적 처리량을 늘릴수록 이상현상이 더 많이 발생합니다.
          - 대표적인 이상현상을 말씀드리면 아래가 있습니다.
            - DIRTY READ : 트랜잭션 실행도중 다른 트랜잭션의 업데이트에 의해 수정된 데이터를 읽는것
            - PHANTOM READ : 동일한 쿼리에 대해서 두번째 쿼리에서 없던 결과가 나오는것
            - NON REPEATABLE READ : 트랜잭션 실행도중 두번째 읽었을 때 다른 값이 나오는것
      - **_격리수준이 이 문제를 어떻게 해결해주나요?_**
        - Serializable : 모든 트랜잭션이 순차적으로 실행되도록 합니다 어떠한 문제도 일어나지 않습니다.
        - Repeatable Read: MySQL의 기본 설정입니다.  
          변경 전 레코드를 언두 공간에 백업해두고 앞서 처리된 트랜잭션에 대해서는 언두 공간의 값을 읽게 합니다.
          트랜잭션이 앞서 처리되었는지는 트랜잭션마다 번호를 부여하여 판단합니다.
          일반적인 Phantom Read 또한 이후 트랜잭션 번호를 가진 언두로그를 봐서 무시하면 되지만
          SELECT FOR UPDATE와 같이 같이 관여할 경우, 락은 언두로그가 아닌 테이블값을 읽기 때문에
          Phantom Read가 발생할 수 있습니다.
        - Read Commited : Commit된 데이터만 읽을 수 있도록 하는 설정입니다.
          Commit 된 데이터를 읽을 경우 Non Repeatable Read가 발생할 수 있습니다.
        - Read Uncommited : Commit 되지 않은 데이터도 읽을 수 있도록 하는것. 모든 이상현상 발생.

</div></details>

<details><summary>정규화</summary><div markdown="1">
* ***정규화란 무엇인가요? 왜 필요한가요?***
  * 정규화란 데이터베이스 테이블들의 관계에서 중복을 최소화시키는 방법입니다.
* ***역정규화?***
  * 테이블에 대한 조인이 성능적으로 문제가 되는 경우(경로가 멀어서)
  * 조회에 대한 처리성능이 중요하다고 판단된다면

</div></details>

<details><summary>인덱스</summary><div markdown="1">
* ***인덱스가 무엇인가요? 왜 필요한가요?***
    * 데이터베이스는 랜덤 IO 엑세스와의 싸움입니다.
    추가적인 쓰기작업과 저장공간을 사용하여 데이터베이스 테이블의 검색 속도를 향상시키기 위한 자료구조를 의미합니다.
    * ***어떤 자료구조인가요?***
      * 일반적으로는 B+ 트리를 이용합니다.
        * ***왜 B+ 트리를 이용할까요?***
          * B+ 트리는 Leaf Node간 연결이 되어있기때문에
            * FullScan과 Range Scan에 유리합니다.
* ***인덱스의 성능 또는 고려해야할 사항?***
    * WHERE 절에 자주 사용되는 열
    * SELECT 절에 자주 등장하는 칼럼
    * JOIN에 자주 사용되는 열
    * 데이터 중복도가 높은 열은 효과가 없음
      * 남성/여성
    * WHERE 절을 변형하면 인덱스를 안타는 이유
      * 결국 시작점을 알 수 없기 때문임 LIKE 또는 SUBSTR
    * 복합 인덱스를 구성할 때 칼럼의 순서를 고려하는 기준
      * 분포도는 의미가 없다
        * 연산자가 "=" 가 아닐 수 있기 때문
      * 연산자를 최우선적으로 고려하라
        * 분포도가 좋더라도 연산자가 선분조건이면 앞에 두는게 안좋을 수 있다는것

</div></details>
