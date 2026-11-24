# Implementation Plan: file-upload-board-api

## Overview

파일 첨부가 가능한 게시판 REST API를 `Src/` 하위에 Java 25, Spring Boot 4.1 이상, Gradle 9.1 이상으로 구현합니다. 설계 문서의 결정(세션 인증, BCrypt, `bytea` 저장, `@OnDelete(CASCADE)`)을 따르며, 인터페이스와 별도 변환 계층은 만들지 않습니다.

구현 순서는 프로젝트 설정과 DB 연결 확인, 회원, 게시글, 첨부파일입니다. 각 단계는 앞 단계의 코드 위에 쌓이며, 마지막 단계에서 게시글 상세에 첨부 목록을 연결해 모든 코드가 하나로 이어집니다. 속성 테스트(jqwik 1.9.3)는 해당 코드를 구현한 직후에 배치합니다. 설계 문서의 "Spring Boot 4 전환 검토" 표에서 "구현 시 확인"으로 남은 항목은 해당 코드를 작성하는 작업에서 확인합니다(하단 Notes 참고).

## Tasks

- [ ] 1. 프로젝트 설정과 DB 연결 확인
  - [ ] 1.1 Gradle 프로젝트와 설정 파일 만들기
    - `Src/` 하위에 Gradle 9.1 이상 프로젝트를 만들고 Spring Boot Gradle 플러그인 4.1.x를 적용한다. Java 25는 Gradle 툴체인(`JavaLanguageVersion.of(25)`)으로 지정한다.
    - `BoardApplication`(패키지 `com.example.board`)을 작성한다.
    - 설계 문서의 의존성(`spring-boot-starter-webmvc`, `spring-boot-starter-data-jpa`, `spring-boot-starter-validation`, `postgresql`, `spring-security-crypto`, 테스트용 `spring-boot-starter-webmvc-test`, `spring-boot-starter-data-jpa-test`, `net.jqwik:jqwik:1.9.3`)을 추가한다. `jqwik-spring`과 `spring-boot-starter-security`는 넣지 않는다. 버전은 jqwik만 직접 지정한다.
    - 빌드해 보고 `validation`에 `-test` 스타터가 필요한지, 필요하면 추가한다(설계 문서의 "구현 시 확인" 항목).
    - `application.yml`에 설계 문서의 설정(DB 접속 정보는 환경변수, `ddl-auto: update`, `open-in-view: false`, multipart `max-file-size: 10MB`, `max-request-size: 11MB`)을 작성한다.
    - 테스트용 설정 `application-test.yml`을 만들어 `board_test` DB에 연결하고, 테스트에서는 `ddl-auto: create`로 `on delete cascade`가 포함된 테이블을 새로 만든다.
    - _Requirements: 1.1_

  - [ ] 1.2 공통 오류 처리 만들기
    - `ApiException(HttpStatus, message)`, `ErrorResponse(message, fields)`, `GlobalExceptionHandler`를 `common` 패키지에 작성한다.
    - `ApiException`, 검증 오류(`MethodArgumentNotValidException`, 누락 항목을 `fields`에 담음), `MaxUploadSizeExceededException`(413)을 JSON 오류 응답으로 바꾼다.
    - 예상하지 못한 예외는 Spring 기본 처리(500)에 맡기고 내부 정보를 응답에 싣지 않는다.
    - _Requirements: 2.4, 8.3, 9.4, 11.3_

  - [ ] 1.3 DB 연결 확인 기능 만들기
    - `DbChecker`가 `DataSource.getConnection().isValid(2)`로 연결을 한 번 확인한다. 실패하면 사유를 `log.error`로 남기고 `ApiException`(503)을 던진다.
    - `ApplicationRunner`로 기동 시 `DbChecker`를 호출해 성공 여부를 로그에 남긴다.
    - `DbCheckController`에 `GET /api/health/db`를 추가해 `{"status":"UP"}`을 반환한다.
    - _Requirements: 1.1, 1.2, 1.3_

  - [ ]* 1.4 DB 연결 확인 테스트 작성하기
    - `@SpringBootTest`, `@AutoConfigureMockMvc`(`org.springframework.boot.webmvc.test.autoconfigure`), `@ActiveProfiles("test")`로 `MockMvc`를 사용한다.
    - `GET /api/health/db`가 200 `UP`을 반환하는지 확인한다.
    - `@MockitoBean`으로 `DataSource`가 예외를 던지도록 바꾸면 503을 반환하는지 확인한다(`@MockBean`은 쓰지 않는다).
    - 스모크 테스트로 애플리케이션이 `board_test` DB에 연결되어 기동하는지 확인한다.
    - _Requirements: 1.1, 1.2, 1.3_

  - [ ]* 1.5 속성 테스트 실행 기반 확인하기
    - 테스트 코드에서 `SpringApplicationBuilder(BoardApplication.class).profiles("test").run()`으로 Spring 컨텍스트를 한 번만 시작해 정적 필드에 두고, 서비스를 `getBean`으로 꺼내는 공용 테스트 보조 클래스를 작성한다. `jqwik-spring`은 쓰지 않는다.
    - 보조 클래스를 사용하는 jqwik 속성 하나(예: 컨텍스트에서 `MemberRepository`를 꺼내 호출)를 작성해 jqwik 1.9.3이 Boot 4의 JUnit 6 위에서 실행되는지 확인한다.
    - 실행되지 않으면 속성 테스트(3.5, 5.4, 5.5, 7.5~7.8)를 JUnit 6 `@ParameterizedTest`와 `@MethodSource`(시드를 고정한 `java.util.Random`으로 100건 생성)로 구현하고, `jqwik` 의존성을 제거한다. 속성 정의와 `Feature: ..., Property N: ...` 주석 형식은 유지한다.
    - 확인 결과를 `Notes`에 한 줄로 남긴다.

