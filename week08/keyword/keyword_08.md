- java의 Exception 종류들
    1. **Checked Exception**

  컴파일러가 반드시 처리(try-catch 또는 throws)하라고 강제하는 예외 → 주로 외부 환경(IO, 네트워크, DB) 때문에 발생.

    - 특징
        - 프로그램 실행 전에 컴파일 단계에서 확인됨
        - "예상 가능한 문제"이기 때문에 복구 가능한 경우가 많음
        - 반드시 예외 처리를 해야 컴파일 가능
    - 종류

        | 예외 | 설명 |
        | --- | --- |
        | IOException | 파일/네트워크 입출력 문제 |
        | FileNotFoundException | 지정 파일이 없음 |
        | SQLException | DB 쿼리 실행/연결 문제 |
        | ClassNotFoundException | 클래스를 찾을 수 없음 |
        | InterruptedException | 스레드가 인터럽트됨 |
        | ParseException | 날짜/숫자 파싱 실패 |
    - 예시
        
        ```java
        try {
            FileInputStream fis = new FileInputStream("a.txt");
        } catch (IOException e) {
            e.printStackTrace();
        }
        ```
    
    **2. Unchecked Exception (RuntimeException)**
    
    컴파일 시점에 검사하지 않는 예외 → 런타임에 갑자기 터짐 → 대부분 개발자의 실수로 인해 발생하는 논리 오류.
    
    - 특징
        - 예외 처리 강제 없음
        - Null, 인덱스 문제, 잘못된 파라미터 등 버그성 오류
        - 복구 불가능하거나 굳이 강제할 필요 없음
    - 종류
        
        | 예외 | 설명 |
        | --- | --- |
        | **NullPointerException** | null 접근 |
        | **ArrayIndexOutOfBoundsException** | 배열 인덱스 초과 |
        | **IllegalArgumentException** | 잘못된 매개변수 전달 |
        | **IllegalStateException** | 객체 상태가 잘못됨 |
        | **ArithmeticException** | 0으로 나누기 |
        | **ClassCastException** | 잘못된 형변환 |
        | **NumberFormatException** | 숫자로 변환 실패 (Integer.parseInt 등) |
    - 예시
    ```java
    String s = null;
    System.out.println(s.length()); // NPE
    ```
    
  3. 어떤 상황에 어떤 예외를 쓰나?
    - Checked Exception 쓰는 경우
        - 외부 요인 때문에 실패할 가능성이 있고
            
            개발자가 그 실패를 "어떤 방식으로든 대응"해야 하는 경우
            
            → IO, DB, 파일, 네트워크
            
    - Unchecked Exception 쓰는 경우
        - 개발자가 잘못된 값을 넘기거나
        - 상태를 잘못 사용해서 발생하는 논리 오류
        - 잘못된 코드 흐름에 대해 강제적으로 try-catch 시킬 필요 없음

---

- @Valid

  ## 1. 개념

  `@Valid`는 객체에 선언된 Bean Validation 규칙을 실행하라는 트리거 역할을 하는 어노테이션이다.

  DTO의 각 필드에 붙어 있는 `@NotBlank`, `@Size`, `@Email` 등의 검증 규칙을 기반으로 값의 유효성을 검사한다.

  ## 2. 동작 조건

  스프링 부트에서는 다음 의존성을 포함하면 자동으로 Bean Validation이 동작한다.

    ```
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    ```

  이 의존성 안에는 Bean Validation API와 Hibernate Validator가 포함되어 있어 별도 설정 없이 `@Valid`가 동작한다.

  ## 3. @Valid의 사용 위치

  ### 3-1. 컨트롤러 파라미터 DTO

  요청 본문(`@RequestBody`)이나 요청 파라미터(`@ModelAttribute`)로 DTO를 받을 때 `@Valid`를 함께 사용한다.

    ```java
    @PostMapping("/members")
    public ResponseEntity<?> createMember(@Valid @RequestBody MemberRequest request) {
        ...
    }
    ```

    - JSON 바디 바인딩 후 검증 실패 시 `MethodArgumentNotValidException`이 발생한다.
    - `@ModelAttribute` 기반 요청이면 `BindException`이 발생한다.

    ---

  ### 3-2. 중첩 객체(Nested DTO) 검증

  DTO 내부에 또 다른 DTO가 있을 경우, 내부 DTO 필드에 `@Valid`를 붙여야 내부 필드의 검증도 함께 수행된다.

    ```java
    public class OrderRequest {
    
        @NotNull
        private Long memberId;
    
        @Valid
        private Address address;
    }
    ```
    
  ---

  ## 4. 검증 실패 시 처리 흐름

  ### 4-1. BindingResult 없이 @Valid 사용

  컨트롤러 파라미터에 `BindingResult`를 받지 않을 경우, 검증 실패 시 스프링이 곧바로 예외를 던진다.

    - `@RequestBody` → `MethodArgumentNotValidException`
    - `@ModelAttribute` → `BindException`

  이 예외는 보통 `@RestControllerAdvice`에서 공통으로 처리한다.
    
  ---

  ### 4-2. BindingResult와 함께 사용

  검증 대상 파라미터 **바로 뒤에** `BindingResult`를 선언하면 예외가 발생하지 않고 컨트롤러 내부에서 검증 결과를 직접 확인할 수 있다.

    ```java
    @PostMapping("/members")
    public String create(@Valid MemberForm form, BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            return "members/new-form";
        }
        ...
    }
    ```

  `BindingResult`는 필드 에러, 객체 에러를 포함하며 `hasErrors()`로 간단하게 검증 실패 여부를 알 수 있다.
    
  ---

  ## 5. 에러 메시지 처리

  검증 어노테이션에 message 값을 직접 작성하거나, 메시지 파일(`messages.properties`)을 통해 관리할 수 있다.

    ```java
    @NotBlank(message = "{member.name.notBlank}")
    private String name;
    ```

    ```
    member.name.notBlank=이름은 필수 입력값이다.
    ```

  Spring의 MessageSource가 코드 → 메시지로 변환해준다.
    
  ---

  ## 6. @Valid vs @Validated

  ### @Valid

    - Bean Validation 표준 어노테이션이다.
    - 객체 검증만 수행하며 **검증 그룹(group)** 기능은 없다.

  ### @Validated

    - 스프링이 제공하는 확장 어노테이션이다.
    - 그룹 검증을 지원한다.
    - 특정 상황(Create, Update 등)에 따라 다른 검증 규칙을 적용할 때 사용한다.

    ```java
    @PostMapping("/members")
    public ResponseEntity<?> create(@Validated(Create.class) @RequestBody MemberRequest req) {
        ...
    }
    ```
    
  ---

  ## 7. 요약

    - `@Valid`는 DTO의 유효성 검증을 트리거하는 어노테이션이다.
    - 검증 규칙은 Bean Validation 어노테이션(@NotBlank 등)을 통해 정의한다.
    - 검증 실패 시 예외가 발생하거나, BindingResult로 오류를 직접 처리할 수 있다.
    - 중첩 객체에는 필드에도 `@Valid`를 붙여야 내부 DTO까지 검증이 수행된다.
    - 그룹 검증이 필요하면 `@Validated`를 사용한다.