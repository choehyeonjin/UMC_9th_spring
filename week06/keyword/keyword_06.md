- QueryDSL에서 FetchJoin 하는 법
    - 목적: 연관 엔티티를 한 번의 쿼리로 같이 로딩해서 N+1 문제를 없앰
    - 방법: `join(...).fetchJoin()`
    - 예제

        ```java
        QMember member = QMember.member;
        QTeam team = QTeam.team;
        
        List<Member> result = queryFactory
          .selectFrom(member)
          .join(member.team, team).fetchJoin()   // 또는 .leftJoin(...).fetchJoin()
          .where(member.name.eq("현진"))
          .fetch();
        ```

- DTO 매핑 방식 (+DTO안에 DTO)
    1. DTO (Data Transfer Object)란?
        - 계층 간 데이터를 전달하기 위한 순수 데이터용 객체.
        - 엔티티와 차이: 엔티티는 JPA가 관리(영속성, 지연로딩, 변경감지 등). DTO는 관리 대상 X, 조회/응답 전용, 필요한 필드만 담아 가볍게 전송.
    2. QueryDSL에서 DTO 매핑
        1. Projections.constructor() - 가장 간단하고 빠른 방법

            ```java
            // DTO
            @Getter
            @AllArgsConstructor
            public class MemberDto {
                private Long id;
                private String name;
                private String teamName;
            }
            
            // QuertDSL 코드
            QMember member = QMember.member;
            QTeam team = QTeam.team;
            
            List<MemberDto> result = queryFactory
                .select(Projections.constructor(MemberDto.class,
                    member.id,
                    member.name,
                    team.name
                ))
                .from(member)
                .leftJoin(member.team, team)
                .fetch();
                
            // 결과
            SELECT m.id, m.name, t.name
            FROM member m
            LEFT JOIN team t ON m.team_id = t.id
            ```

        2. @QueryProjection - 타입 안전하고 유지보수 강함
            - QueryDSL이 생성한 QDTO 클래스를 직접 select에 사용

            ```java
            // DTO
            import com.querydsl.core.annotations.QueryProjection;
            import lombok.Getter;
            
            @Getter
            public class MemberDto {
                private final Long id;
                private final String name;
                private final String teamName;
            
                @QueryProjection
                public MemberDto(Long id, String name, String teamName) {
                    this.id = id;
                    this.name = name;
                    this.teamName = teamName;
                }
            }
            // QueryDSL 코드
            QMember member = QMember.member;
            QTeam team = QTeam.team;
            
            List<MemberDto> result = queryFactory
                .select(new QMemberDto(
                    member.id,
                    member.name,
                    team.name
                ))
                .from(member)
                .leftJoin(member.team, team)
                .fetch();
            ```

    3. DTO 안에 DTO
        - 하나의 DTO 안에, 또 다른 DTO(다른 엔티티의 일부 정보)를 포함하는 구조
            - 예: `MemberDto` 안에 `TeamDto`, `OrderDto` 안에 `List<OrderItemDto>`
            - 엔티티 간 연관관계를 DTO 구조로 표현→ 복잡한 조인 결과를 깔끔하게 계층 구조로 응답할 수 있음
        1. Projections.constructor(TeamDto.class) - To-One 중첩 DTO
            - MemberDto → TeamDto

            ```java
            // DTO
            @Getter
            @AllArgsConstructor
            public class TeamDto {
                private Long id;
                private String name;
            }
            
            @Getter
            @AllArgsConstructor
            public class MemberDto {
                private Long id;
                private String name;
                private TeamDto team;   // ← 내부에 TeamDto 포함
            }
            // QueryDSL
            QMember member = QMember.member;
            QTeam team = QTeam.team;
            
            List<MemberDto> result = queryFactory
                .select(Projections.constructor(MemberDto.class,
                    member.id,
                    member.name,
                    Projections.constructor(TeamDto.class,   // ← 중첩 DTO 매핑
                        team.id,
                        team.name
                    )
                ))
                .from(member)
                .join(member.team, team)
                .fetch();
            ```

        2. GroupBy.transform() - To-Many 중첩 DTO
            - OrderDto → List<OrderItemDto>

            ```java
            // DTO
            @Getter
            @AllArgsConstructor
            public class OrderItemDto {
                private Long id;
                private String name;
                private int quantity;
            }
            
            @Getter
            @AllArgsConstructor
            public class OrderDto {
                private Long id;
                private String orderNumber;
                private List<OrderItemDto> items;  // ← 내부에 리스트로 포함
            }
            // QueryDSL
            import static com.querydsl.core.group.GroupBy.*;
            
            QOrder order = QOrder.order;
            QOrderItem item = QOrderItem.orderItem;
            
            Map<Long, OrderDto> resultMap = queryFactory
                .from(order)
                .leftJoin(order.items, item)
                .transform(groupBy(order.id).as(
                    Projections.constructor(OrderDto.class,
                        order.id,
                        order.orderNumber,
                        list(Projections.constructor(OrderItemDto.class,   // ← 리스트 DTO
                            item.id,
                            item.name,
                            item.quantity
                        ))
                    )
                ));
            
            List<OrderDto> result = new ArrayList<>(resultMap.values());
            // 결과
            OrderDto(
              id=1,
              orderNumber="A-1001",
              items=[
                OrderItemDto(id=1, name="키보드", qty=1),
                OrderItemDto(id=2, name="마우스", qty=2)
              ]
            )
            ```

