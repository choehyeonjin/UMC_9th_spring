- **지연로딩과 즉시로딩의 차이**
    - 로딩: 엔티티를 언제 로딩할 지에 대한 것
    - 즉시 로딩: 엔티티를 로드할 때 연관 엔티티까지 즉시 함께 조회.

        ```java
            @Entity
            @Table(name = "member_food")
            public class MemberFood {
                @ManyToOne(fetch = FetchType.EAGER) @JoinColumn(name = "member_id")
                private Member member;
                @ManyToOne(fetch = FetchType.EAGER) @JoinColumn(name = "food_id")
                private Food food;
            }
        ```

        - findAll() 시점에 member, food까지 즉시 조인/조회.
        - 즉시 데이터 접근이 가능.
        - 목록에서 거의 쓰지 않는 연관까지 매번 당겨 과다 데이터·조인 비용 증가.
        - 쿼리 최적화 관련하여 눈에 보이지 않는 쿼리까지 발생.
    - 지연 로딩: 연관 엔티티는 프록시로 두고, 실제로 접근할 때 쿼리 실행.

        ```java
        @Entity
        @Table(name = "member_food")
        public class MemberFood {
            @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "member_id")
            private Member member;
            @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "food_id")
            private Food food;
        }
        ```

        - 처음에는 불필요한 데이터 로딩 방지
        - 초기에는 MemberFood만 SELECT. member.getName() 등 접근 시점에 쿼리 발행.
---
- **JPQL**

    - **개념**
        - JPA의 객체 지향 쿼리 언어
        - SQL처럼 보이지만, 테이블이 아닌 엔티티(Entity) 와 필드를 대상으로 동작함
        - 즉, 데이터베이스 중심이 아니라 객체 중심의 쿼리를 작성하게 도와줌
    - 예시
        ```java
        @Query("select m from Member m where m.name = :name")
        List<Member> findByName(@Param("name") String name);
        ```

    - 사용법
        1. 메서드 이름 기반 쿼리 생성
            - Spring Data JPA가 메서드 이름을 해석해 JPQL 자동 생성

                ```java
                List<Member> findByNameAndDeletedAtIsNull(String name);
                ```

            - 자동 실행되는 JPQL

                ```java
                select m from Member m where m.name = :name and m.deletedAt is null
                ```

                - 장점: 간단한 조건이면 SQL을 직접 쓸 필요 없음
                - 단점: 복잡한 조건일수록 메서드 이름이 너무 길어짐
        2. @Query 어노테이션으로 직접 작성
            - 복잡한 조건이나 커스터마이징이 필요할 때 사용

                ```java
                @Query("select m from Member m where m.name = :name and m.deletedAt is null")
                List<Member> findActiveMember(@Param("name") String name);
                ```

                - :파라미터명으로 변수 바인딩
                - nativeQuery = true 옵션으로 SQL 직접 실행도 가능

  - 주의할 점

    - JPQL 자동 생성 쿼리가 항상 최적화되어 있진 않음 → 실행 쿼리 로그 꼭 확인
    - 연관관계 로딩 시 N+1 문제가 생길 수 있음
    - 복잡한 조건, 동적 쿼리, 타입 안정성이 필요한 경우엔 QueryDSL 사용이 더 적합

---
- **Fetch Join**
    - 개념
        - JPQL에서 연관된 엔티티를 한 번의 쿼리로 함께 조회하기 위한 방법 → 지연 로딩 쿼리에서 N+1 문제를 해결할 때 가장 많이 쓰임

        ```java
        @Query("select m from Member m join fetch m.memberFoodList")
        List<Member> findAllWithFoods();
        ```

    - 특징
  
        | 항목 | 설명 |
        | --- | --- |
        | **의미** | 연관 엔티티를 즉시 함께 가져옴 (조인해서 한 번에) |
        | **장점** | 쿼리 1번으로 필요한 데이터 전부 조회 가능 |
        | **단점** | 컬렉션 fetch join은 페이징 불가, 중복 로우 발생 가능 |
        | **활용 예시** | 상세 조회 등 연관 엔티티가 꼭 필요한 경우 |