- [ ] 2. Checkpoint - 프로젝트 설정 확인
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 3. 회원 기능 구현
  - [ ] 3.1 Member 엔티티와 저장소 만들기
    - `Member`(`username` 유니크, setter 없음)와 `update(name, email, newPasswordHash)`를 작성한다. `newPasswordHash`가 null이면 비밀번호를 유지한다.
    - `MemberRepository`에 `existsByUsername`, `findByUsername`을 둔다.
    - _Requirements: 2.1, 4.1, 4.4_

  - [ ] 3.2 LoginSession과 BCrypt 설정 만들기
    - `BCryptPasswordEncoder`를 빈으로 등록한다. Boot BOM이 관리하는 Spring Security 7.x에서 클래스 패키지가 그대로인지 컴파일로 확인한다.
    - `LoginSession`을 작성한다. `login`은 기존 세션을 폐기하고 새 세션에 회원 번호를 저장한다. `requireMember`는 로그인하지 않았거나 DB에 회원이 없으면 401을 던진다. `logout`은 `invalidate()`한다.
    - _Requirements: 3.1, 3.3, 4.3, 5.2, 5.3_

  - [ ] 3.3 MemberService와 요청 record 만들기
    - `JoinRequest`, `LoginRequest`, `MemberUpdateRequest` record를 설계 문서의 검증 규칙대로 작성한다. 비밀번호는 가입, 로그인, 정보 수정 모두 `@Size(max = 20)`을 둔다(`LoginRequest`도 포함). 72바이트를 넘는 비밀번호가 BCrypt까지 가서 `IllegalArgumentException`(500)이 나지 않게 하기 위해서다.
    - `MemberService`에 가입(아이디 중복이면 409, BCrypt 해시 저장), 로그인(아이디 없음과 비밀번호 불일치 모두 같은 401 메시지), 정보 수정(비밀번호가 비어 있으면 유지, 변경하면 해시 저장), 탈퇴(`memberRepository.delete`)를 `@Transactional`로 구현한다.
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 3.1, 3.2, 4.1, 4.2, 4.4, 5.1, 5.4_

  - [ ] 3.4 MemberController 만들기
    - `POST /api/members`(201), `POST /api/login`(200), `POST /api/logout`(204), `PUT /api/members/me`(200), `DELETE /api/members/me`(204)를 구현한다.
    - 보호된 API는 `LoginSession.requireMember`로 회원을 확인한다. 탈퇴가 끝나면 현재 세션을 `invalidate()`한다.
    - _Requirements: 2.1, 3.1, 3.3, 4.1, 4.3, 5.1, 5.2, 5.3_

  - [ ]* 3.5 비밀번호 해시 속성 테스트 작성하기
    - **Property 6: 비밀번호는 해시로 저장되고 같은 비밀번호만 통과한다**
    - 1.5의 보조 클래스로 컨텍스트를 시작하고, jqwik으로 1~20자의 서로 다른 비밀번호 p, q를 생성한다. 가입과 비밀번호 변경 후 저장된 값이 p와 다르고, p로 로그인하면 성공, q로 로그인하면 401인지 확인한다.
    - 100회 이상 반복하고, `@BeforeTry`에서 테이블을 비우며, `Feature: file-upload-board-api, Property 6: ...` 주석을 단다.
    - **Validates: Requirements 2.1, 2.2, 3.1, 3.2, 4.2**

  - [ ]* 3.6 회원 기능 예제 테스트 작성하기
    - `MockMvc`로 아이디 중복 가입 409, 필수값 누락 400을 확인한다.
    - 로그인 후 보호된 API 호출 성공, 로그아웃 후 401을 확인한다.
    - 로그인하지 않은 상태에서 회원정보 수정과 탈퇴가 401인지 확인한다.
    - 회원정보 수정 후 아이디가 유지되고, 비밀번호를 비우면 기존 비밀번호로 로그인되는지 확인한다.
    - 탈퇴 후 같은 세션으로 호출하면 401인지 확인한다.
    - _Requirements: 2.3, 2.4, 3.1, 3.3, 4.1, 4.3, 4.4, 5.1, 5.2, 5.3_

  - [ ]* 3.7 긴 비밀번호 로그인 테스트 작성하기
    - 21자 이상의 비밀번호로 로그인하면 500이 아니라 400이 반환되는지 `MockMvc`로 확인한다. 3.6과 같은 테스트 클래스에 추가한다.
    - _Requirements: 2.4, 3.2_