- 커스텀 페이지네이션
    1. 페이지네이션이란?
        - 한 번에 많은 데이터를 불러오지 않고, 페이지 단위로 가져오는 기법

        ```sql
        SELECT * FROM member ORDER BY id LIMIT 10 OFFSET 0;   -- 1페이지
        SELECT * FROM member ORDER BY id LIMIT 10 OFFSET 10;  -- 2페이지
        SELECT * FROM member ORDER BY id LIMIT 10 OFFSET 20;  -- 3페이지
        ```

        - LIMIT: 몇 개 가져올지
        - OFFSET: 몇 개 건너뛸지
    2. JPA 기본 페이지네이션
        - Pageable 인터페이스와 그 구현체 PageRequest로 페이지 정보를 전달 → 자동으로 LIMIT, OFFSET, ORDER BY 생성

        ```java
        public interface MemberRepository extends JpaRepository<Member, Long> {
            Page<Member> findByAgeGreaterThan(int age, Pageable pageable);
        }
        
        PageRequest pageRequest = PageRequest.of(0, 10, Sort.by("name").descending());
        Page<Member> page = memberRepository.findByAgeGreaterThan(20, pageRequest);
        ```

    3. QueryDSL 커스텀 페이지네이션
        - JPA는 기본적으로 Pageable을 사용하면 자동으로 페이징 쿼리(limit, offset, count)를 만들어줌
        - 하지만 복잡한 동적 조건 / JOIN / DTO 매핑을 할 때는 이 자동 기능이 부족함
        - 그래서 QueryDSL을 이용해 직접 쿼리와 count를 작성하고, PageImpl로 감싸서 Page 형태로 반환하는 것이 QueryDSL 커스텀 페이지네이션

        ```java
        // 커스텀 인터페이스 정의
        public interface PostRepositoryCustom {
            Page<Post> findAllByCustomPaging(Pageable pageable);
        }
        
        // 구현 클래스 작성
        @RequiredArgsConstructor
        public class PostRepositoryImpl implements PostRepositoryCustom {
        
            private final JPAQueryFactory queryFactory;
        
            @Override
            public Page<Post> findAllByCustomPaging(Pageable pageable) {
        
                // 전체 개수 count 쿼리
                Long totalCount = queryFactory
                        .select(post.count())
                        .from(post)
                        .fetchOne();
        
                // 실제 페이지 데이터 조회
                List<Post> posts = queryFactory
                        .selectFrom(post)
                        .orderBy(post.id.desc())                       // 정렬 예시
                        .offset(pageable.getOffset())                  // 몇 번째부터
                        .limit(pageable.getPageSize())                 // 몇 개 가져올지
                        .fetch();
        
                // PageImpl 로 감싸서 반환
                return new PageImpl<>(posts, pageable, totalCount == null ? 0 : totalCount);
            }
        }
        
        // Repository에서 확장
        public interface PostRepository
                extends JpaRepository<Post, Long>, PostRepositoryCustom {
        }
        ```

- transform - groupBy
    - 개념
        - 그룹화된 데이터를 한 번의 쿼리로 계층 구조로 묶어주는 기능
        - SQL의 GROUP BY 개념을 자바 객체 형태로 변환해주는 도구
    - 예시

        ```java
        Map<Long, List<String>> result = queryFactory
            .from(order)
            .leftJoin(order.items, item)
            .transform(
                groupBy(order.id).as( // 묶을 기준 (부모의 키)
                    list(item.name) // 묶일 데이터 (자식들)
                )
            );
        ```

- order by null
    - 개념
        - DB가 정렬 연산을 하지 않아 결과를 임의 순서로 반환하게 만듦
        - 성능 최적화에 사용
        - ORDER BY 생략: “정렬 조건이 없다” (DB가 알아서 판단)
        - ORDER BY NULL: “정렬하지 말라” (DB에 확실히 명령)
    - 예시

        ```sql
        SELECT * FROM member ORDER BY NULL;
        ```

        ```java
        import com.querydsl.core.types.dsl.Expressions;
        
        List<Member> members = queryFactory
            .selectFrom(member)
            .orderBy(Expressions.nullExpression()) // ORDER BY NULL
            .fetch();
        ```

    - transform - groupby에서 사용
        - groupby 변환은 내부적으로 메모리에서 정렬하므로, DB에서의 정렬 불필요

        ```java
        Map<Long, OrderDto> map = queryFactory
            .from(order)
            .leftJoin(order.items, item)
            .orderBy(Expressions.nullExpression())   // ORDER BY NULL
            .transform(groupBy(order.id).as(
                list(Projections.constructor(OrderItemDto.class, item.id, item.name))
            ));
        ```