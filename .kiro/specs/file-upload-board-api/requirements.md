# Requirements Document

## Introduction

파일 첨부가 가능한 게시판 REST API를 만듭니다. 이 문서는 `TODO.md`의 할 일 목록을 요구사항으로 정리한 것입니다.

- 기술 환경: Spring Boot, PostgreSQL (`README.md` 기준)
- 범위: 회원(가입, 로그인, 정보 수정, 탈퇴), 게시글(목록, 상세, 저장, 편집, 삭제), 첨부파일(업로드, 다운로드), DB 연결 확인
- 원칙: `AGENTS.md`의 "Simple is Best"에 따라 학습용으로 필요한 기능만 다룹니다. 화면(UI)은 범위에 포함하지 않습니다.
- 파일 저장 방식(Blob 또는 파일 경로)과 인증 방식 등 구현 방법에 관한 사항은 요구사항에서 정하지 않고, 하단의 "설계에서 결정할 사항"에 모아 두었습니다.

## Glossary

- **Board_API**: 게시판 기능을 HTTP로 제공하는 Spring Boot 서버 애플리케이션
- **Database**: Board_API가 회원, 게시글, 첨부파일 정보를 저장하는 PostgreSQL 데이터베이스
- **Member**: Board_API에 가입한 사용자
- **Post**: Member가 작성한 게시글. 제목, 내용, 작성자, 작성 시각을 가진다.
- **Attachment**: Post에 첨부된 파일
- **Author**: 해당 Post를 작성한 Member
- **Login_Session**: 로그인에 성공한 Member를 이후 요청에서 식별하기 위한 인증 상태
- **Authenticated_Request**: 유효한 Login_Session을 포함한 요청
- **Max_File_Size**: Attachment 하나의 최대 크기. 초기값은 10MB이며 설계에서 확정한다.
- **Max_File_Count**: Post 하나에 첨부할 수 있는 Attachment의 최대 개수. 초기값은 3개이며 설계에서 확정한다.

## Requirements

### Requirement 1: DB 연결 확인
**User Story:** As a 개발자, I want Board_API가 Database와 연결되었는지 확인하고 싶다, so that 실행 환경 문제를 빠르게 찾을 수 있다.

#### Acceptance Criteria

1. WHEN Board_API가 시작되면, THE Board_API SHALL Database 연결을 시도하고 성공 여부를 로그에 기록한다
2. WHEN 연결 확인 요청을 받으면, THE Board_API SHALL Database 연결 상태를 응답으로 반환한다
3. IF Database 연결에 실패하면, THEN THE Board_API SHALL 연결 실패 사유를 로그에 기록하고 연결 확인 요청에 실패 상태를 반환한다

### Requirement 2: 회원가입
**User Story:** As a 방문자, I want 회원가입을 하고 싶다, so that 게시판 기능을 사용할 수 있다.

#### Acceptance Criteria

1. WHEN 방문자가 아이디, 비밀번호, 이름, 이메일을 제출하면, THE Board_API SHALL 새 Member를 Database에 저장한다
2. THE Board_API SHALL Member의 비밀번호를 원문이 아닌 단방향 해시 값으로 저장한다
3. IF 제출한 아이디가 이미 존재하면, THEN THE Board_API SHALL 가입을 거부하고 중복 아이디 오류를 반환한다
4. IF 필수 입력값이 비어 있으면, THEN THE Board_API SHALL 가입을 거부하고 누락된 항목을 알리는 오류를 반환한다

### Requirement 3: 로그인
**User Story:** As a Member, I want 로그인하고 싶다, so that 내 계정으로 게시판을 사용할 수 있다.

#### Acceptance Criteria

1. WHEN Member가 올바른 아이디와 비밀번호를 제출하면, THE Board_API SHALL Login_Session을 발급한다
2. IF 아이디 또는 비밀번호가 일치하지 않으면, THEN THE Board_API SHALL 로그인을 거부하고 인증 실패 오류를 반환한다
3. WHEN Member가 로그아웃을 요청하면, THE Board_API SHALL 해당 Login_Session을 무효화한다

### Requirement 4: 회원정보 수정
**User Story:** As a Member, I want 내 회원정보를 수정하고 싶다, so that 이름, 이메일, 비밀번호를 최신 상태로 유지할 수 있다.

