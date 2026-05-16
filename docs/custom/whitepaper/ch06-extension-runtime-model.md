# Chapter 6. SPI, Provider, Quarkus 런타임

> "Keycloak의 확장성은 플러그인을 꽂는 일이 아니라, 신뢰 경계 안에 코드를 들이는 일입니다."

Keycloak은 확장성이 강합니다. custom authenticator를 만들 수 있고, token mapper를 추가할 수 있고, 외부 user store를 붙일 수 있으며, event listener로 audit pipeline을 확장할 수 있습니다. 그래서 처음에는 “필요하면 plugin 하나 만들면 되겠네”라고 생각하기 쉽습니다.

하지만 custom provider는 sandbox에서 도는 작은 스크립트가 아닙니다. Keycloak server process 안에서 실행되고, login/token/admin 경로의 latency와 transaction에 영향을 줍니다. Quarkus 기반 배포에서는 provider와 theme 변경이 build/re-augmentation pipeline과도 연결됩니다.

---

## 6.1 설계 질문: "확장성을 얻는 대신 무엇을 신뢰하게 되는가?"

확장이 필요한 순간은 분명합니다.

1. 사내 MFA나 추가 본인확인 절차가 필요합니다.
2. token에 사내 표준 claim을 넣어야 합니다.
3. LDAP이 아닌 custom user store를 조회해야 합니다.
4. login/admin event를 SIEM이나 audit system으로 보내야 합니다.

이 모든 요구는 합리적입니다. 문제는 확장 코드가 Keycloak의 hot path 안으로 들어온다는 점입니다.

---

## 6.2 Provider lifecycle을 이해해야 하는 이유

| 개념 | lifecycle | 쉬운 설명 |
| --- | --- | --- |
| `Spi` | 서버 전체 | 어떤 종류의 확장 지점이 있는지 정의 |
| `ProviderFactory` | 서버 전체 | provider를 만들고 설정을 보관하는 factory |
| `Provider` | request/session 단위 | 실제 요청 안에서 동작하는 실행 객체 |
| `KeycloakSessionFactory` | 서버 전체 | provider factory registry와 default provider 관리 |
| `KeycloakSession` | request/transaction 단위 | context, transaction, provider lookup, datastore 접근의 중심 |

`ProviderFactory`는 singleton에 가깝고, `Provider`는 request/session 단위입니다. 따라서 factory에 request-specific 상태를 넣으면 thread-safety 문제가 생길 수 있습니다. 반대로 provider가 외부 API를 blocking으로 호출하면 로그인 path 전체가 느려집니다.

---

## 6.3 `KeycloakSession`은 작은 세계다

`KeycloakSession`은 단순한 service locator가 아닙니다. 한 요청이 처리되는 동안 realm context, client context, transaction manager, provider instance, cache invalidation, datastore facade를 묶는 작은 세계입니다.

| Session 요소 | 의미 |
| --- | --- |
| `getContext().getRealm()` | 현재 요청이 어떤 realm에 속하는지 결정 |
| `getProvider(clazz, id)` | session-scoped provider instance 조회 |
| `getTransactionManager()` | provider 작업의 commit/rollback 경계 |
| `setAttribute/getAttribute` | 요청 중 provider 간 임시 상태 공유 |
| `realms/users/clients/sessions` | model 접근 facade |
| `invalidate` | cache/provider invalidation hook |

custom provider가 session context를 잘못 이해하면 잘못된 realm에서 동작하거나, transaction boundary 밖에서 side effect를 내거나, cache consistency 문제를 만들 수 있습니다.

---

## 6.4 Quarkus가 바꾼 운영 모델

예전식 application server 모델에서는 runtime에 많은 것을 동적으로 바꾸는 느낌이 강했습니다. Quarkus 기반 Keycloak은 더 “appliance”에 가깝습니다. build-time augmentation 단계에서 provider discovery, Hibernate, Liquibase, cache config, static resource 처리가 정리되고, runtime에는 예측 가능한 서버가 빠르게 뜹니다.

