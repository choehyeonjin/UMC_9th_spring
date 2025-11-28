- 객체 그래프 탐색

  ### 1. 객체 그래프(Object Graph)란?

  프로그램에서 객체들이 서로를 참조하며 형성하는 구조 전체를 말함.

  예시:

    ```java
    class Member {
        Profile profile;
        List<Order> orders;
    }
    
    class Order {
        Delivery delivery;
    }
    ```

  `Member → orders → Order → delivery`

  이렇게 연결된 모든 객체들의 네트워크가 객체 그래프.
    
  ---

  ### 2. 객체 그래프 탐색(Object Graph Traversal)

  객체 그래프 탐색은 최초 객체에서 시작해 참조를 타고 들어가며 필요한 데이터를 ‘깊이’ 탐색하는 과정이다.

  예:

    ```java
    Member member = findMember();
    Delivery d = member.getOrders().get(0).getDelivery();
    ```

  → Member에서 Orders로

  → Orders에서 Order로

  → Order에서 Delivery로

  이렇게 객체를 “따라 들어가는” 것이 그래프 탐색.
    
  ---

  ### 3 . JPA에서의 객체 그래프 탐색 핵심 개념

    - 프록시 초기화
        - `member.getOrders()` 호출 시 Hibernate는 **프록시를 통해 필요할 때 로딩**함.
    - FetchType.LAZY / EAGER
        - **LAZY**: 접근하는 순간 쿼리 실행
        - **EAGER**: Member 로딩 시 Order도 같이 로딩

      → 결국 탐색 과정이 *쿼리 실행 흐름*에 직접 영향.

    - N+1 문제

      객체 그래프를 잘못 순회하면 반복적으로 DB 조회 발생.
  
  ### 4. 한 줄 요약
  JPA에서 객체 그래프 탐색이란, 엔티티 간 연관관계를 통해 객체 필드를 따라가면서 데이터를 접근하는 것이며, 이 과정에서 로딩 전략(LAZY/EAGER)에 따라 SQL 쿼리가 실행되고 이는 성능, 순환 참조(무한 루프), N+1 문제 등 다양한 결과를 낳는다.
    

- RESTful API

  ## 1. RESTful API

  REST(Representational State Transfer) 는 웹 API 설계 스타일이고, 핵심은:

    1. 리소스(Resource) 중심
        - “동사(행위)”가 아니라 “명사(대상)” 위주 설계
        - 예:
            - ❌ `/createReview`
            - ✅ `POST /reviews` (리뷰라는 리소스에 ‘행위’를 HTTP Method로 표현)
    2. HTTP Method로 의도 표현
        - `GET` : 조회
        - `POST` : 생성
        - `PUT` : 전체 수정
        - `PATCH` : 부분 수정
        - `DELETE`: 삭제
    3. URI에는 ‘리소스’만, 행위는 넣지 말기
        - `/stores/{id}/reviews` ✅
        - `/getStoreReviews` ❌
        - `/stores/{id}/reviews/listAll` ❌
    4. Stateless
        - 서버는 클라이언트 상태를 세션으로 들고 있지 않고,

          각 요청이 필요한 정보를 다 들고 오게 설계.

    ## 2. RESTful URI 설계 기본 원칙

    ### 2-1. 명사, 복수형
    
    - 보통 컬렉션은 복수형으로:
        - `/stores`
        - `/reviews`
        - `/members`
    
    ### 2-2. 계층 구조 (계층 관계가 분명할 때만)
    
    - A에 종속된 B가 명확하면 중첩 리소스:
        - `/stores/{storeId}/reviews`
        - `/members/{memberId}/orders`
    - 반대로, 리소스가 독립적이면 최상단으로:
        - `/reviews/{reviewId}`
        - `/members/{memberId}`
    
    ### 2-3. 필터링/검색은 쿼리 스트링
    
    - `/reviews?storeId=1&rating=5&sort=latest`
    
    ## 3. 두 가지 URI 비교
    
    > 가게의 리뷰 목록 조회 API
    > 
    
    ### 1) `GET /reviews`
    
    “리뷰 전체 컬렉션” 에 대한 엔드포인트.
    
    - 의미: “시스템에 있는 리뷰 전체를 조회하거나, 조건으로 필터링”
    - 조건 넣고 싶으면:
        - `GET /reviews?storeId=1`
        - `GET /reviews?memberId=3`
        - `GET /reviews?storeId=1&rating=5`
    
    ✔ 장점
    
    - 리뷰라는 리소스를 최상위 객체로 취급 (독립성↑)
    - 나중에 “유저가 쓴 모든 리뷰 조회” 같은 것도 같은 엔드포인트에서 해결 가능
    - 필터 조합이 많을수록 유연함
    
    ✖ 단점
    
    - “가게라는 컨텍스트 하의 리뷰”가 URI에서 바로 안 보임
        
        (`/stores/{storeId}/reviews`에 비해 의미가 덜 직관적)

    ---
    
    ### 2) `GET /stores/{storeId}/reviews`
    
    이건 “특정 가게에 소속된 리뷰들” 이라는 의미가 아주 명확한 중첩 리소스 패턴.
    
    - 의미: “이 가게의 리뷰 컬렉션”
    - 예:
        - `GET /stores/1/reviews` → 1번 가게의 리뷰 목록
    - 보통 같이 쓰는 패턴:
        - `POST /stores/{storeId}/reviews` → 이 가게에 리뷰 하나 작성
        - `GET /stores/{storeId}/reviews/{reviewId}` → 필요하면 특정 리뷰에 접근
    
    ✔ 장점
    
    - 비즈니스 관점에서 “가게-리뷰” 관계가 URI에 드러남
    - 프론트 코드를 볼 때도:
        
        `/stores/1/reviews` ← “아 이건 1번 가게에 대한 리뷰들” 바로 이해 가능
        
    - 권한/검증에서도 직관적:
        - 예: `storeId`가 path에 항상 있으니까, 그 가게가 존재하는지 체크 후 리뷰 로직 처리.
    
    ✖ 단점
    
    - 리뷰를 “전역적으로 다루는” API (예: 관리자 페이지)에서는
        
        또 다른 엔드포인트(`/reviews`)를 만들어야 할 수도 있음

    ---

    ### 3) 선택 팁
    
    단순히 “RESTful이냐 / 아니냐” 문제보다는,
    
    > 리뷰라는 리소스를 “가게에 종속된 애”로 보느냐,
    아니면 “독립된 1급 리소스”로 보느냐 의 차이.
    > 
    
    실전에서는 둘 다 같이 쓰는 경우가 많음:
    
    1. 전역 컬렉션
        - `GET /reviews`
            - admin이 전체 리뷰 관리할 때
            - 필터: `?storeId=1&memberId=3&rating=4`
    2. 가게 하위 리소스
        - `GET /stores/{storeId}/reviews`
            - 유저가 “특정 가게 상세” 들어갔을 때 보여줄 리뷰 목록
        - `POST /stores/{storeId}/reviews`
            - 가게 상세 페이지에서 “리뷰 작성하기” 눌렀을 때