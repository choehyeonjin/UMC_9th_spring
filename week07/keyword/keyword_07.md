- RestContollerAdvice
    - 역할: 컨트롤러 전역에서 발생하는 예외를 공통 포맷 JSON으로 응답
    - 구성: 클래스 레벨에 `@RestControllerAdvice`, 메서드 레벨에 `@ExceptionHandler`
    - 장점:
        - 하나의 클래스로 모든 컨트롤러에 대해 전역적으로 예외 처리가 가능함
        - 중복 제거(try-catch문 삭제) → 가독성 및 유지보수성 향상
        - 직접 정의한 에러 응답을 일관성있게 클라이언트에게 내려줄 수 있음
    - 구성:
        1. 도메인 서비스에서 도메인 예외(`TestException`) 발생

        ```java
        // domain: test
        @Override
        public void checkFlag(Long flag){
            if (flag == 1){
                throw new TestException(TestErrorCode.TEST_EXCEPTION);
            }
        ```

        2. `TestException`은 `GeneralException`를 상속

        ```java
        // global
        @Getter
        @AllArgsConstructor
        public class GeneralException extends RuntimeException {
            private final BaseErrorCode code;
        }
        
        // domain
        public class TestException extends GeneralException {
            public TestException(BaseErrorCode code) { super(code); }
        }
        ```

        3. 전역 핸들러가 `GeneralException`을 한 번에 처리
        - `TestException` 핸들러가 있으면 그게 먼저, 없으면 `GeneralException`, 그래도 없으면 `Exception`

        ```java
        @RestControllerAdvice
        public class GeneralExceptionAdvice {
        
            @ExceptionHandler(GeneralException.class)
            public ResponseEntity<ApiResponse<Void>> handleException(GeneralException ex) {
                return ResponseEntity.status(ex.getCode().getStatus())
                        .body(ApiResponse.onFailure(ex.getCode(), null));
            }
        
            @ExceptionHandler(Exception.class)
            public ResponseEntity<ApiResponse<String>> handleException(Exception ex) {
                BaseErrorCode code = GeneralErrorCode.INTERNAL_SERVER_ERROR;
                return ResponseEntity.status(code.getStatus())
                        .body(ApiResponse.onFailure(code, ex.getMessage()));
            }
        }
        ```

        4. 컨트롤러는 성공 로직만 작성

        ```java
        @RestController
        @RequiredArgsConstructor
        @RequestMapping("/temp")
        public class TestController {
        
            private final TestQueryService testQueryService;
        
            @GetMapping("/exception")
            public ApiResponse<TestResDTO.Exception> exception(@RequestParam Long flag) {
                testQueryService.checkFlag(flag); // 예외 처리
                return ApiResponse.onSuccess(GeneralSuccessCode.OK,
                        TestConverter.toExceptionDTO("This is Test!"));
            }
        }
        ```

