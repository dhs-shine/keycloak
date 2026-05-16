# Chapter 8. 보안, 감사, 미결 결정과 로드맵

> "보안 운영에서 가장 중요한 질문은 '왜 허용되었는가'와 '누가 바꿨는가'입니다."

좋은 IAM 운영은 기능 목록을 많이 켜는 일이 아닙니다. 좋은 IAM 운영은 실패 모드를 줄이고, 변경을 설명 가능하게 만들고, 사고가 났을 때 복구할 수 있게 만드는 일입니다. Keycloak은 강력한 표준 프로토콜과 확장성을 제공하지만, 아무렇게나 설정해도 안전한 마법 상자는 아닙니다.

이 마지막 장은 앞선 장의 결정을 하나의 production operating model로 묶고, 우리가 아직 결정해야 할 질문을 정리합니다.

---

## 8.1 우리가 이미 이해한 것들

우리는 Keycloak을 다음과 같이 읽었습니다.

| 영역 | 핵심 이해 |
| --- | --- |
| 설계 철학 | 신뢰는 중앙에서 선언하고, 서비스는 검증 가능한 증거를 소비합니다. |
| 신뢰 경계 | Browser, App, Keycloak, DB, cache, IdP, Operator마다 서로 다른 위험이 있습니다. |
| 정책 모델 | Realm은 보안 구역, Client는 신뢰 계약, Role은 권한 주장입니다. |
| 실행 흐름 | 로그인은 token, session, cache, DB가 결합된 lifecycle입니다. |
| Federation | 외부 신원은 받아들이되 local authorization은 다시 결정해야 합니다. |
| 확장성 | SPI/provider는 강력하지만 server process 내부 신뢰 경계로 들어옵니다. |
| 운영 | Operator, image, DB, cache, secret, IdP가 함께 맞아야 안정적입니다. |

이 기반 위에서 운영자는 “어떤 실패를 반드시 막을 것인가”를 정해야 합니다.

---

## 8.2 대표 실패 모드

| 실패 모드 | 원인 | 영향 | 기본 방어 |
| --- | --- | --- | --- |
| Redirect URI 오설정 | wildcard 또는 잘못된 callback 허용 | authorization code 탈취 | exact redirect URI, 환경별 client 분리 |
| Audience 검증 누락 | API가 signature만 검증 | 다른 API용 token 재사용 | `aud`, `azp`, scope/role 검증 |
| Issuer mismatch | hostname/proxy 설정 오류 | token validation 실패 | discovery endpoint smoke test |
| 긴 token TTL | UX만 고려한 lifespan | 탈취 token 장기 유효 | 짧은 access token, refresh rotation |
| Group claim 과다 노출 | 모든 group path를 token에 포함 | 조직 구조 노출, token size 증가 | mapper 최소화, optional scope |
| Composite role 남용 | role 묶음 중첩 | effective permission 추적 어려움 | role design review |
| LDAP timeout | external user store 지연 | login 지연/실패 | timeout, monitoring, fallback 정책 |
| Cache split | Infinispan topology 문제 | stale session/realm state | cluster health와 cache metrics |
| DB down | DB HA 부재 | login/token/admin 광범위 장애 | DB HA, readiness, failover drill |
| Key rotation 실패 | JWKS cache와 active key 관리 미흡 | resource server validation 실패 | staged rotation과 JWKS cache TTL 관리 |
| Operator drift | 수동 변경과 CR 충돌 | 예상치 못한 rollback/overwrite | GitOps와 status condition 모니터링 |
| Custom provider 장애 | provider exception/latency | login/token/admin path 장애 | canary, timeout, rollback image |

실패 모드는 겁주기 위한 목록이 아닙니다. 테스트와 alert와 runbook을 만들기 위한 checklist입니다.

---

## 8.3 Runbook 관점의 탐지와 검증

| 실패 모드 | 탐지 신호 | 즉시 조치 | 검증 기준 |
| --- | --- | --- | --- |
| Issuer mismatch | resource server token validation 실패 | hostname/proxy 설정 수정 | token `iss`와 discovery issuer 일치 |
| Audience 누락 | 다른 client token으로 API 접근 가능 | API audience 검증 추가 | 잘못된 audience token 접근 실패 |
| DB failover | readiness 실패, connection pool saturation | failover와 readiness delay 적용 | login/token/admin smoke test 회복 |
| Cache split | node별 login/token behavior 차이 | 문제 node 격리, cache health 확인 | 같은 설정 변경이 모든 node에서 반영 |
| LDAP/IdP outage | login latency와 broker error 증가 | timeout 축소, 장애 IdP disable 검토 | thread 고갈 없이 실패 처리 |
| Key rotation 실패 | JWKS `kid` 미발견 | active/passive key 상태 확인 | 기존 token과 신규 token 모두 검증 |
| Provider 장애 | 특정 flow 5xx, blocking 증가 | 이전 image rollback 또는 provider disable | 핵심 login/token/admin flow 통과 |

좋은 runbook은 “무엇을 봐야 하는가”, “지금 무엇을 할 것인가”, “복구되었는지 어떻게 알 것인가”를 모두 포함합니다.

---

## 8.4 운영 지표와 감사