| 구분 | 동적 서버 모델 | Quarkus optimized image 모델 |
| --- | --- | --- |
| 장점 | runtime 유연성 | 빠른 startup, 예측 가능한 배포, immutable 운영 |
| provider 변경 | 서버에 넣고 재시작하는 느낌 | image build 또는 re-augmentation과 연결 |
| theme/resource | runtime 변경 여지가 큼 | build/resource cache와 배포 pipeline 고려 |
| 운영 방식 | 서버를 직접 조정 | CI/CD에서 조립된 artifact를 배포 |

이 변화는 나쁜 것이 아닙니다. 운영 안정성을 얻기 위해 runtime 동적성을 일부 포기한 것입니다. 대신 provider, theme, feature flag, build-time option은 배포 pipeline의 일부로 다뤄야 합니다.

---

## 6.5 Custom provider의 위험 지도

| 위험 | 설명 | 운영 대응 |
| --- | --- | --- |
| classpath 충돌 | provider dependency가 서버 dependency와 충돌 | dependency scan, shaded jar 전략 검토 |
| secret 접근 | provider는 server process 권한으로 실행 | code review와 secret access 최소화 |
| latency 전파 | 외부 API 호출이 login/token path를 지연 | timeout, circuit breaker, canary test |
| transaction side effect | provider 실패가 요청 transaction에 영향 | 실패 정책과 rollback boundary 정의 |
| rollback 어려움 | provider 변경이 image와 DB/config에 결합 | 이전 image, config cleanup, smoke test 준비 |

Provider는 확장성의 문이지만 동시에 공격 표면과 장애 표면입니다. “작은 custom code”라고 가볍게 취급하지 않는 것이 중요합니다.

---

## 6.6 코드로 확인하는 증거

| 주장 | 확인할 파일 |
| --- | --- |
| provider factory lifecycle은 public SPI에 정의되어 있다 | `server-spi/src/main/java/org/keycloak/provider/ProviderFactory.java` |
| provider contract는 close 가능한 실행 객체로 정의된다 | `server-spi/src/main/java/org/keycloak/provider/Provider.java` |
| session은 provider lookup과 model facade를 제공한다 | `server-spi/src/main/java/org/keycloak/models/KeycloakSession.java` |
| default session은 provider instance를 session map에 cache한다 | `services/src/main/java/org/keycloak/services/DefaultKeycloakSession.java` |
| Quarkus build step은 provider loading과 build-time augmentation을 담당한다 | `quarkus/deployment/src/main/java/org/keycloak/quarkus/deployment/KeycloakProcessor.java` |
| runtime entrypoint와 application bootstrap은 Quarkus runtime에 있다 | `quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/KeycloakMain.java`, `quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/integration/jaxrs/QuarkusKeycloakApplication.java` |

---

## 6.7 운영자의 체크포인트

| 질문 | 기준 |
| --- | --- |
| provider를 server process 안에 넣어도 되는 신뢰 수준인가? | 보안 리뷰와 dependency 검증 필요 |
| provider 변경은 어떤 image/version으로 추적되는가? | rollback 가능한 artifact가 필요 |
| event listener 실패가 요청 실패로 이어져야 하는가? | audit 완전성과 availability tradeoff |
| user storage provider timeout은 얼마인가? | login latency와 외부 장애 격리 기준 |
| provider 포함 image로 smoke test를 수행하는가? | build-time/runtime mismatch 방지 |

---

## 6.8 핵심 인사이트

1. **SPI는 확장 지점이자 신뢰 경계입니다.** provider는 Keycloak process 내부에서 실행됩니다.
2. **`KeycloakSession`은 요청의 세계입니다.** realm, transaction, provider, datastore 접근이 session 안에 묶입니다.
3. **Quarkus는 운영 모델을 바꿉니다.** provider와 theme 변경은 runtime 수작업이 아니라 build와 delivery pipeline의 일부입니다.

---

| 방향 | 문서 |
| --- | --- |
| **이전 챕터** | [Ch.5 Federation과 Identity Brokering](./ch05-federation-and-brokering.md) |
| **다음 챕터** | [Ch.7 Release, Operator, 운영 안정성](./ch07-release-and-operations.md) |
| **백서 홈** | [WHITEPAPER.md](../WHITEPAPER.md) |