---
- **@EntityGraph**
    - 개념

      JPA에서 **fetch join을 어노테이션으로 선언형으로 표현**할 수 있는 기능

      → JPQL 없이도 특정 연관 엔티티를 함께 로딩하도록 설정 가능.

        ```java
        @EntityGraph(attributePaths = {"memberFoodList"})
        @Query("select m from Member m")
        List<Member> findAllWithFoods();
        ```

    - 특징
  
        | 항목 | 설명 |
        | --- | --- |
        | **목적** | fetch join과 동일하게 N+1 방지 |
        | **차이점** | 코드보다 선언적으로 작성, 유지보수 쉬움 |
        | **적합한 경우** | 단순 조회나 기본 Repository 메서드 재사용 시 |
        | **주의** | join 조건 제어 불가능 (fetch join보다 제약 있음) |
  
---
- **commit과 flush 차이점**
    - 개념

      트랜잭션 커밋 과정에서 JPA는 내부적으로 **flush → commit** 순으로 실행됨.

      | 구분 | flush | commit |
              | --- | --- | --- |
      | **의미** | 영속성 컨텍스트의 변경 내용을 DB에 반영 (SQL 실행) | 트랜잭션을 실제로 종료하고 확정 |
      | **시점** | 커밋 직전 자동 실행 또는 직접 호출 가능 | flush 이후 실행됨 |
      | **예시** | 변경 감지(Dirty Checking) 후 update 쿼리 전송 | DB에 변경사항 영구 반영 |
---
- **QueryDSL, OpenFeign의 QueryDSL**
1. QueryDSL

    JPA에서 JPQL을 타입 안전하게 자바 코드로 작성할 수 있게 해주는 쿼리 빌더 라이브러리.
    
    → 문자열이 아닌 코드로 작성하므로, 컴파일 시점에 문법 오류를 잡고, IDE 자동완성 지원.
    
    ```java
    QMember m = QMember.member;
    
    List<Member> members = queryFactory
        .selectFrom(m)
        .where(m.name.eq("현진").and(m.deletedAt.isNull()))
        .fetch();
    ```

2. 개발 중단 및 보안 이슈

   마지막 공식 버전: 5.1.0 (2024.01 릴리스 이후 업데이트 없음)

   보안 취약점:

    - 사용자 입력값이 `PathBuilder`를 통해 쿼리에 직접 반영될 때 SQL Injection 위험 발생

   유지보수 현황:

    - 공식적으로 중단 선언은 없지만,

      새로운 기능/버전/보안 패치는 사실상 멈춤 상태

    결과:
    
    - 안정성, 보안성, 최신 자바/Spring과의 호환성 문제로 **대체 필요성 증가**
3. OpenFeign의 Query DSL

   Netflix의 OpenFeign 팀이 포크(fork)하여 새롭게 관리 중인 QueryDSL 프로젝트.

   → 기존 QueryDSL 문법을 그대로 유지하면서 보안 패치 + 성능 개선 + 최신 Java/Spring 호환성 강화 버전.

   → 코드 수정 없이 의존성만 바꾸면 바로 전환 가능 (Q클래스 패키지 구조 동일)
    
