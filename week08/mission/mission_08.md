1. 깃허브 링크: [Feat/Chapter8](https://github.com/choehyeonjin/UMC-9th-spring-study/tree/Feat/Chapter8)
2. 회원가입 API
    - food ENUM을 와이어프레임에 맞게 수정하였다.
    - 이번 구현에 사용하지 않는 member 컬럼 email, phone, social_uid, social_type을 NULL로 수정하였다.

        ```sql
        ALTER TABLE food 
        MODIFY COLUMN name ENUM(
            '한식',
            '일식',
            '중식',
            '양식',
            '치킨',
            '분식',
            '고기구이',
            '도시락',
            '야식',
            '패스트푸드',
            '디저트',
            '아시안푸드'
        ) 
        CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
        
        INSERT INTO food (name) VALUES
        ('한식'),
        ('일식'),
        ('중식'),
        ('양식'),
        ('치킨'),
        ('분식'),
        ('고기구이'),
        ('도시락'),
        ('야식'),
        ('패스트푸드'),
        ('디저트'),
        ('아시안푸드');
        
        ALTER TABLE member 
            MODIFY COLUMN email VARCHAR(255) NULL,
            MODIFY COLUMN phone VARCHAR(255) NULL,
            MODIFY COLUMN social_type ENUM('KAKAO','NAVER','GOOGLE','APPLE') NULL,
            MODIFY COLUMN social_uid VARCHAR(255) NULL;
        ```

    - Swagger 결과

      ![image.png](./mission_08_(1).png)


2. **가게에 리뷰 추가하기 API**

- 3주차에 작성한 마이 페이지 리뷰 작성한 API 명세서를 기반으로 구현하였다.
- 요구사항 정의
    - 별점 입력 페이지
        - 가게 ID, 별점(1~5, 0.5 단위)
    - 리뷰 작성 페이지
        - 가게 ID, 별점, 리뷰 내용
        - 리뷰 사진 업로드
            - 사진 URL 최대 3개까지 저장 (review_photo 테이블)
    - 로그인 없음
        - member_id는 하드코딩된 회원으로 처리 (jwt 토큰 사용 X)
- 6주차에 생성한 DB 데이터가 충돌이 나서 새로운 시드를 넣었다.

    ```sql
    INSERT INTO region (id, name) VALUES
    (1, '서울');
    
    INSERT INTO store (id, region_id, name, address, identify_number) VALUES
    (1, 1, '반이학생마라탕마라반', '서울 서교동', '123-45-67890'),
    (2, 1, '장사부마라탕', '서울 개봉동', '999-999');
    ```

- Swagger 결과

  ![image.png](./mission_08_(2).png)


3. **가게의 미션을 도전 중인 미션에 추가(미션 도전하기) API**

- 요구사항 정의
    - 유저가 특정 미션에 “미션 도전!” 버튼을 누르면 member_mission 테이블에 추가
    - 초기 상태는 IN_PROGRESS
    - 로그인 기능 없음 → member_id 하드코딩
- API 명세
    - **Endpoint**

      `POST /missions/challenge`

    - **Request Body**

        ```json
        {
          "mission_id": 10
        }
        ```

    - **Response (201 Created)**

        ```json
        {
          "isSuccess": true,
          "code": "COMMON201_1",
          "message": "리소스가 성공적으로 생성되었습니다.",
          "result": {
            "memberMissionId": 1,
            "missionId": 1,
            "status": "IN_PROGRESS",
            "expiresAt": "2025-11-30T00:00:00",
            "createdAt": "2025-11-23T20:06:38.4427257"
          }
        }
        ```

- 와이어프레임 바탕으로 mission 테이블에 시드 데이터를 넣었다.

    ```sql
    INSERT INTO mission (id, store_id, mission_condition, deadline, point) VALUES
    (1, 1, '10,000원 이상의 식사 시', DATE_ADD(CURDATE(), INTERVAL 7 DAY), 500);
    ```

- Swagger 결과
    ![image.png](./mission_08_(3).png)