| 지표 / 기록 | 왜 중요한가 |
| --- | --- |
| login success/error rate | 인증 장애와 brute force 공격 탐지 |
| token endpoint latency/error | application authentication path SLO |
| DB connection pool saturation | Keycloak 전체 장애의 선행 신호 |
| Infinispan cluster health | session/cache consistency 확인 |
| external LDAP/IdP latency | federation/broker login 장애 탐지 |
| admin event volume | 운영 변경과 audit 추적 |
| failed login/login failure cache | 공격 또는 UX 문제 탐지 |
| JVM memory/GC/thread | pod sizing과 custom provider leak 탐지 |
| Operator reconcile errors | Kubernetes desired state drift 탐지 |

감사의 핵심 질문은 두 가지입니다. “왜 허용되었는가?”와 “누가 바꿨는가?” 전자는 token claim, role mapping, client scope, resource server 검증으로 답합니다. 후자는 admin event, GitOps history, Operator status, change approval로 답합니다.

---

## 8.5 앞으로 넘어야 할 산들

| 항목 | 현재 권장 | 확정 필요 |
| --- | --- | --- |
| realm 분리 기준 | 환경/tenant/조직 요구에 따라 명시적으로 선택 | 실제 운영 조직과 tenant 모델 |
| role naming convention | `<app>:<resource>:<action>` 형태 후보 | 앱팀과 보안팀 합의 |
| token lifetime | 짧은 access token + 제한된 refresh token | 사용자 UX와 보안 정책 |
| admin 권한 위임 | 개인별 admin + 최소 권한 role | 운영팀 R&R과 break-glass 절차 |
| user source of truth | HR/LDAP/IdP/Keycloak ownership 분리 | 계정 lifecycle 정책 |
| event/audit sink | DB event store + 외부 SIEM 후보 | retention, privacy, 검색 요구사항 |
| cache/session 전략 | embedded + sticky 우선, 필요 시 remote 검토 | multi-AZ/multi-site 요구사항 |
| 배포 방식 | Operator 또는 GitOps hybrid | 조직의 Kubernetes 운영 표준 |

로드맵은 기능 wishlist가 아닙니다. 운영자가 사고를 더 빨리 설명하고, 더 안전하게 되돌리고, 더 적은 수작업으로 반복하기 위한 결정 목록입니다.

---

## 8.6 코드로 확인하는 증거

| 주장 | 확인할 파일 |
| --- | --- |
| DB readiness는 운영 health의 핵심 신호다 | `quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/services/health/KeycloakReadyHealthCheck.java` |
| cluster/cache readiness는 별도 health check로 다뤄진다 | `quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/services/health/KeycloakClusterReadyHealthCheck.java` |
| brute force protection은 login failure state와 event를 기반으로 동작한다 | `services/src/main/java/org/keycloak/services/managers/DefaultBruteForceProtector.java` |
| CORS와 origin 검증은 browser trust boundary의 일부다 | `services/src/main/java/org/keycloak/services/cors/DefaultCors.java` |
| hostname/proxy 설정은 issuer와 외부 URL 정합성에 연결된다 | `quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/configuration/mappers/HostnameV2PropertyMappers.java` |
| health/metrics/cache/database option은 Quarkus config API와 mapper로 노출된다 | `quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/configuration/mappers/HealthPropertyMappers.java`, `quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/configuration/mappers/MetricsPropertyMappers.java`, `quarkus/config-api/src/main/java/org/keycloak/config/DatabaseOptions.java`, `quarkus/config-api/src/main/java/org/keycloak/config/CachingOptions.java` |
| Operator reconcile/update 상태는 Kubernetes 운영 실패 탐지의 일부다 | `operator/src/main/java/org/keycloak/operator/controllers/KeycloakController.java`, `operator/src/main/java/org/keycloak/operator/update/impl/AutoUpdateLogic.java` |

---

## 8.7 최종 요약: 신뢰를 설명할 수 있는 플랫폼

Keycloak을 성공적으로 도입한다는 것은 Admin Console에서 realm과 client를 만드는 일이 아닙니다. 진짜 목표는 조직의 신뢰 경계를 설명 가능하게 만드는 것입니다.

누가 로그인했는가. 어떤 IdP를 통해 들어왔는가. 어떤 client가 어떤 scope로 token을 받았는가. 왜 이 API가 token을 받아들였는가. 누가 mapper를 바꿨는가. 장애가 났을 때 어디까지 되돌릴 수 있는가.

이 질문들에 답할 수 있다면 Keycloak은 단순 로그인 서버가 아니라 신뢰 운영 플랫폼이 됩니다.

---

## 8.8 핵심 인사이트

1. **좋은 IAM 운영은 실패 모드 제거입니다.** redirect, issuer, audience, TTL, cache, DB, provider 장애를 명시적으로 다뤄야 합니다.
2. **감사는 설명 가능성입니다.** “왜 허용되었는가”와 “누가 바꿨는가”를 연결해야 합니다.
3. **로드맵은 운영 결정 목록입니다.** realm, role, token, admin, audit, cache, 배포 기준을 조직적으로 확정해야 합니다.

---

| 방향 | 문서 |
| --- | --- |
| **이전 챕터** | [Ch.7 Release, Operator, 운영 안정성](./ch07-release-and-operations.md) |
| **백서 홈** | [WHITEPAPER.md](../WHITEPAPER.md) |