#### Acceptance Criteria

1. WHEN Authenticated_Request로 회원정보 수정이 요청되면, THE Board_API SHALL 요청한 Member의 이름, 이메일, 비밀번호를 갱신한다
2. WHEN 비밀번호가 변경되면, THE Board_API SHALL 새 비밀번호를 단방향 해시 값으로 저장한다
3. IF 요청이 Authenticated_Request가 아니면, THEN THE Board_API SHALL 수정을 거부하고 인증 필요 오류를 반환한다
4. THE Board_API SHALL Member의 아이디를 변경할 수 없도록 한다

### Requirement 5: 회원탈퇴
**User Story:** As a Member, I want 회원탈퇴를 하고 싶다, so that 내 정보를 서비스에서 제거할 수 있다.

#### Acceptance Criteria

1. WHEN Authenticated_Request로 회원탈퇴가 요청되면, THE Board_API SHALL 해당 Member를 Database에서 삭제한다
2. WHEN 회원탈퇴가 완료되면, THE Board_API SHALL 해당 Member의 Login_Session을 무효화한다
3. IF 요청이 Authenticated_Request가 아니면, THEN THE Board_API SHALL 탈퇴를 거부하고 인증 필요 오류를 반환한다
4. WHEN 회원탈퇴가 완료되면, THE Board_API SHALL 탈퇴한 Member가 작성한 Post와 Attachment를 설계에서 결정한 방식으로 처리한다

### Requirement 6: 게시글 목록 표시
**User Story:** As a 방문자, I want 게시글 목록을 보고 싶다, so that 어떤 글이 있는지 한눈에 확인할 수 있다.

#### Acceptance Criteria

1. WHEN 게시글 목록이 요청되면, THE Board_API SHALL 각 Post의 번호, 제목, 작성자 이름, 작성 시각을 최신 작성순으로 반환한다
2. THE Board_API SHALL 게시글 목록을 한 페이지당 10건 단위로 나누어 반환한다
3. IF 등록된 Post가 없으면, THEN THE Board_API SHALL 빈 목록과 성공 상태를 반환한다

### Requirement 7: 게시글 상세 표시
**User Story:** As a 방문자, I want 목록의 게시글을 선택해 세부내용을 보고 싶다, so that 글의 전체 내용과 첨부파일을 확인할 수 있다.

#### Acceptance Criteria

1. WHEN 특정 Post의 상세 조회가 요청되면, THE Board_API SHALL 해당 Post의 번호, 제목, 내용, 작성자 이름, 작성 시각, Attachment 목록을 반환한다
2. IF 요청한 번호의 Post가 존재하지 않으면, THEN THE Board_API SHALL 찾을 수 없음 오류를 반환한다

### Requirement 8: 게시글 저장
**User Story:** As a Member, I want 게시글을 저장하고 싶다, so that 다른 사람들과 내용을 공유할 수 있다.

#### Acceptance Criteria

1. WHEN Authenticated_Request로 제목과 내용이 제출되면, THE Board_API SHALL 요청한 Member를 Author로 하는 새 Post를 저장한다
2. WHEN Post가 저장되면, THE Board_API SHALL 저장된 Post의 번호를 반환한다
3. IF 제목 또는 내용이 비어 있으면, THEN THE Board_API SHALL 저장을 거부하고 누락된 항목을 알리는 오류를 반환한다
4. IF 요청이 Authenticated_Request가 아니면, THEN THE Board_API SHALL 저장을 거부하고 인증 필요 오류를 반환한다

### Requirement 9: 게시글 편집
**User Story:** As a Author, I want 내가 쓴 게시글을 편집하고 싶다, so that 내용을 바로잡을 수 있다.

#### Acceptance Criteria

1. WHEN Author의 Authenticated_Request로 제목과 내용의 수정이 요청되면, THE Board_API SHALL 해당 Post의 제목과 내용을 갱신한다
2. IF 요청한 Member가 해당 Post의 Author가 아니면, THEN THE Board_API SHALL 수정을 거부하고 권한 없음 오류를 반환한다
3. IF 요청한 번호의 Post가 존재하지 않으면, THEN THE Board_API SHALL 찾을 수 없음 오류를 반환한다
4. IF 제목 또는 내용이 비어 있으면, THEN THE Board_API SHALL 수정을 거부하고 누락된 항목을 알리는 오류를 반환한다