---
- **N+1 문제 해결할 수 있는 여러 방안들**

  ### 0) 전제

    - 연관 매핑의 기본은 LAZY로 두고, 조회 쿼리에서 명시적으로 필요한 연관만 가져온다.

    ---

  ### 1) Fetch Join (JPQL/QueryDSL)

    - 목적: 필요한 연관을 한 번의 쿼리로 함께 로딩
    - 예시 (JPQL)

        ```java
        @Query("select m from Member m join fetch m.team")
        List<Member> findAllWithTeam();
        ```

    - 예시 (QueryDSL)

        ```java
        queryFactory.selectFrom(member)
            .join(member.team, team).fetchJoin()
            .fetch();
        ```
    
    ### 2) @EntityGraph (선언형 fetch 계획)
    
    - 목적: 레포지토리 메서드에 선언적으로 연관 로딩 지정
        
        ```java
        @EntityGraph(attributePaths = {"team"})
        List<Member> findByName(String name);
        ```
        
    - 장점: fetch join 효과를 간단히 적용, 유지보수 용이
    - 제약: 조인 조건/방향 같은 세밀 제어는 fetch join보다 제한적
    
    ---
    
    ### 3) Batch Fetch (Hibernate)
    
    - 목적: LAZY 접근 시 ID들을 IN 쿼리로 묶어 쿼리 수 대폭 감소
        
        ```yaml
        spring.jpa.properties.hibernate.default_batch_fetch_size: 100
        ```
        
        혹은
        
        ```java
        @BatchSize(size = 100)
        private List<OrderItem> items;
        ```
        
    - 효과: 1 + N → 1 + ⌈N/배치사이즈⌉ 정도로 축소
    - 특징: ToOne/컬렉션 모두에 유효, 페이징과도 잘 맞음
    
    ---
    
    ### 4) DTO 전용 쿼리(프로젝션)
    
    - 목적: 화면/응답에 필요한 필드만 정확히 선택
        
        ```java
        @Query("select new com.app.MemberDto(m.id, m.name, t.name) " +
               "from Member m join m.team t")
        List<MemberDto> findList();
        ```
        
    - 장점: 과다 로딩 방지, 페이징 친화적
    - 패턴: 목록 API는 DTO, 상세 API는 fetch join/EntityGraph 조합
    
    ---
    
    ### 5) 쿼리 2단 분할(식별자 먼저 → 본문 나중)
    
    - 목적: 페이징 정확도와 N+1 동시 해결
        1. ID만 페이징: `select m.id from Member m order by m.id desc`
        2. 본문/연관 일괄 조회: `where m.id in (:ids)` + 필요한 조인
    - 장점: 정확한 페이징 + 필요한 연관만 한 번에
---
- **영속 상태의 종류**

  > 영속성 컨텍스트(Persistence Context)는 엔티티의 상태를 추적하며,
  >
  >
  > 엔티티는 생명주기 동안 아래 4가지 상태 중 하나를 가진다.
  >

  ### 1. 비영속 (Transient / New)

    - 엔티티가 아직 영속성 컨텍스트에 관리되지 않은 상태
    - 단순히 `new`로 생성한 객체로, DB와 전혀 연관 없음
    - 예시

        ```java
        Member member = new Member("현진"); // 아직 DB 저장 X
        ```

    ### 2. 영속 (Persistent / Managed)
    
    - 영속성 컨텍스트에 의해 관리되는 상태
    - `EntityManager.persist()` 호출 시 진입
    - 이후 변경 감지(Dirty Checking), 1차 캐시, 자동 플러시 등 기능 적용
    - 예시
        
        ```java
        em.persist(member); // 영속 상태 진입
        ```

    ### 3. 준영속 (Detached)
    
    - 한때 영속이었지만, 이제 컨텍스트에서 분리된 상태
    - `em.detach()`, `em.clear()`, `em.close()` 등 호출 시 발생
    - 변경 감지나 캐시 기능이 동작하지 않음
    - 예시
        
        ```java
        em.detach(member); // 컨텍스트에서 분리됨
        member.setName("변경됨"); // DB 반영 안 됨
        ```

    ### 4. 삭제 (Removed)
    
    - 영속성 컨텍스트에서 제거된 상태 (삭제 예약 상태)
    - `em.remove()` 호출 시 진입
    - flush 또는 commit 시 실제 DB에서 삭제
    - 예시
        
        ```java
        em.remove(member); // DB에서 삭제 예정
        ```

    ### 5. 상태 전이 요약표
    
    | 상태 | 설명 | 전이 메서드 / 이벤트 |
    | --- | --- | --- |
    | 비영속 | 아직 컨텍스트에 등록 안 됨 | `persist()` → 영속 |
    | 영속 | 컨텍스트가 관리 중 | `detach()`, `clear()`, `close()` → 준영속`remove()` → 삭제 |
    | 준영속 | 컨텍스트에서 분리됨 | `merge()` → 영속 복귀 |
    | 삭제 | 삭제 예약 상태 | flush/commit 시 DB 반영 |