# Chapter 10. 운영, 보안, 실패 모드

> 좋은 IAM 설계는 무엇을 선택했는지만이 아니라 어떤 실패를 방지하려 했는지까지 설명해야 한다.

---

## 10.1 설계 질문

Keycloak을 production Identity Control Plane으로 운영할 때, 어떤 설계 결정을 명시해야 하고 어떤 실패 모드를 반드시 방어해야 하는가?

## 10.2 설계 결정 매트릭스

| # | 설계 결정 | 대안 | 선택 이유 | 대가 |
| --- | --- | --- | --- | --- |
| 1 | 중앙 IdP로 Keycloak 사용 | 앱별 인증 구현 | 보안 정책, SSO, 감사 중앙화 | Keycloak이 critical dependency가 됨 |
| 2 | Realm을 격리 단위로 사용 | client/group만으로 분리 | key, IdP, flow, user, token 정책 독립성 | realm 수 증가 시 운영 자동화 필요 |
| 3 | Authorization Code + PKCE | Implicit flow | browser code 탈취 위험 감소, 최신 표준 | client 구현 복잡도 증가 |
| 4 | Confidential client 기본 | public client 남용 | secret/private key 기반 client auth 가능 | secret rotation과 저장소 필요 |
| 5 | 짧은 access token TTL | 긴 access token TTL | 탈취 피해와 stale permission 최소화 | refresh/token endpoint 부하 증가 |
| 6 | Group/Role 중앙 관리 | 앱별 권한 DB | 권한 정책 통합과 audit 쉬움 | mapping drift와 composite role 복잡도 |
| 7 | Protocol mapper 최소화 | 모든 user attribute claim화 | PII 노출과 token bloat 방지 | resource server 추가 조회 가능성 |
| 8 | DB를 source of truth로 사용 | cache-only 정책 | durability와 transaction 확보 | DB HA/backup/migration 필요 |
| 9 | Infinispan으로 cache/session 관리 | pod-local session | multi-pod SSO와 invalidation 지원 | cache topology와 sticky session 고려 필요 |
| 10 | Quarkus optimized image | runtime dynamic server | startup predictability와 immutable 운영 | provider/theme 변경 시 rebuild 필요 |
| 11 | Operator로 Kubernetes 운영 | raw manifests only | desired state와 status reconciliation | CRD가 모든 외부 상태를 소유하지 못함 |
| 12 | test-framework 기반 신규 테스트 | legacy testsuite 중심 | 선언적 resource lifecycle과 JUnit 5 통합 | 기존 testsuite migration 비용 |

## 10.3 대표 실패 모드

| 실패 모드 | 원인 | 영향 | 방어/대응 |
| --- | --- | --- | --- |
| Redirect URI 오설정 | wildcard 또는 잘못된 callback 허용 | authorization code 탈취 가능 | exact redirect URI, environment별 client 분리 |
| Audience 검증 누락 | resource server가 signature만 검증 | 다른 API용 token 재사용 가능 | `aud`, `azp`, scope/role 검증 |
| Issuer mismatch | proxy/hostname 설정 오류 | token validation 실패 | hostname/proxy 설정과 discovery endpoint smoke test |
| 너무 긴 token TTL | UX만 고려한 lifespan 설정 | 탈취 token 장기 유효 | 짧은 access token, refresh rotation |
| Group claim 과다 노출 | 모든 group path를 token에 포함 | 조직 구조 노출, token size 증가 | mapper 최소화, optional scope 사용 |
| Composite role 남용 | role 묶음 중첩 | effective permission 추적 어려움 | role design review와 admin event audit |
| Admin 계정 공유 | break-glass/운영 계정 관리 미흡 | audit 불가 | 개인별 admin, break-glass 절차 |
| LDAP timeout | external user store 지연 | login 지연/실패 | timeout, monitoring, fallback 정책 |
| Cache split/invalidation 실패 | Infinispan topology 문제 | stale realm/user/session state | cluster health, sticky session, cache metrics |
| DB down | DB HA 부재 | login/token/admin 광범위 장애 | DB HA, backup, readiness, failover drill |
| Key rotation 실패 | JWKS cache/active key 관리 미흡 | resource server validation 실패 | staged rotation, JWKS cache TTL 관리 |
| Logout 전파 실패 | front/back-channel logout 미구성 | client session 잔존 | logout protocol별 테스트 |
| Operator drift | 수동 변경과 CR desired state 충돌 | 예상치 못한 rollback/overwrite | GitOps 원칙, status condition 모니터링 |
| Custom provider 장애 | provider exception/latency/dependency 충돌 | login/token/admin path 장애 | provider test, canary, rollback image |

