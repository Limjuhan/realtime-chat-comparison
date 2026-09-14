# 아키텍처 설계

## 공통 도메인

```
ChatRoom (1:1)
 ├── id
 ├── user1Id, user2Id
 └── createdAt

Message
 ├── id
 ├── chatRoomId
 ├── senderId
 ├── content
 ├── sentAt
 └── status (SENT, DELIVERED, READ)
```

## HTTP Long Polling

```
Client                          Server
  |-- GET /messages/poll ------->|
  |          (연결 유지, 대기)     |  (새 메시지 있을 때까지 대기)
  |<----- 응답 (새 메시지) --------|
  |-- 즉시 재요청 ---------------->|
```

- `DeferredResult` 또는 `CompletableFuture`로 요청을 일정 시간(예: 30초) 보류
- 새 메시지가 생기면 즉시 응답, 없으면 타임아웃 후 빈 응답 반환 → 클라이언트가 재요청

## WebSocket (STOMP)

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

## 비교 측정 지표

| 지표 | 측정 방법 |
|---|---|
| 메시지 전달 지연시간 | 전송 시각 - 수신 시각 |
| 서버 CPU/메모리 사용률 | Actuator + 부하 중 모니터링 |
| 동시 접속자당 스레드/커넥션 수 | 서버 로그, JVM 스레드 덤프 |
| 초당 처리 가능한 요청 수 (throughput) | JMeter/k6 결과 |
| 불필요한 네트워크 트래픽 | 응답 없는 polling 요청 비율 |
| 구현 복잡도 | 코드 라인 수, 에러 핸들링 케이스 수 |
| 연결 끊김 복구 용이성 | 강제 종료 후 재연결 테스트 |

## 향후 개선 방향 (범위 외)

- SSE를 3번째 비교군으로 추가
- WebSocket 다중 서버 스케일아웃 (Redis Pub/Sub, RabbitMQ STOMP broker relay)
