# Realtime Chat Comparison: HTTP Long Polling vs WebSocket

HTTP Long Polling과 WebSocket(STOMP), 두 가지 실시간 통신 방식을 직접 구현해보며
동작 원리를 이해하기 위한 학습 프로젝트입니다. 동일한 1:1 채팅 기능을 두 방식으로
각각 구현하고, 동작 방식·성능·구현 복잡도를 코드와 부하 테스트 데이터로 비교합니다.

## 목적

"WebSocket이 더 낫다"는 결론을 확인하는 게 아니라, **왜, 어떤 상황에서** 어떤
기술이 유리한지 직접 구현하고 측정해서 데이터와 코드로 설명할 수 있게 되는 것이
목표입니다. 학습 목표를 포함한 구현 전 설계 근거는
[docs/design-plan.md](docs/design-plan.md)에, 구현하면서 겪은 것과 배운 점은
[docs/learning-log.md](docs/learning-log.md)에 기록합니다.

## 구조

```
realtime-chat-comparison/
├── common/               # 공통 도메인/DTO (ChatRoom, Message)
├── long-polling-chat/    # DeferredResult 기반 Long Polling 채팅 서버
├── websocket-chat/       # STOMP 기반 WebSocket 채팅 서버
├── load-test/            # JMeter/k6 부하 테스트 스크립트 및 결과
└── docs/                 # 설계 계획, 학습 기록, 벤치마크 결과
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

## 비교 결과

측정 완료 후 [docs/benchmark-results.md](docs/benchmark-results.md)에 정리 예정.
