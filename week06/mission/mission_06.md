깃허브 링크: [UMC-9th-spring-study](https://github.com/choehyeonjin/UMC-9th-spring-study/tree/Feat/Chapter6)

1. QueryDSL 설치 과정 인증하기

   ![build.gradle 파일 수정](./mission_06_(1).png)

   build.gradle 파일 수정

   ![생성된 Q클래스](./mission_06_(2).png)

   생성된 Q클래스

2. 내가 작성한 리뷰 보기 API, QueryDSL로 구현하기
   필터링 조건: 가게별(반이학생마라탕마라반), 별점별 (5점, 4점대, 3점대 …)
   제약 조건: 하나의 API로 설계

    1. “내가 작성한” 조건(= memberId)은 항상 고정 필터로 두고,
        거기에 가게명(storeName) 과 별점대(ratingBand)를 선택하여 필터링하는 동적 쿼리로 하나의 API를 설계하고자 함

    1. 구현 및 수정 코드

       ![image.png](./mission_06_(3).png)

    2. 테스트를 위해 DB에 시드 데이터 추가

        ```sql
        INSERT INTO region (id, name)
        VALUES (1, '서울')
        
        INSERT INTO store
        (id, region_id, name, address, identify_number)
        VALUES
        (10, 1, '반이학생마라탕마라반', '서교동', '123-45-67890'),
        (20, 1, '장사부마라탕', '개봉동', '999-999')
        
        INSERT INTO review
        (member_id, store_id, rating, content, created_at)
        VALUES
        (1, 10, 5.0, '최고예요', NOW()),
        (1, 10, 4.3, '맛있어요', NOW()),
        (1, 10, 3.8, '나쁘지 않아요', NOW()),
        (1, 20, 4.1, '맛도리', NOW()),
        (2, 10, 3.8, '괜찮아요', NOW());
        ```

    3. Postman API 테스트 결과 확인

       ![내가 작성한 리뷰 (가게이름 필터)](./mission_06_(4).png)

       내가 작성한 리뷰 (가게이름 필터)

       ![내가 작성한 리뷰 (4점대 필터)](./mission_06_(5).png)

       내가 작성한 리뷰 (4점대 필터)

       ![내가 작성한 리뷰 (가게이름 + 5점 필터링)](./mission_06_(6).png)

       내가 작성한 리뷰 (가게이름 + 5점 필터링)