- [ ] 4. Checkpoint - 회원 기능 확인
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 5. 게시글 기능 구현
  - [ ] 5.1 Post 엔티티와 저장소 만들기
    - `Post`를 작성한다. `author`는 `@ManyToOne(LAZY)`와 `@OnDelete(CASCADE)`로 두고, `content`는 `text` 컬럼, `createdAt`은 생성자에서 현재 시각으로 지정한다. `isWrittenBy(member)`와 `update(title, content)`를 둔다.
    - `PostRepository`에 `select new PostSummary(...) ... order by p.id desc` 목록 조회를 작성한다.
    - _Requirements: 6.1, 8.1, 9.1, 9.2, 5.4_

  - [ ] 5.2 PostService와 요청, 응답 record 만들기
    - `PostRequest`, `PostSummary`, `PostDetail` record를 작성한다. `AttachmentInfo`는 7.1에서 만들므로, 이 단계에서는 `PostDetail`의 첨부 목록을 빈 목록으로 채운다.
    - `PostService`에 목록(`PageRequest.of(max(page, 0), 10)`), 상세(없으면 404), 저장(작성자 지정, 번호 반환), 편집(404, 작성자가 아니면 403), 삭제(404, 403)를 `@Transactional`로 구현한다.
    - _Requirements: 6.1, 6.2, 6.3, 7.1, 7.2, 8.1, 8.2, 9.1, 9.2, 9.3, 10.1, 10.2, 10.3_

  - [ ] 5.3 PostController 만들기
    - `GET /api/posts?page=0`, `GET /api/posts/{id}`, `POST /api/posts`(201 `{"id": 번호}`), `PUT /api/posts/{id}`, `DELETE /api/posts/{id}`(204)를 구현한다.
    - 저장, 편집, 삭제는 `LoginSession.requireMember`로 회원을 확인한다. 조회는 로그인 없이 허용한다.
    - _Requirements: 6.1, 7.1, 8.1, 8.2, 8.4, 9.1, 10.1_

  - [ ]* 5.4 게시글 저장과 편집 round trip 속성 테스트 작성하기
    - **Property 1: 게시글 저장과 편집 round trip**
    - 한글, 공백, 줄바꿈, 특수문자를 포함한 유효한 제목과 내용을 생성한다. 저장 후 상세 조회 결과가 입력과 같고, 다른 값으로 편집한 뒤에도 조회 결과가 편집한 값과 같은지 확인한다.
    - **Validates: Requirements 7.1, 8.1, 8.2, 9.1**

  - [ ]* 5.5 게시글 목록 속성 테스트 작성하기
    - **Property 2: 게시글 목록은 최신순이며 페이지당 10건이다**
    - 게시글 수 N(0~35)과 페이지 p(0~4)를 생성한다. 응답 건수가 `min(10, max(0, N - 10p))`이고, 번호 내림차순이며, 각 항목에 번호, 제목, 작성자 이름, 작성 시각이 있는지 확인한다.
    - **Validates: Requirements 6.1, 6.2, 6.3**

  - [ ]* 5.6 게시글 기능 예제 테스트 작성하기
    - `MockMvc`로 제목 또는 내용이 비어 있을 때 저장과 편집이 400인지 확인한다.
    - 로그인하지 않은 상태에서 저장, 편집, 삭제가 401인지 확인한다.
    - 존재하지 않는 번호의 상세, 편집, 삭제가 404인지 확인한다.
    - _Requirements: 7.2, 8.3, 8.4, 9.3, 9.4, 10.3_

  - [ ]* 5.7 createdAt JSON 형식 테스트 작성하기
    - 게시글 상세와 목록 응답의 `createdAt`이 숫자가 아닌 ISO 문자열인지 `MockMvc`로 확인한다(Jackson 3 기본값 확인). 5.6과 같은 테스트 클래스에 추가한다.
    - 숫자로 나오면 `application.yml`의 Jackson 설정으로 ISO 문자열이 되게 고친다.
    - _Requirements: 6.1, 7.1_