- lombok
    - 개념: 반복되는 보일러플레이트 코드를 자동으로 생성해주는 라이브러리
    - 동작 방식: 컴파일 시점에 annotation processor가 작동하여 바이트코드에 메서드 삽입
    - lombok 주요 어노테이션

        | 어노테이션 | 자동 생성되는 함수 | 주요 용도 | 비고 |
        | --- | --- | --- | --- |
        | **@Getter** | 모든 필드의 `getXxx()` | 읽기 전용 접근자 자동 생성 | 클래스나 필드 단위 가능 |
        | **@Setter** | 모든 필드의 `setXxx()` | 쓰기 전용 접근자 자동 생성 | 엔티티에서는 조심해서 사용 |
        | **@ToString** | `toString()` | 객체를 문자열로 출력 | `exclude = {"필드명"}`로 제외 가능 |
        | **@EqualsAndHashCode** | `equals()`, `hashCode()` | 객체 동등성 비교 | JPA 엔티티에서는 ID만 비교하도록 제한 권장 |
        | **@NoArgsConstructor** | 기본 생성자 | JPA 엔티티 등에서 필수 | 접근 제한자 지정 가능 (`access = PROTECTED`) |
        | **@AllArgsConstructor** | 모든 필드 포함한 생성자 | DTO, 테스트용 객체 | - |
        | **@RequiredArgsConstructor** | `final` 또는 `@NonNull` 필드만 포함한 생성자 | `@Service`, `@Controller`에서 의존성 주입용 | `@NonNull` 필드만 포함 |
        | **@Builder** | 빌더 패턴(`.builder()`) | DTO, 엔티티 생성 시 가독성 향상 | `@Builder.Default`로 기본값 설정 가능 |
        | **@Data** | `@Getter + @Setter + @ToString + @EqualsAndHashCode + @RequiredArgsConstructor` | DTO | 엔티티에는 비추천 |
    
  - 예시 동작

    ```java
    import lombok.*;
    
    @Getter
    @Setter
    @ToString
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public class Member {
    private Long id;
    private String name;
    }
    ```        
        
    → 컴파일 후 생성되는 메서드
        
    ```java
    public class Member {
        private Long id;
        private String name;
        
        public Long getId() { return id; }
        public void setId(Long id) { this.id = id; }
        
        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        
        public String toString() { return "Member(id=" + id + ", name=" + name + ")"; }
        
        public Member() {}
        public Member(Long id, String name) {
            this.id = id;
            this.name = name;
        }
        
        public static MemberBuilder builder() { return new MemberBuilder(); }
        // builder 내부 클래스 자동 생성
        }
    ```
        
    - @Builder
        - 사용 이유
            - 일반 생성자 방식
                
                ```java
                Member m = new Member("홍길동", 25, "010-1234-5678", "SEOUL");
                ```
                
                - 어떤 값이 어떤 필드인지 가독성 낮음
            - 빌더 방식
                
                ```java
                Member m = Member.builder()
                        .name("홍길동")
                        .age(25)
                        .phone("010-1234-5678")
                        .address("SEOUL")
                        .build();
                ```
                
                - 순서 상관 없음
                - 필드명으로 값이 어떤 의미인지 명확함
                - 선택적 파라미터 처리 용이 (null 허용 필드 생략 가능)
                - 실제로는 내부 static 클래스가 자동 생성됨
                
                ```java
                public static class MemberBuilder {
                    private String name;
                    private int age;
                
                    public MemberBuilder name(String name) { this.name = name; return this; }
                    public MemberBuilder age(int age) { this.age = age; return this; }
                
                    public Member build() {
                        return new Member(name, age);
                    }
                }
                ```

- dto 형식 public static VS record 비교하기
    - 표 비교

  | 항목 | inner static DTO (lombok 조합)                                                                                | `record` DTO |
      | --- |-------------------------------------------------------------------------------------------------------------| --- |
  | 의미 | 한 도메인의 여러 응답/요청 스키마를 한 파일/네임스페이스로 묶음                                                                        | 순수 데이터 캐리어(불변)를 간결하게 표현 |
  | 파일 구조 | `TestResDTO.Testing`, `MemberResDTO.Summary`처럼 한 클래스 안에 public static으로 다수 정의 → 파일 수 ↓                      | 보통 레코드별 파일 1개 → 파일 수 ↑ (패키지로 그룹핑) |
  | 가변성 | Lombok 조합에 따라 가변/불변 선택 가능 (`@Getter` + no `@Setter` = 사실상 불변)                                               | 기본 불변 (모든 컴포넌트 final 성격) |
  | 빌더 지원 | `@Builder` 로 바로 지원 → 선택 파라미터/대형 DTO에 유리                                                                     | 기본 미제공. 필요하면 별도 빌더 구현 or 다른 패턴 사용 |
  | 기본값 처리 | `@Builder.Default`로 매우 간단                                                                                   | 생성자에서 직접 지정해야 함 |
  | 가독성 | “도메인별/응답별 묶음”을 한 파일에서 한눈에 보기 좋음                                                        , 다만 파일 커짐→ PR 충돌 가능 | 파일은 늘어나지만 각 DTO가 독립 → 충돌↓, 이식/재사용↑                                                     |
  | 요청 vs 응답 | 응답: `@Builder`로 생성 가독성↑, 요청: 빌더 불필요                                                                         | 요청/응답 모두 가능하나 응답(불변)에 더 자연스러움 |
    - public static DTO
        - 사용하는 상황
            - 필드가 많고 선택적 파라미터가 많은 대형 DTO (옵션/기본값/중첩 구조 등…)
            - Service에서 여러 도메인 데이터 합쳐 Response로 만드는 경우
            - 테스트 코드에서 필드를 다양하게 조합할 때
        - 워크북에서 사용한 예시

            ```java
            public class TestResDTO {
            
                @Builder
                @Getter
                public static class Testing {
                    private String testing;
                }
            }
            ```

    - record
        - 사용하는 상황
            - 단순 요청/응답 DTO
                - 요청: 프론트엔드 → 백엔드로 전달되는 JSON 바디를 그대로 받는 객체
            - 필드 수가 적고, 빌더까지 필요 없음
            - 조회용 api
                - 쿼리 결과를 그대로 받는 용도
                - Setter나 빌더 필요 없음
        - 예시

            ```java
            public record TestTesting(String testing) { }
            ```