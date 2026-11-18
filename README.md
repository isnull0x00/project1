# 냉·난방 AS 실시간 예약 (AS-FAST)

회원이 냉·난방 AS를 **예약·조회·삭제하고 리뷰를 남기면**, 관리자가 접수 현황과 리뷰를 **대시보드로 확인**하는 예약 시스템입니다.
그린AI캠퍼스 1차 팀 프로젝트(COMMIT, 3인 · 2026.09.08 ~ 09.16)입니다.

## 제가 맡은 부분

저는 예약 기능을 맡았습니다. [내 커밋 보기](https://github.com/isnull0x00/project1/commits/dev?author=isnull0x00)

- 예약 DB 스키마를 12개에서 4개 테이블로 줄여 설계했습니다.
- 날짜·시간 선택 캘린더와 타임슬롯, 예약 내역 조회와 취소 기능 구현에 참여했고, 예약 내역 조회 화면(`reserve_list`)은 주로 제가 작성했습니다.

### 동시 예약 방지

- **문제:** 자바 `if`로 중복을 확인해도, 확인과 저장 사이의 짧은 틈에 두 요청이 모두 통과해 같은 시간이 이중 예약될 수 있습니다.
- **해결:** MariaDB 가상 컬럼 `SLOT_KEY`에 유니크 인덱스를 걸어, 중복 판단을 DB가 하게 했습니다. 취소된 예약은 키가 `NULL`이라 같은 시간에 다시 예약할 수 있습니다.
- **확인:** 브라우저 콘솔에서 같은 날짜·슬롯으로 10건을 동시에 보내 1건만 저장되고 9건이 "이미 예약된 시간입니다"로 차단되는 것을 직접 확인했고, DB 조회에서도 1건만 있었습니다.

<details>
<summary>검토한 방식과 동작 방식 보기</summary>

| 방식 | 판단 |
|---|---|
| 비관적 락 (`SELECT ... FOR UPDATE`) | 다른 요청이 대기열에 묶여, 단순한 신규 예약에는 오버헤드가 크다고 판단했습니다. |
| 낙관적 락 (`@Version`) | 이미 있는 행을 수정할 때 쓰는 방식이라, 새 행을 넣는 예약 생성에는 맞지 않았습니다. |
| `UNIQUE(날짜, 슬롯)` | 취소된 예약 행이 남아 있어 같은 시간을 다시 예약할 수 없었습니다. 취소 이력은 지울 수 없어서 쓰지 않았습니다. |
| 가상 컬럼 `SLOT_KEY` + 유니크 (채택) | 취소 건은 키가 `NULL`이 되어 유니크 검사에서 빠지고, 취소 처리에서 키를 따로 비울 필요가 없습니다. |

```sql
ALTER TABLE RESERVE ADD COLUMN SLOT_KEY VARCHAR(30) AS (
  IF(RESERVE_STATUS <> 'C', CONCAT(RESERVE_DATE, '-', SLOT_NO), NULL)
) VIRTUAL;
ALTER TABLE RESERVE ADD CONSTRAINT UX_RESERVE_SLOT UNIQUE (SLOT_KEY);
```

1. 두 번째 요청의 `INSERT`가 유니크 인덱스에 막혀 중복 오류가 납니다.
2. 컨트롤러가 `DataIntegrityViolationException`을 받아 "이미 예약된 시간입니다"로 응답합니다([`ReserveController.java`](src/main/java/com/commit/project1/reserve/controller/ReserveController.java#L141)).

**한계:** 위 스키마 SQL은 이 저장소에 없습니다. 동시 요청 시험은 자동 테스트가 아니라 수동으로 한 번 측정한 것입니다.

</details>

![시연](docs/demo.gif)

> 로그인 → AS 예약(제품 · 일정 선택) → 예약 내역 조회 → 리뷰 작성 → 관리자 대시보드 순서의 시연입니다.

## 한눈에 보기 (팀 결과)

| | |
|---|---|
| 카테고리 통계 | 예약이 0건인 카테고리도 보이도록 `RIGHT OUTER JOIN`으로 건수 집계 |
| 리뷰 통계 | 전체 건수 · 평균 평점 · 별점별 개수를 **한 번의 쿼리**로 집계 |
| 예약 흐름 | 제품 → 일정 → 확정의 3단계 화면 (FullCalendar, axios 비동기) |
| 데이터 | `RESERVE` 테이블 중심으로 5개 테이블 연결 |

## 핵심 코드

| 주제 | 위치 |
|---|---|
| 동시 예약 중복을 사용자 메시지로 바꾸는 처리 | [`ReserveController.java`](src/main/java/com/commit/project1/reserve/controller/ReserveController.java#L141) |
| 이미 예약된 슬롯 번호 조회 | [`reserve-mapper.xml`](src/main/resources/mapper/reserve-mapper.xml#L51) |
| 예약 시간(타임슬롯) 선택 화면 | [`reserve_time.js`](src/main/resources/static/js/pages/reserve/reserve_time.js) |
| 예약 내역 조회 화면 | [`reserve_list.js`](src/main/resources/static/js/pages/reserve/reserve_list.js) |

```java
try {
  reserveService.saveReserve(dto);
} catch (DataIntegrityViolationException e) {
  result.put("success", false);
  result.put("error", "이미 예약된 시간입니다.");
  return result;
}
```

DB의 중복 오류를 서비스 코드가 직접 판단하지 않고, 예외를 받아 사용자 메시지로만 바꿉니다.

## 기술 스택

Java 17 · Spring Boot · MyBatis · Thymeleaf · MariaDB · JavaScript(axios) · FullCalendar · Gradle

## 구조

```
src/main/java/com/commit/project1
├─ member    회원 · 로그인
├─ reserve   예약 (Controller / Service / Mapper / DTO)
├─ review    리뷰
└─ admin     관리자 대시보드
src/main/resources
├─ mapper    MyBatis XML
├─ templates Thymeleaf (layouts / pages)
└─ static    CSS · JS
```

## 실행

1. MariaDB에 `project-1` 데이터베이스를 준비합니다. (스키마 SQL은 이 저장소에 없습니다.)
2. `src/main/resources/application.yaml`의 DB 접속 정보와 업로드 경로(`file.upload.dir`)를 본인 환경에 맞게 고칩니다.
3. 실행합니다.

```bash
./gradlew bootRun
```

## 한계와 다음 단계

- 자동 테스트가 애플리케이션 로드 테스트 하나뿐이라 통계 쿼리를 테스트로 고정하지 못했습니다.
- 통계는 요청마다 전체 테이블을 집계합니다. 데이터가 커지면 느려질 수 있고 측정하지 않았습니다.
- 동시 예약은 제출한 뒤에야 거절을 알게 되므로, 일정 선택 시점에 알려 주는 구조(예: Redis 슬롯 홀딩)가 다음 과제입니다.

## 기록

- [프로젝트 작업 일지 (Notion)](https://fluffy-plane-19c.notion.site/3e3a969c48278070bbaaf5dee74b31b4)