- [ ] 6. Checkpoint - 게시글 기능 확인
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 7. 첨부파일 기능 구현
  - [ ] 7.1 Attachment 엔티티, 저장소, AttachmentInfo 만들기
    - `Attachment`를 작성한다. `post`는 `@ManyToOne(LAZY)`와 `@OnDelete(CASCADE)`로 두고, `content`는 `@Lob` 없이 `byte[]`로 두어 `bytea`로 저장한다.
    - `AttachmentRepository`에 `countByPostId`와, `content`를 읽지 않고 `AttachmentInfo`를 만드는 `select new AttachmentInfo(...)` 목록 조회를 작성한다.
    - `AttachmentInfo(id, originalName, size, contentType)` record를 작성한다.
    - _Requirements: 11.1, 11.2, 7.1_

  - [ ] 7.2 AttachmentService 만들기
    - 업로드: Post 존재(404), 작성자(403), 개수 3개 이상(400) 순으로 확인한 뒤 원본 파일명, 크기, 파일 형식과 함께 저장하고 `AttachmentInfo`를 반환한다.
    - 삭제: Attachment 존재(404), 소속 Post의 작성자(403)를 확인한 뒤 삭제한다.
    - 다운로드 조회: 없으면 404를 던진다.
    - _Requirements: 11.1, 11.2, 11.4, 11.5, 11.6, 12.1, 12.2_

  - [ ] 7.3 AttachmentController 만들기
    - `POST /api/posts/{id}/attachments`(multipart, 파트 이름 `file`, 201 `AttachmentInfo`), `DELETE /api/attachments/{id}`(204), `GET /api/attachments/{id}`를 구현한다.
    - 업로드와 삭제는 `LoginSession.requireMember`로 회원을 확인한다. 다운로드는 로그인 없이 허용한다.
    - 다운로드는 `ContentDisposition.attachment().filename(name, UTF_8)`와 저장된 `contentType`으로 `byte[]`를 반환한다. `ContentDisposition`, `ResponseEntity`가 Spring Framework 7에서 같은 이름과 패키지로 컴파일되는지 확인한다.
    - _Requirements: 11.1, 11.3, 11.6, 12.1, 12.2_

  - [ ] 7.4 게시글 상세에 첨부 목록 연결하기
    - `PostService.상세`에서 5.2의 빈 목록 대신 `AttachmentRepository`의 `AttachmentInfo` 목록을 `PostDetail`에 담는다.
    - _Requirements: 7.1_

  - [ ]* 7.5 파일 업로드와 다운로드 round trip 속성 테스트 작성하기
    - **Property 4: 파일 업로드와 다운로드 round trip**
    - 길이 0~64KB의 바이트 배열과 원본 파일명, 파일 형식을 생성한다. 업로드 후 상세 조회의 첨부 목록에 같은 파일명, 크기, 파일 형식이 있고, 다운로드한 바이트가 원본과 같으며 응답 파일명이 원본과 같은지 확인한다.
    - **Validates: Requirements 11.1, 11.2, 12.1, 12.3**

  - [ ]* 7.6 첨부파일 개수 제한 속성 테스트 작성하기
    - **Property 5: 게시글당 첨부파일은 3개를 넘지 않는다**
    - 업로드 시도 횟수 n을 생성한다. 성공이 `min(n, 3)`번이고 나머지는 400으로 거부되며, 어느 시점에도 첨부파일 수가 3개 이하인지 확인한다.
    - **Validates: Requirements 11.4**

  - [ ]* 7.7 작성자가 아닌 회원의 변경 거부 속성 테스트 작성하기
    - **Property 3: 작성자가 아닌 회원의 변경은 거부된다**
    - 게시글과 작성자가 아닌 임의의 회원을 생성한다. 그 회원의 편집, 삭제, 파일 업로드, 첨부파일 삭제가 모두 403이고 게시글과 첨부파일의 상태가 요청 전과 같은지 확인한다.
    - **Validates: Requirements 9.2, 10.2, 11.5**

  - [ ]* 7.8 삭제 시 하위 데이터 제거 속성 테스트 작성하기
    - **Property 7: 삭제하면 하위 데이터가 남지 않는다**
    - 첨부파일 0~3개를 가진 게시글을 삭제하면 게시글과 첨부파일이 모두 조회되지 않는지 확인한다.
    - 여러 회원의 게시글과 첨부파일을 만든 뒤 한 회원이 탈퇴하면 그 회원의 데이터만 사라지고 다른 회원의 데이터는 남는지 확인한다.
    - **Validates: Requirements 5.4, 10.1**

  - [ ]* 7.9 첨부파일 기능 예제 테스트 작성하기
    - `MockMvc`로 정확히 10MB 파일은 201, 10MB + 1바이트는 413인지 확인한다.
    - 로그인하지 않은 상태의 업로드와 첨부파일 삭제가 401인지 확인한다.
    - 첨부파일 삭제 후 다운로드가 404인지 확인한다.
    - 존재하지 않는 Post나 Attachment 번호에 404를 반환하는지 확인한다.
    - 다운로드 응답의 `Content-Disposition`이 `attachment`이고 한글 파일명이 유지되는지 확인한다.
    - _Requirements: 11.3, 11.6, 12.1, 12.2_

  - [ ]* 7.10 생성된 스키마 확인 테스트 작성하기
    - `board_test` DB의 `information_schema`와 `pg_constraint`를 조회해 `attachment.content`가 `bytea`인지 확인한다.
    - `post.author_id`와 `attachment.post_id` 외래 키가 `on delete cascade`인지 확인한다(Hibernate 7 DDL 확인). 7.9와 같은 테스트 클래스에 추가한다.
    - _Requirements: 5.4, 10.1, 11.1_

