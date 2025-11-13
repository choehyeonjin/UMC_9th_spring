1. 깃허브 링크: [UMC-9th-spring-study](https://github.com/choehyeonjin/UMC-9th-spring-study/tree/Feat/Chapter7)
2. RestControllerAdvice의 장점, 그리고 없을 경우 어떤 점이 불편한지
    - 장점:
        - 하나의 클래스로 모든 컨트롤러에 대해 전역적으로 예외 처리가 가능함
        - 중복 제거(try-catch문 삭제) → 가독성 및 유지보수성 향상
        - 직접 정의한 에러 응답을 일관성있게 클라이언트에게 내려줄 수 있음
    - 없으면 생기는 불편:
        - 컨트롤러마다 try-catch문 작성
        - 일관되지 않은 상태코드/메시지 → QA/프론트 작업시 혼란
3. 지금까지 진행하면서 작성한 API들 모두 응답통일 처리하기
    1. GET /temp/test

       ![image.png](./mission_07_(1).png)

    2. GET /temp/exception

       ![image.png](./mission_07_(2).png)

       ![image.png](./mission_07_(3).png)

    3. GET /members/{memberId}/reviews
        - 6주차 미션 때 구현한 “내가 작성한 리뷰 필터링해서 조회” API에 응답 통일 기능을 추가했다.
        - 기존 ReviewQueryController
            - List<ReviewResponse>만 바로 반환하였음.

            ```java
            @RestController
            @RequiredArgsConstructor
            @RequestMapping("/members/{memberId}/reviews")
            public class ReviewQueryController {
            
                private final ReviewQueryService reviewQueryService;
            
                @GetMapping
                public List<ReviewResponse> myReviews(
                        @PathVariable Long memberId,
                        @RequestParam(required = false) String storeName,
                        @RequestParam(required = false) Integer ratingBand
                ) {
                    return reviewQueryService.getMyReviews(memberId, storeName, ratingBand);
                }
            }
            ```

        - 수정한 ReviewQueryController
            - List<ReviewResponse>를 ApiResponse<T>로 감싸서 반환.

                ```java
                @RestController
                @RequiredArgsConstructor
                @RequestMapping("/members/{memberId}/reviews")
                public class ReviewQueryController {
                
                    private final ReviewQueryService reviewQueryService;
                
                    @GetMapping
                    public ApiResponse<List<ReviewResponse>> myReviews(
                            @PathVariable Long memberId,
                            @RequestParam(required = false) String storeName,
                            @RequestParam(required = false) Integer ratingBand
                    ) {
                        // 응답 코드 정의
                        GeneralSuccessCode code = GeneralSuccessCode.OK;
                
                        // 응답 데이터 정의
                        List<ReviewResponse> data = reviewQueryService.getMyReviews(memberId, storeName, ratingBand);
                
                        // code+message, result 응답
                        return ApiResponse.onSuccess(code, data);
                    }
                }
                ```

        - 테스트 결과

          ![image.png](./mission_07_(4).png)

4. 성공 메서드, 성공 Enum 제작하기

   ![image.png](./mission_07_(5).png)

    - global/apiPayload에 성공한 경우 200,201,204 상태에 대해 api 응답을 구현하였다.

    ```java
    // BaseSuccessCode
    public interface BaseSuccessCode {
    
        HttpStatus getStatus();
        String getCode();
        String getMessage();
    }
    
    // GeneralSuccessCode
    @Getter
    @AllArgsConstructor
    public enum GeneralSuccessCode implements BaseSuccessCode{
    
        OK(HttpStatus.OK,
                "COMMON200_1",
                "성공적으로 요청을 처리했습니다."),
        CREATED(HttpStatus.CREATED,
                "COMMON201_1",
                "리소스가 성공적으로 생성되었습니다."),
        NO_CONTENT(HttpStatus.NO_CONTENT,
                "COMMON204_1",
                "성공했지만 반환할 데이터가 없습니다."),
        ;
    
        private final HttpStatus status;
        private final String code;
        private final String message;
    }
    
    // ApiResponse
    @Getter
    @AllArgsConstructor
    @JsonPropertyOrder({"isSuccess", "code", "message", "result"})
    public class ApiResponse<T> {
    
        @JsonProperty("isSuccess")
        private final Boolean isSuccess;
    
        @JsonProperty("code")
        private final String code;
    
        @JsonProperty("message")
        private final String message;
    
        @JsonProperty("result")
        private T result;
    
        // 성공한 경우 (result 포함)
        public static <T> ApiResponse<T> onSuccess(BaseSuccessCode code, T result) {
            return new ApiResponse<>(true, code.getCode(), code.getMessage(), result);
        }
    
        // 실패한 경우 (result 포함)
        public static <T> ApiResponse<T> onFailure(BaseErrorCode code, T result) {
            return new ApiResponse<>(false, code.getCode(), code.getMessage(), result);
        }
    }
    ```
