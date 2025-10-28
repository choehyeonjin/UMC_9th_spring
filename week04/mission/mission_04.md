## 미션 수행한 깃허브 리포지토리 링크:

- [UMC-9th-spring-study](https://github.com/choehyeonjin/UMC-9th-spring-study)

## ERD 사진:
- 0주차 erd 수정사항
  - user -> mission 변경 (예약어 방지)
  - mission 테이블 condition -> mission_condition 변경 (예약어 방지)
  ![erd_04.png](./erd_04.png)

## DB 사진

### 구조:

![db_structure_04.JPG](./db_structure_04.JPG)

### 각 테이블 DDL:

```sql
create table member
(
    id             bigint auto_increment
        primary key,
    created_at     datetime(6)                                not null,
    updated_at     datetime(6)                                not null,
    address        varchar(200)                               null,
    birthdate      date                                       null,
    deleted_at     datetime(6)                                null,
    email          varchar(255)                               not null,
    gender         enum ('FEMALE', 'MALE', 'NONE')            not null,
    name           varchar(255)                               not null,
    phone          varchar(255)                               not null,
    phone_verified bit                                        not null,
    point          int                                        not null,
    social_type    enum ('APPLE', 'GOOGLE', 'KAKAO', 'NAVER') not null,
    social_uid     varchar(255)                               not null,
    user_type      enum ('ADMIN', 'OWNER', 'USER')            not null
);

create table food
(
    id   bigint auto_increment
        primary key,
    name enum ('CAFE', 'CHINESE', 'JAPANESE', 'KOREAN', 'WESTERN') not null,
    constraint UKqkhr2yo38c1g9n5ss0jl7gxk6
        unique (name)
);

create table member_food
(
    id        bigint auto_increment
        primary key,
    food_id   bigint null,
    member_id bigint null,
    constraint FK5i0iwac2dcdifkxtb64cd17o8
        foreign key (member_id) references member (id),
    constraint FKj1eo2o1lys37eeqycfahfr0h9
        foreign key (food_id) references food (id)
);

create table term
(
    id          bigint auto_increment
        primary key,
    created_at  datetime(6)                                                   not null,
    updated_at  datetime(6)                                                   not null,
    is_required bit                                                           not null,
    type        enum ('IMAGE', 'LOCATION', 'MARKETING', 'PRIVACY', 'SERVICE') not null,
    constraint UKeij62dxd14rgsxcso48r7y4ko
        unique (type)
);

create table member_term
(
    id        bigint auto_increment
        primary key,
    agreed    bit         not null,
    agreed_at datetime(6) not null,
    member_id bigint      not null,
    term_id   bigint      not null,
    constraint FK99valiqden00uing9dy6221uy
        foreign key (term_id) references term (id),
    constraint FKrtih56dvkore774vnao4lglic
        foreign key (member_id) references member (id)
);

create table mission
(
    id                bigint auto_increment
        primary key,
    deadline          date         not null,
    mission_condition varchar(255) not null,
    point             int          not null,
    store_id          bigint       not null,
    constraint FKckx1b8plp95qtdk73kylhy12n
        foreign key (store_id) references store (id)
);

create table member_mission
(
    id         bigint auto_increment
        primary key,
    created_at datetime(6)                                                 not null,
    updated_at datetime(6)                                                 not null,
    expires_at datetime(6)                                                 null,
    status     enum ('CANCELLED', 'IN_PROGRESS', 'NOT_STARTED', 'SUCCESS') not null,
    member_id  bigint                                                      not null,
    mission_id bigint                                                      not null,
    constraint FKibnub11mc8k2g39v44smt9jqi
        foreign key (mission_id) references mission (id),
    constraint FKpgw7uojmq1tkna2ubpxmhlyuo
        foreign key (member_id) references member (id)
);

create table store
(
    id              bigint auto_increment
        primary key,
    address         varchar(255) not null,
    identify_number varchar(255) not null,
    name            varchar(255) not null,
    region_id       bigint       not null,
    constraint FKiecbc1b9m21semcf714lasyi5
        foreign key (region_id) references region (id)
);

create table region
(
    id   bigint auto_increment
        primary key,
    name varchar(255) not null
);

create table review
(
    id         bigint auto_increment
        primary key,
    content    varchar(255) not null,
    rating     float        not null,
    member_id  bigint       not null,
    store_id   bigint       not null,
    created_at datetime(6)  not null,
    constraint FK74d12ba8sxxu9vpnc59b43y30
        foreign key (store_id) references store (id),
    constraint FKk0ccx5i4ci2wd70vegug074w1
        foreign key (member_id) references member (id)
);

create table review_photo
(
    id        bigint auto_increment
        primary key,
    url       varchar(255) not null,
    review_id bigint       not null,
    constraint FK80ti8nek4uv8vn4vjhpre6mwg
        foreign key (review_id) references review (id)
);

create table review_reply
(
    id         bigint auto_increment
        primary key,
    created_at datetime(6)  not null,
    updated_at datetime(6)  not null,
    content    varchar(255) not null,
    review_id  bigint       not null,
    constraint UK4i2wqc6vs38qe09e5bx51t4py
        unique (review_id),
    constraint FK3mfomicwiqwm49ahq8yevep7x
        foreign key (review_id) references review (id)
);

create table inquiry
(
    id         bigint auto_increment
        primary key,
    created_at datetime(6)                                            not null,
    updated_at datetime(6)                                            not null,
    content    text                                                   not null,
    status     enum ('ANSWERED', 'CLOSED', 'PENDING')                 not null,
    title      varchar(255)                                           not null,
    type       enum ('ACCOUNT', 'MISSION', 'OTHER', 'POINT', 'STORE') not null,
    member_id  bigint                                                 not null,
    constraint FKf47xlnlpohnc6gd4k511g0wra
        foreign key (member_id) references member (id)
);

create table inquiry_photo
(
    id         bigint auto_increment
        primary key,
    url        varchar(255) not null,
    inquiry_id bigint       not null,
    constraint FK9m88uysxflyu2jdoehj473bs
        foreign key (inquiry_id) references inquiry (id)
);

create table inquiry_reply
(
    id         bigint auto_increment
        primary key,
    created_at datetime(6) not null,
    updated_at datetime(6) not null,
    content    text        not null,
    inquiry_id bigint      not null,
    constraint UKs5j1rc90ww3ba3p003vyp20qv
        unique (inquiry_id),
    constraint FKlswrro8h5dy5y2ba8q9useih7
        foreign key (inquiry_id) references inquiry (id)
);
```