### 10.3.1 Runbook 관점의 탐지와 검증

| 실패 모드 | 탐지 신호 | 즉시 조치 | 검증 기준 |
| --- | --- | --- | --- |
| Redirect URI 오설정 | authorization error, unexpected callback, admin event 변경 이력 | client redirect URI를 exact match로 축소, 의심 client disable | discovery endpoint와 정상 login callback 재검증 |
| Audience 검증 누락 | resource server 간 token 재사용 가능성, API audit 이상 | resource server에서 `aud`/scope/role 검증 추가 | 다른 client용 token으로 API 접근 실패 확인 |
| Issuer mismatch | resource server token validation 실패, discovery URL과 token `iss` 불일치 | hostname/proxy 설정과 외부 URL 정합성 수정 | `/.well-known/openid-configuration` issuer와 token issuer 일치 |
| 긴 token TTL | 탈취 의심 token이 장시간 유효, 권한 변경 반영 지연 | access token TTL 축소, refresh rotation/revocation 검토 | 권한 제거 후 신규 token claim과 기존 token 만료 window 확인 |
| Cache split/invalidation 실패 | node별 login/token behavior 차이, cluster readiness 실패 | 문제 node 격리, cache clear/restart, topology 확인 | 같은 realm/client 변경이 모든 node에서 동일하게 반영 |
| DB down/failover | readiness 실패, connection pool saturation, token/admin 오류 | DB failover, pod readiness delay, connection pool 보호 | login/token/admin smoke test와 readiness 회복 |
| LDAP/IdP timeout | login latency 증가, federation/broker error event | timeout 축소, 장애 IdP disable 또는 fallback 안내 | 외부 장애 시 Keycloak thread 고갈 없이 실패 처리 |
| Key rotation 실패 | resource server에서 JWKS kid 미발견, signature validation 실패 | active/passive key 상태 확인, JWKS cache TTL 고려 | 신규 token과 기존 token을 각 resource server가 검증 |
| Operator drift | reconcile error, status condition degraded, 수동 object 변경 | GitOps source와 live object 차이 확인, 수동 변경 회수 | CR 재적용 후 StatefulSet/status condition 정상 |
| Custom provider 장애 | 특정 flow에서 5xx, thread blocking, classpath error | provider 비활성화 또는 이전 image rollback | login/token/admin 핵심 smoke test 통과 |

## 10.4 운영 지표

| 지표 | 왜 중요한가 |
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

## 10.5 Backup과 restore는 DB만의 문제가 아니다

| 대상 | 이유 |
| --- | --- |
| Relational DB | realm, client, user, credential, event, persistent session의 source of truth |
| Realm keys | token validation continuity에 직접 영향 |
| Kubernetes Secrets | DB credential, client secret, TLS key, admin secret |
| Custom provider/theme artifacts | image 재생성과 rollback에 필요 |
| Operator CR/GitOps manifests | desired state 복구에 필요 |
| External IdP/LDAP config | broker/federation trust 복구에 필요 |

Realm export는 유용하지만 DB backup의 완전한 대체가 아니다. 운영 복구는 DB, key material, secret, provider artifact, CR desired state가 함께 맞아야 성공한다.

## 10.6 Security hardening checklist

| 영역 | 기준 |
| --- | --- |
| Client redirect | wildcard 최소화, environment별 client 분리 |
| Public client | Authorization Code + PKCE 강제 |
| Confidential client | secret/private key rotation, secret manager 사용 |
| Token mapper | 최소 claim, PII 노출 제한, audience 명시 |
| Realm key | staged rotation, JWKS cache 고려 |
| Admin | 개인별 admin, break-glass 계정, admin event retention |
| Password/MFA | password policy, WebAuthn/OTP, brute force protection |
| Federation | LDAPS, timeout, sync/deprovisioning policy |
| Proxy | hostname/issuer/proxy header trust 검증 |
| Provider | dependency scan, code review, canary rollout |
| Operator | RBAC 최소화, secret reference, network policy |

## 10.7 소스코드 증거

