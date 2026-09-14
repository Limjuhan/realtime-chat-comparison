# 설계 계획서

구현을 시작하기 전에 "무엇을 만들지, 왜 그렇게 설계했는지"를 먼저 정리한다.
실제 구현 중 겪은 것과 배운 점은 이 문서가 아니라 [learning-log.md](learning-log.md)에
날짜별로 기록한다.

## 1. 학습 목표

이 프로젝트를 통해 다음을 직접 구현하며 이해하는 것이 목표다.

- HTTP Long Polling이 실제로 어떻게 "연결을 유지"하는지 — Spring MVC
  `DeferredResult`가 서블릿 스레드를 반환하고 별도 스레드에서 대기하는 구조,
  그리고 이게 왜 "스레드 낭비 없는 비동기 처리"로 불리는지
- WebSocket(STOMP) 핸드셰이크 이후 지속 연결이 어떻게 유지되고,
  `@MessageMapping`/`@SendTo` 라우팅이 내부적으로 어떻게 동작하는지
- 두 방식의 실제 성능 차이(지연시간, 서버 자원 사용량, 동시 접속 처리량)를
  체감이 아니라 부하 테스트 수치로 확인하는 것
- 각 방식의 에러/재연결 처리 난이도 차이를 직접 코드로 겪어보는 것

즉 "WebSocket이 더 낫다"는 결론을 확인하는 게 아니라, **그 결론이 왜, 어떤
조건에서 성립하는지를 데이터로 설명할 수 있게 되는 것**이 최종 목표다.

## 2. 전체 아키텍처

### 2.1 HTTP Long Polling

```
Client                          Server
  |-- GET /messages/poll ------->|
  |          (연결 유지, 대기)     |  (새 메시지 있을 때까지 대기)
  |<----- 응답 (새 메시지) --------|
  |-- 즉시 재요청 ---------------->|
```

- `DeferredResult` 또는 `CompletableFuture`로 요청을 일정 시간(예: 30초) 보류
- 새 메시지가 생기면 즉시 응답, 없으면 타임아웃 후 빈 응답 반환 → 클라이언트가 재요청
- **왜 `DeferredResult`인가**: 서블릿 스레드를 점유한 채 `Thread.sleep`으로
  대기하는 방식은 동시 접속자 수만큼 스레드가 blocking되어 Tomcat 스레드 풀이
  금방 고갈된다. `DeferredResult`는 요청 처리를 다른 스레드로 넘기고 서블릿
  스레드는 즉시 반환하므로, 대기 중인 연결이 많아도 스레드 풀 자체는 점유되지
  않는다. 이 차이를 실제로 스레드 덤프로 확인하는 것이 Phase 3 실험의 목표 중
  하나다.

### 2.2 WebSocket (STOMP)

```
Client                          Server
  |-- WS Handshake (Upgrade) --->|
  |<---- 101 Switching Protocol -|
  |<===== 양방향 지속 연결 =======>|
  |-- 메시지 전송 ----------------|
  |<---- 메시지 수신 --------------|
```

- 최초 핸드셰이크 이후 지속 연결 유지
- `@MessageMapping`, `@SendTo` 기반 메시지 라우팅
- **왜 STOMP인가**: 순수 WebSocket(`TextWebSocketHandler`)만 써도 구현은
  가능하지만, pub/sub 패턴(특정 채팅방 구독)과 세션 관리를 직접 다 만들어야
  한다. STOMP는 이걸 표준화해두어서 실무 코드에 더 가깝고, "구현 복잡도" 비교
  항목에서 순수 WebSocket과의 차이도 부가적으로 관찰할 수 있다.

## 3. 도메인 설계

두 구현이 동일한 데이터 모델을 공유해야 통신 방식 차이만 순수하게 비교할 수 있다.

### 3.1 ChatRoom

| 필드 | 타입 | 설명 |
|---|---|---|
| id | Long | PK |
| user1Id | Long | 참여자 1 |
| user2Id | Long | 참여자 2 |
| createdAt | LocalDateTime | 생성 시각 |

- **1:1로 범위 고정**: 그룹 채팅까지 지원하려면 참여자 테이블을 분리해야 하지만,
  이 프로젝트의 목적은 통신 방식 비교이지 채팅 도메인의 확장성이 아니다.
- **user1Id/user2Id 순서 문제**: 방 생성 시 `min(userId), max(userId)` 순으로
  정렬해서 저장 → 조회 쿼리가 단순해지고, 유니크 제약(`user1Id, user2Id`) 하나로
  중복 방 생성을 막을 수 있다.
- **id는 Long(auto-increment)**: UUID는 분산 환경에 유리하지만 이 프로젝트는
  단일 서버 비교 실험이 목적이라 불필요한 복잡도다.

### 3.2 Message

| 필드 | 타입 | 설명 |
|---|---|---|
| id | Long | PK |
| chatRoomId | Long | 소속 채팅방 FK |
| senderId | Long | 보낸 사람 |
| content | String | 메시지 내용 |
| sentAt | LocalDateTime | 전송 시각 |
| status | Enum (SENT, DELIVERED, READ) | 메시지 상태 |

- **status를 3단계로 제한**: 세밀한 상태 추적은 범위 밖이지만, 완전히 없애면
  "메시지 상태 갱신"이라는 두 통신 방식이 실제로 다르게 처리하는 지점(WebSocket은
  즉시 push, Long Polling은 다음 polling까지 지연)을 비교할 수 없다.
