> **github 링크**: 
> [Feat/Chapter8](https://github.com/choehyeonjin/UMC-9th-spring-study/commits/Feat/Chapter9)

1. 가게의 리뷰 목록 조회 API (워크북 본문)
    - 리뷰에 사용자 본명 말고 닉네임도 필요해 member 테이블에 nickname 컬럼을 추가했다.
    - Swagger 결과

   ![image.png](./mission_09(1).png)

2. 내가 작성한 리뷰 목록 API
    - 9주차 워크북 본문에서 실습한 내용이랑, 6주차 미션에서 “내가 작성한 리뷰 보기 API, QueryDSL로 구현하기” 내용이 겹쳤다. 화면은 똑같고 필터링만 다르며, 6주차 때 구현한 DTO가 화면 구성에 더 맞는 것 같아 미션 기록 1번 부분을 공통 리뷰 목록 조회 DTO, Converter로 리팩토링하였다.

        ```java
        @Getter
        @NoArgsConstructor(access = AccessLevel.PROTECTED)
        @AllArgsConstructor
        public class ReviewResDTO {
        
            // 단건 조회용
            private Long reviewId;
            private String storeName;
            private Float rating;
            private String content;
            private LocalDateTime createdAt;
            private String replyContent;
        
            // 공통 리뷰 목록 응답 DTO
            @Builder
            public record ReviewListDTO(
                    List<ReviewItemDTO> reviewList,
                    Integer listSize,
                    Integer totalPage,
                    Long totalElements,
                    Boolean isFirst,
                    Boolean isLast
            ) {}
        
            @Builder
            public record ReviewItemDTO(
                    Long reviewId,
                    String storeName,
                    Float rating,
                    String content,
                    LocalDateTime createdAt,
                    String replyContent
            ) {}
        }
        ```

    - Swagger 결과

      ![image.png](./mission_09(2).png)

3. 특정 가게의 미션 목록 API
    - Swagger 결과

   ![image.png](./mission_09(3).png)

4. 내가 진행 중인 미션 목록 API
    - Swagger 결과

   ![image.png](./mission_09(4).png)


5. 커스텀 어노테이션

- page를 쿼리 스트링으로 받음 → 커스텀 어노테이션 `@ValidPage`을 설정해 `PageValidator`가 내부에서 1 이상인지 검사 → 실패 시 INVALID_PAGE (COMMON400_3) 응답

    ```java
    // ReviewQueryController
    @GetMapping("/reviews")
    public ApiResponse<...> getReviews(
            @RequestParam String storeName,
            @RequestParam(defaultValue = "1") @ValidPage Integer page
    ) { }
    
    // PageValidator
    public class PageValidator implements ConstraintValidator<ValidPage, Integer> {
    
        @Override
        public boolean isValid(Integer value, ConstraintValidatorContext context) {
    
            boolean isValid = value != null && value >= 1;
    
            if (!isValid) {
                // 기본 메시지 제거
                context.disableDefaultConstraintViolation();
                // GeneralErrorCode의 메시지로 덮어쓰기
                context.buildConstraintViolationWithTemplate(
                                GeneralErrorCode.INVALID_PAGE.getMessage()
                        )
                        .addConstraintViolation();
            }
    
            return isValid;
        }
    }
    
    // GeneralErrorCode
    public enum GeneralErrorCode implements BaseErrorCode{
    
        ...
        INVALID_PAGE(HttpStatus.BAD_REQUEST,
                "COMMON400_3",
                "page는 1 이상이어야 합니다."),
        ...
    }
    
    // GeneralExceptionAdvice
    
        // 메서드 파라미터 검증
        @ExceptionHandler(jakarta.validation.ConstraintViolationException.class)
        public ResponseEntity<ApiResponse<String>> handleConstraintViolationException(
                jakarta.validation.ConstraintViolationException ex
        ) {
            String message = ex.getConstraintViolations()
                    .iterator()
                    .next()
                    .getMessage();
    
            // 기본: 일반 유효성 검증 실패 (VALID_FAIL)
            GeneralErrorCode code = GeneralErrorCode.VALID_FAIL;
    
            // @ValidPage: INVALID_PAGE
            if (message.equals(GeneralErrorCode.INVALID_PAGE.getMessage())) {
                code = GeneralErrorCode.INVALID_PAGE;
            }
            return ResponseEntity.status(code.getStatus())
                    .body(ApiResponse.onFailure(code, message));
        }
    ```

- Swagger 결과
    - page가 0일 때

  ![image.png](./mission_09(5).png)