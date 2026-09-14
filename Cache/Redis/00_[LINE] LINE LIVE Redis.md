## Pub/Sub - 서버 간 메시지 중계
Line Live는 채팅 서버가 100대 이상. 같은 채팅방 유저라도 서로 다른 서버에 접속. 그럼 Chat Server1에 붙은 유저 코멘트를
Chat Server2에 붙은 유저에게 어떻게 전달할까?

-> 여기서 Redis의 Pub/Sub 기능이 쓰임. 방송국 라디오와 똑같은 구조. 

- 각 채팅 서버는 자기가 담당하는 채팅방 채널을 subscribe(구독)
- 코멘트가 들어오면 그 서버가 Redis에 Publish(발행)
- Redis가 그 채널을 구독 중인 모든 서버에 메시지를 뿌려주고, 각 서버는 자기한테 붙은 클라이언트들에게 WebSocket으로 내려보냄

즉 서버끼리 직접 통신하지 않고 Redis를 중앙 메시지 허브로 쓴다. 글에서 Akka Cluster 같은 대안 대신 Redis Pub/Sub을 고른 이유로
"운영과 구현이 간편해서"를 들었는데. 이게 Pub/Sub의 강점.

"실시간 채팅처럼 지금 못 받은 건 놓쳐도 되는" 용도에 사용, 유실되면 안되는 메시지에는 kafka 같은 메시지 큐를 쓴다.

## 고속 kvs - 임시 저장소 (쓰기 버퍼)
분당 1만 건씩 쏟아지는 코멘트를 매번 MySQL에 INSERT하면 DB가 버티기 힘듬. 그래서 이런 흐름 을씀
> 방송중 : 코멘트/기프트를 Redis에 저장 (메모리라 빠름) -> 방송이 끝나면: 정리해서 MySQL로 옮김

Redis가 빠르지만 임시, MySQL이 느리지만 영구 라는 역할 분담. Redis를 DB 앞단의 완충 장치로 쓰는 전형적인 패턴.

Sorted Set이라는 자료구조, Redis 배울 때 꼭 알아야 하는 것 중 하나. 일반 Set과 달리 각 원소에 Score를 붙여 자동정렬.

### Redis Cluster를 쓴 이유
Line Live는 코멘트 유입량이 단일 Redis 한 대로 감잗이 안되는 규모. 데이터를 여러 노드에 나누고(확장성) 노드가 죽어도 failover로
버티는(가용성) 클러스터를 택함. 

### Lettuce 채택 이유
- 비동기 API : akka actor 안에서 블로킹하면 스레드가 고갈되기 때문에, 명령을 보내놓고 기다리지 않는 비동기 클라이언트가 필수. Jedis 탈락.
- 클러스터 지원 + MOVED/ASK 리다이렉트 처리 : 첫 대화에서 설명했던 "키가 어느 노드에 있는지 슬롯 맵을 캐싱하고, 틀리면 MOVED 응답 따라가는" 그 동작을 Lettuce가 알아서 해준다.
- subscribe 커넥션의 failover : pub/sub 구독은 방송 내내 커넥션을 유지해야 하는데, 그 커넥션이 붙어있던 노드가 죽으면 자동으로 재연결 해줘야 함. 이게 안되면 그 서버의 유저들은 코멘트를 못 받음
---



















---
## 관련 기술 블로그
[LINE LIVE 채팅 기능의 기반이 되는 아키텍처]
https://engineering.linecorp.com/ko/blog/the-architecture-behind-chatting-on-line-live

[올영세일 선착순 쿠폰, 미발급 0%를 향한 여정
Redis와 Message Queue로 구축한 비동기 시스템의 정합성 개선기]
https://oliveyoung.tech/2025-12-15/fcfs-coupon/

[개발자가 알면 좋은 Redis 꿀팁 모음
실무에서 바로 쓰는 Redis 핵심 팁 공유]
https://oliveyoung.tech/2025-07-23/redis-tips-for-developer/

[Atomic 처리와 cache stampede 대책을 위해 Redis Lua script를 활용한 이야기]
https://engineering.linecorp.com/ko/blog/atomic-cache-stampede-redis-lua-script