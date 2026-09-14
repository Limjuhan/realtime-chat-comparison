# Realtime Chat Comparison: HTTP Long Polling vs WebSocket

동일한 1:1 채팅 기능을 HTTP Long Polling과 WebSocket(STOMP) 두 가지 방식으로 각각 구현하고,
동작 방식·성능·구현 복잡도를 데이터와 코드로 비교 분석하는 프로젝트입니다.

## 목적

"WebSocket이 무조건 좋다"가 아니라, **왜, 어떤 상황에서** 어떤 기술이 유리한지를
실제 구현과 부하 테스트 결과로 증명합니다.

## 구조

```
realtime-chat-comparison/
├── common/               # 공통 도메인/DTO (ChatRoom, Message)
├── long-polling-chat/    # DeferredResult 기반 Long Polling 채팅 서버
├── websocket-chat/       # STOMP 기반 WebSocket 채팅 서버
├── load-test/            # JMeter/k6 부하 테스트 스크립트 및 결과
└── docs/                 # 아키텍처 문서, 벤치마크 결과
```

## 기술 스택

- Java 17, Spring Boot 3.x, Gradle
- Long Polling: Spring MVC `DeferredResult`
- WebSocket: Spring WebSocket + STOMP
- H2 (개발) → PostgreSQL (선택)
- 부하 테스트: Apache JMeter 또는 k6

## 진행 상황

- [x] Phase 0: 레포 구조 세팅
- [ ] Phase 1: Long Polling 채팅 구현
- [ ] Phase 2: WebSocket 채팅 구현
- [ ] Phase 3: 비교 실험 설계 및 측정
- [ ] Phase 4: 문서화 및 정리

자세한 계획은 [docs/architecture.md](docs/architecture.md) 참고.

## 비교 결과

측정 완료 후 [docs/benchmark-results.md](docs/benchmark-results.md)에 정리 예정.