| 주장 | 근거 파일 |
| --- | --- |
| DB readiness는 Keycloak 운영 health의 핵심 신호다 | `quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/services/health/KeycloakReadyHealthCheck.java` |
| cluster/cache readiness는 별도 health check로 다뤄진다 | `quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/services/health/KeycloakClusterReadyHealthCheck.java` |
| bootstrap readiness와 load balancer probe resource가 startup/traffic 경계를 만든다 | `quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/services/health/BootstrapReadyHealthCheck.java`, `services/src/main/java/org/keycloak/services/resources/LoadBalancerResource.java` |
| brute force protection은 login failure state와 event를 기반으로 동작한다 | `services/src/main/java/org/keycloak/services/managers/DefaultBruteForceProtector.java`, `services/src/main/java/org/keycloak/services/managers/DefaultBlockingBruteForceProtector.java` |
| CORS와 origin 검증은 browser trust boundary의 일부다 | `services/src/main/java/org/keycloak/services/cors/DefaultCors.java` |
| 전역 error handler는 transaction rollback과 OAuth2/HTML error response를 정리한다 | `services/src/main/java/org/keycloak/services/error/KeycloakErrorHandler.java` |
| hostname/proxy 설정은 issuer와 외부 URL 정합성에 직접 연결된다 | `quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/configuration/mappers/HostnameV2PropertyMappers.java`, `services/src/main/java/org/keycloak/url/HostnameV2Provider.java` |
| truststore 설정은 외부 LDAP/IdP/DB/TLS trust boundary를 구성한다 | `services/src/main/java/org/keycloak/truststore/TruststoreBuilder.java`, `services/src/main/java/org/keycloak/truststore/FileTruststoreProviderFactory.java`, `quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/configuration/mappers/TruststorePropertyMappers.java` |
| health/metrics/cache/database option은 Quarkus config API와 mapper로 노출된다 | `quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/configuration/mappers/HealthPropertyMappers.java`, `quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/configuration/mappers/MetricsPropertyMappers.java`, `quarkus/config-api/src/main/java/org/keycloak/config/DatabaseOptions.java`, `quarkus/config-api/src/main/java/org/keycloak/config/CachingOptions.java` |
| Operator reconcile/update 상태는 Kubernetes 운영 실패 탐지의 일부다 | `operator/src/main/java/org/keycloak/operator/controllers/KeycloakController.java`, `operator/src/main/java/org/keycloak/operator/update/impl/AutoUpdateLogic.java` |

## 10.8 운영자가 결정할 것

| 결정 | 질문 | 영향 |
| --- | --- | --- |
| SLO boundary | login, token, admin, account, JWKS 중 어떤 endpoint를 SLO로 볼 것인가? | alert와 smoke test 범위 |
| Token validation contract | resource server가 issuer/audience/scope/role을 어떻게 검증할 것인가? | API 간 token replay 방어 |
| Break-glass | emergency admin을 어떻게 발급, 보관, 감사할 것인가? | 장애 대응과 audit 균형 |
| Backup scope | DB 외에 key, secret, CR, provider artifact를 어디까지 백업할 것인가? | 재해 복구 성공 여부 |
| Incident drill | DB failover, cache split, IdP outage, key rotation 실패를 얼마나 자주 연습할 것인가? | runbook 실효성 |
| Change approval | realm/client/mapper/flow/provider 변경을 어떤 pipeline으로 승인할 것인가? | misconfiguration과 권한 상승 방지 |

## 10.9 백서 관점의 최종 결론

Keycloak을 성공적으로 도입한다는 것은 Admin Console에서 realm과 client를 만드는 일을 뜻하지 않는다. 진짜 설계 과제는 조직의 신뢰 경계를 모델링하고, 어떤 identity source를 신뢰할지 정하며, token에 어떤 claim을 실을지 결정하고, session과 cache의 consistency를 운영하며, DB와 Operator와 custom provider의 lifecycle을 하나의 production system으로 맞추는 것이다.

Keycloak의 강점은 표준 protocol과 확장성이다. 그러나 같은 이유로 Keycloak은 운영자가 아무렇게나 설정해도 안전한 magic box가 아니다. Realm, Client, Role, Mapper, Flow, Provider, Cache, DB, Operator는 모두 명시적인 설계 결정이다. 좋은 Keycloak 운영은 이 결정들을 문서화하고, 대안을 비교하고, 실패 모드를 테스트하고, 변경을 audit 가능한 pipeline으로 만드는 데서 완성된다.

## 10.10 이 챕터의 핵심 인사이트

1. Keycloak 운영의 핵심은 기능 활성화가 아니라 실패 모드 제거다.
2. DB, cache, key, secret, CR, provider artifact는 함께 백업/복구되어야 한다.
3. Token validation 실패의 상당수는 code bug가 아니라 issuer/audience/hostname/proxy 설정 오류에서 나온다.
4. 좋은 IAM 운영은 설계 결정, audit, test, rollback이 연결된 pipeline이다.
5. Production runbook은 탐지 신호, 즉시 조치, 검증 기준까지 포함해야 한다.

---

| 방향 | 문서 |
| --- | --- |
| 이전 | [Ch.9 Operator, 테스트, 배포 모델](ch09-operator-test-delivery.md) |
| 백서 색인 | [WHITEPAPER.md](../WHITEPAPER.md) |