- **(chatRoomId, sentAt) 복합 인덱스**: 메시지 목록 조회와, Long Polling에서
  "마지막으로 받은 시각 이후 메시지"를 찾는 쿼리 모두 이 인덱스에 의존한다.
- **content는 평문 String**: 암호화, 첨부파일 등은 통신 방식 비교라는 핵심
  목적과 무관해 의도적으로 배제한다.

### 3.3 DTO

Entity를 API 응답에 직접 노출하지 않고 분리한다.

- `MessageSendRequest` (senderId, content)
- `MessageResponse` (id, senderId, content, sentAt, status)
- `ChatRoomResponse` (id, user1Id, user2Id, createdAt)

**근거**: Entity를 그대로 직렬화하면 JPA 연관관계 추가 시 순환 참조나 지연
로딩(LAZY) 예외가 API 레이어까지 새어나갈 위험이 있다.

### 3.4 코드 공유 전략: 복사 방식 채택

| 방식 | 장점 | 단점 |
|---|---|---|
| A. `common` Gradle 모듈로 분리 | 중복 없음 | 멀티모듈 설정 필요, 두 앱의 "완전한 독립 실행" 인상이 약해짐 |
| B. 각 앱에 동일 코드 복사 (채택) | 각 앱이 완전히 독립적인 Spring Boot 프로젝트 | 도메인 변경 시 두 곳 다 수정 필요 |

도메인은 3장에서 이미 고정되어 거의 바뀌지 않으므로 B안의 중복 관리 비용은
작다. 반면 각 서버가 독립 실행 가능한 편이, 학습 기록이자 포트폴리오로서
"각각의 구현을 완결된 프로젝트로 이해했다"는 걸 보여주기에 더 적합하다.
`common/`은 실제 모듈이 아니라 참조용 원본을 두는 용도로만 쓴다.

## 4. 구현체별 세부 설계 계획

### 4.1 Long Polling (`long-polling-chat`)

- 엔드포인트: `GET /api/rooms/{roomId}/messages/poll?after={lastMessageId}`
- 서버는 `after` 이후 새 메시지가 있으면 즉시 응답, 없으면 `DeferredResult`로
  최대 30초 대기 후 빈 리스트 반환
- 새 메시지 도착을 대기 중인 `DeferredResult`에 알리는 방법: 메시지 저장 시
  해당 채팅방을 구독 중인 `DeferredResult`를 찾아 `setResult()` 호출 (in-memory
  `Map<roomId, List<DeferredResult>>` 로 우선 구현, 다중 서버 확장은 범위 밖)
- 타임아웃 처리: `DeferredResult.onTimeout()`으로 빈 리스트 반환, 클라이언트는
  받는 즉시 재요청
- 클라이언트: JS `fetch` 재귀 호출 (응답 오면 바로 다음 poll 요청)

### 4.2 WebSocket (`websocket-chat`)

- `/ws` 엔드포인트에 SockJS + STOMP 설정
- 구독 경로: `/topic/rooms/{roomId}`
- 발행 경로: `/app/rooms/{roomId}/send` → `@MessageMapping`에서 저장 후
  `@SendTo`로 브로드캐스트
- 연결 끊김 감지: `SessionDisconnectEvent` 리스너로 로그 남기고, 클라이언트는
  STOMP.js의 자동 재연결 옵션 사용
- 세션-사용자 매핑: WebSocket 세션 ID와 userId를 연결하는 `Map` 유지 (읽음
  처리 등에서 필요시 사용)

## 5. 측정/비교 계획

| 지표 | 측정 방법 | 예상 결과 방향 |
|---|---|---|
| 메시지 전달 지연시간 (latency) | 전송 시각 - 수신 시각 | WebSocket이 낮음 |
| 서버 CPU/메모리 사용률 | Actuator + 부하 중 모니터링 | Long Polling이 높음 |
| 동시 접속자당 스레드/커넥션 수 | 서버 로그, JVM 스레드 덤프 | 4.1에서 언급한 `DeferredResult`의 효과를 여기서 직접 확인 |
| 초당 처리 가능한 요청 수 (throughput) | JMeter/k6 결과 | WebSocket 우세 예상 |
| 불필요한 네트워크 트래픽 | 응답 없는 polling 요청 비율 | Long Polling에서 발생 |
| 구현 복잡도 | 코드 라인 수, 에러 핸들링 케이스 수 | 정성적 비교 |
| 연결 끊김 복구 용이성 | 강제 종료 후 재연결 테스트 | 정성적 비교 |

동시 접속자 수를 10 → 100 → 1000으로 늘려가며 측정한다. 두 서버는 동일 스펙
(같은 머신, 같은 JVM 힙 설정)에서 테스트해야 비교가 의미 있다.

결과가 항상 "WebSocket 승리"로 끝나지 않을 수도 있다 — 동시 접속자가 적고
메시지 빈도가 낮다면 Long Polling의 구현 단순함이 오히려 실용적일 수 있다.
이런 조건부 결론을 [benchmark-results.md](benchmark-results.md)에 남긴다.

## 6. 범위 밖 (향후 개선 방향)

- SSE(Server-Sent Events)를 3번째 비교군으로 추가
- WebSocket 다중 서버 스케일아웃 (Redis Pub/Sub, RabbitMQ STOMP broker relay)
- 그룹 채팅, 메시지 암호화, 첨부파일