- [ ] 8. Final checkpoint - 전체 테스트 확인
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- `*` 표시가 붙은 하위 작업은 선택 사항이며, 빠른 MVP를 위해 건너뛸 수 있습니다.
- 각 작업은 요구사항 번호를 참조해 추적할 수 있습니다.
- 속성 테스트는 설계 문서의 Correctness Properties 7개를 하나씩 구현하며 jqwik 1.9.3을 사용합니다. Spring 컨텍스트는 1.5의 보조 클래스로 시작합니다. jqwik이 JUnit 6에서 실행되지 않으면 1.5의 대안(`@ParameterizedTest`)으로 바꿉니다.
- 설계 문서 "Spring Boot 4 전환 검토"의 "구현 시 확인" 항목은 다음 작업에서 확인합니다.
  - `validation`의 `-test` 스타터 필요 여부: 1.1
  - jqwik 1.9.3과 JUnit 6 동작: 1.5
  - `BCryptPasswordEncoder` 패키지: 3.2
  - `LocalDateTime`의 ISO 문자열 JSON 형식: 5.7
  - `ContentDisposition`, `ResponseEntity` 컴파일: 7.3
  - `bytea`, `on delete cascade` DDL: 7.10
- 테스트는 `@MockBean` 대신 `@MockitoBean`을, `@AutoConfigureMockMvc`는 `org.springframework.boot.webmvc.test.autoconfigure` 패키지의 것을 사용합니다.
- 테스트는 인메모리 DB가 아니라 PostgreSQL `board_test` DB에서 실행합니다. 테스트 실행 전에 이 DB를 만들어 두어야 합니다.
- DB 접속 계정과 비밀번호는 환경변수(`DB_USERNAME`, `DB_PASSWORD`)로 전달하며 코드와 저장소에 넣지 않습니다.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2"] },
    { "id": 2, "tasks": ["1.3"] },
    { "id": 3, "tasks": ["1.4", "1.5"] },
    { "id": 4, "tasks": ["2"] },
    { "id": 5, "tasks": ["3.1"] },
    { "id": 6, "tasks": ["3.2"] },
    { "id": 7, "tasks": ["3.3"] },
    { "id": 8, "tasks": ["3.4"] },
    { "id": 9, "tasks": ["3.5", "3.6"] },
    { "id": 10, "tasks": ["3.7"] },
    { "id": 11, "tasks": ["4"] },
    { "id": 12, "tasks": ["5.1"] },
    { "id": 13, "tasks": ["5.2"] },
    { "id": 14, "tasks": ["5.3"] },
    { "id": 15, "tasks": ["5.4", "5.5", "5.6"] },
    { "id": 16, "tasks": ["5.7"] },
    { "id": 17, "tasks": ["6"] },
    { "id": 18, "tasks": ["7.1"] },
    { "id": 19, "tasks": ["7.2"] },
    { "id": 20, "tasks": ["7.3", "7.4"] },
    { "id": 21, "tasks": ["7.5", "7.6", "7.7", "7.8", "7.9"] },
    { "id": 22, "tasks": ["7.10"] },
    { "id": 23, "tasks": ["8"] }
  ]
}
```

## 개정이력

| 일시 (KST) | 내용 |
|------------|------|
| 2026-11-24 10:02:22 | 최초 작성 |
| 2026-11-24 10:27:35 | 설계 변경 반영: Java 25, Spring Boot 4.1 이상, Gradle 9.1 이상. 의존성 갱신, jqwik-spring 제거. 테스트 설정 작업(1.4) 갱신. 속성 테스트 실행 기반 확인(1.5), 긴 비밀번호 로그인(3.7), createdAt 형식(5.7), 스키마 확인(7.10) 작업 추가. 전환 검토의 구현 시 확인 항목을 Notes에 정리 |