### Requirement 10: 게시글 삭제
**User Story:** As a Author, I want 내가 쓴 게시글을 삭제하고 싶다, so that 더 이상 공개하고 싶지 않은 글을 없앨 수 있다.

#### Acceptance Criteria

1. WHEN Author의 Authenticated_Request로 Post 삭제가 요청되면, THE Board_API SHALL 해당 Post와 그 Post의 모든 Attachment를 삭제한다
2. IF 요청한 Member가 해당 Post의 Author가 아니면, THEN THE Board_API SHALL 삭제를 거부하고 권한 없음 오류를 반환한다
3. IF 요청한 번호의 Post가 존재하지 않으면, THEN THE Board_API SHALL 찾을 수 없음 오류를 반환한다

### Requirement 11: 파일 첨부 및 업로드
**User Story:** As a Author, I want 게시글에 파일을 첨부하고 싶다, so that 글과 함께 자료를 공유할 수 있다.

#### Acceptance Criteria

1. WHEN Author의 Authenticated_Request로 Post에 파일 업로드가 요청되면, THE Board_API SHALL 파일을 저장하고 해당 Post의 Attachment로 연결한다
2. THE Board_API SHALL 각 Attachment의 원본 파일명, 크기, 파일 형식을 Database에 기록한다
3. IF 업로드한 파일의 크기가 Max_File_Size를 초과하면, THEN THE Board_API SHALL 업로드를 거부하고 파일 크기 초과 오류를 반환한다
4. IF 해당 Post의 Attachment 개수가 Max_File_Count에 도달한 상태에서 업로드가 요청되면, THEN THE Board_API SHALL 업로드를 거부하고 첨부 개수 초과 오류를 반환한다
5. IF 요청한 Member가 해당 Post의 Author가 아니면, THEN THE Board_API SHALL 업로드를 거부하고 권한 없음 오류를 반환한다
6. WHEN Author의 Authenticated_Request로 Attachment 삭제가 요청되면, THE Board_API SHALL 해당 Attachment를 삭제한다

### Requirement 12: 첨부파일 다운로드
**User Story:** As a 방문자, I want 첨부된 파일을 내려받고 싶다, so that 게시글에 공유된 자료를 사용할 수 있다.

#### Acceptance Criteria

1. WHEN 특정 Attachment의 다운로드가 요청되면, THE Board_API SHALL 업로드된 원본과 동일한 내용의 파일을 원본 파일명과 함께 반환한다
2. IF 요청한 Attachment가 존재하지 않으면, THEN THE Board_API SHALL 찾을 수 없음 오류를 반환한다
3. FOR ALL 업로드된 파일, 업로드한 뒤 다운로드한 파일의 바이트 내용 SHALL 업로드한 원본과 같다 (round-trip property)

## 설계에서 결정할 사항

요구사항 단계에서는 정하지 않고, 설계 문서에서 비교한 뒤 하나로 확정할 항목입니다.

| 번호 | 결정 항목 | 선택지 | 관련 요구사항 |
|------|-----------|--------|---------------|
| 1 | 업로드 파일 저장 방식 | Database Blob 저장 / 서버 파일 경로 저장 | 11, 12 |
| 2 | 인증 방식 | 서버 세션 / JWT | 3 |
| 3 | 비밀번호 해시 방식 | Spring Security의 BCrypt 등 | 2, 4 |
| 4 | 회원탈퇴 시 작성 글 처리 | 글 함께 삭제 / 글 유지(작성자 표시 변경) | 5 |
| 5 | Max_File_Size, Max_File_Count 확정값 | 초기값 10MB, 3개 | 11 |
| 6 | 허용 파일 형식 제한 여부 | 제한 없음 / 확장자 허용 목록 | 11 |
| 7 | 게시글 조회, 파일 다운로드의 로그인 필요 여부 | 현재는 로그인 없이 허용으로 가정 | 6, 7, 12 |
| 8 | Attachment를 Post 저장 요청과 함께 받을지 별도 요청으로 받을지 | 한 번에 전송 / 분리 | 8, 11 |

## 개정이력

| 일시 (KST) | 내용 |
|------------|------|
| 2026-11-24 09:54:53 | 최초 작성 |
