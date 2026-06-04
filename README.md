# SeatUp

공연 티켓 예매 서비스

대규모 트래픽 상황에서의 동시 접속 제어와 실시간 대기열 관리를 중심으로, 실제 서비스와 유사한 예매 흐름을 구현했습니다.

---

## 프로젝트 소개

티켓팅 서비스를 이용하면서 동시 접속 처리 방식에 대한 궁금증에서 시작한 개인 프로젝트입니다.

또한 Redis, JPA, JWT, Spring Batch 등을 실제 서비스 흐름에 적용해보며 학습하는 것을 목표로 진행했습니다.

* 대량 트래픽 상황에서의 동시 접속 제어
* 실시간 대기열 관리
* 결제 기한 초과 및 공연 상태 변경 자동화
* 인증/인가 및 토큰 관리

---

## 프로젝트 정보

* 프로젝트명: SeatUp
* 진행 기간: 2026.03 ~ 2026.05
* 인원: 개인 프로젝트

---

## 기술 스택

| 분류 | 기술 |
|------|------|
| Language | Java 17 |
| Framework | Spring Boot 3.5.9 |
| ORM | Spring Data JPA |
| Database | MySQL |
| Cache | Redis |
| Batch | Spring Batch |
| Auth | JWT |
| Frontend | Thymeleaf |
| Payment | Toss Payments API |
| Test | K6 |

---

## 주요 기능

### 회원

- 회원가입 / 로그인 / 로그아웃
- 회원정보 수정 / 비밀번호 변경 / 회원 탈퇴
- JWT 기반 인증/인가

### 공연

- 카테고리별 공연 목록 조회
- 공연 등록 / 수정 / 삭제 (관리자)

### 좌석

- 좌석 등록 / 수정 / 삭제 / 조회 (관리자)

### 예매

* 공연 예매
* Redis 기반 예매 대기열
* 좌석 수량 동시성 제어

### 결제

- Toss Payments API 연동 결제 / 결제 취소

### 마이페이지

* 예매 내역 조회
* 회원 정보 관리

### 배치

- 티켓 예매 오픈/종료 시간에 따른 공연 상태 자동 변경
- 결제 기한 초과 예약 자동 취소

---

# Technical Challenges

## 1. Redis 기반 예매 대기열 시스템

동시 접속 상황에서 서버 부하를 줄이고 선착순 진입을 보장하기 위해 Redis 기반 대기열을 구현했습니다.
 
- **Sorted Set**: 진입 시간을 score로 저장해 대기 순번 관리
- **Set**: 현재 예매창 접속 중인 활성 사용자 관리
- **Lua Script**: 활성 사용자 수 체크와 추가를 원자적으로 처리해 Race Condition 방지
- **SSE**: 실시간 순번 알림

**Race Condition 발생 원인**
 
활성 사용자 수 조회(`SCARD`)와 추가(`SADD`) 사이에 다른 스레드가 개입하면서 설정한 입장 인원을 초과하는 문제가 발생했습니다.
 
```
Long activeCount = redisTemplate.opsForSet().size(activeKey);  // 읽기
// ← 다른 스레드 개입
redisTemplate.opsForSet().add(activeKey, token);               // 쓰기
```
 
**해결**
 
Lua Script를 사용해 두 연산을 원자적으로 묶어 처리했습니다. Redis는 단일 스레드로 동작하므로 스크립트 실행 중 다른 명령이 개입할 수 없습니다.
 
| 방식 | 문제점 |
|------|--------|
| `synchronized` | 다중 서버 환경에서 효력 없음 |
| Redisson 분산 락 | 락 획득/해제 과정에서 네트워크 왕복 추가 발생 |
| **Redis Lua Script** | Redis 단일 스레드 보장, 네트워크 왕복 1회, 다중 서버 환경 대응 |
 
**K6 부하 테스트 결과**
 
동시 접속자 200명, 활성 입장 가능 인원 50명 제한 시나리오
 
| | immediate_users | waiting_users |
|---|---|---|
| Lua Script 적용 전 | 68명 | 129명 |
| Lua Script 적용 후 | 50명 | 163명 |

---

## 2. 좌석 수량 동시성 제어

동일 좌석에 여러 사용자가 동시에 예매 요청을 보내는 상황을 처리하기 위해 `SeatGrade` 엔티티에 `@Version`을 적용해 낙관적 락을 사용했습니다.
 
비관적 락 대비 DB 락을 최소화하면서 데이터 정합성을 유지할 수 있습니다.

---

## 3. Spring Batch 기반 상태 자동화

시간 조건에 따른 반복 작업을 자동화하기 위해 Spring Batch를 적용했습니다.
 
단순 `@Scheduled`보다 Job 단위 관리, 실패 처리 및 재실행, 트랜잭션 관리 측면에서 유리하다고 판단했습니다.

---

## 4. JWT 인증

Thymeleaf 기반 SSR 환경에서는 브라우저가 URL을 직접 호출할 때 JS로 `Authorization` 헤더를 붙일 수 없었습니다. 이를 해결하기 위해 액세스 토큰을 쿠키와 localStorage에 이중 저장했습니다.

또한 회원 탈퇴 시 Redis 블랙리스트를 활용해 기존 토큰을 무효화했습니다.

---


## 환경 변수 설정
 
| 변수명 | 설명 |
|--------|------|
| `DB_PASSWORD` | MySQL 비밀번호 |
| `JWT_ACCESS_SECRET` | JWT Access 시크릿 키 |
| `JWT_REFRESH_SECRET` | JWT Refresh 시크릿 키 |
| `TOSS_SECRET_KEY` | Toss Payments 시크릿 키 |
