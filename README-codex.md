# README-codex — Nearby Friends (Spring Boot, Java, Gradle)

## Цель
Сгенерировать систему “друзья рядом”:
- ws-gateway (WebSocket/STOMP)
- api-service (friends/profile)
- location-cache (Redis)
- pubsub (Redis Pub/Sub или Kafka)
- optional history store
- load-test симулятор клиентов

## Стек
- Java 17+, Spring Boot 3.x, Gradle
- WebSocket: spring-boot-starter-websocket (STOMP) или raw WS
- Redis: cache + pubsub
- User DB: Postgres
- Observability: actuator + micrometer

## Модули
/modules
  /contracts
  /api-service
  /ws-gateway
  /load-test-client
/infra/docker-compose.yml
/docs (C4)

## Контракты
### WS messages
Client -> Server:
- `location_update`: {userId, lat, lng, ts}
Server -> Client:
- `friend_update`: {friendId, lat, lng, ts, distanceMeters}

### HTTP
- GET /v1/friends
- POST /v1/friends/{friendId}
- DELETE /v1/friends/{friendId}

## Реализация ws-gateway
- Аутентификация: JWT (handshake interceptor)
- При connect:
  - загрузить friendIds из api-service или userdb
  - подписать пользователя на каналы обновлений друзей
- При location_update:
  - записать последнюю локацию в Redis с TTL (например, 10 минут)
  - publish update в Redis Pub/Sub (канал friendId или geohash)
- При получении update:
  - вычислить расстояние (Haversine)
  - если within radius — отправить friend_update

## Масштабирование (практичные хуки)
- Sticky sessions (LB) по userId для WS
- Backpressure/rate limiting на updates
- Degrade: если pubsub перегружен — клиент периодически делает pull `GET /v1/nearby` (optional endpoint)

## Тесты
- Unit: distance calc, TTL logic
- Integration: Testcontainers Redis+Postgres; два клиента WS получают обновления
- Load-test: симуляция N пользователей, измерение latency доставки

## Порядок генерации
1) infra: redis + postgres.
2) api-service: friendship CRUD + auth stub.
3) ws-gateway: websocket endpoints + redis integration.
4) load-test-client: java app, много WS соединений, отправка updates.
5) Observability: metrics (connected sessions, msg/sec, delivery latency).
