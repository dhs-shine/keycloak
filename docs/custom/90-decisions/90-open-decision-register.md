# 열린 결정 기록

> 네비게이션: [문서 색인](../README.md) | 이전: [운영, 보안, 관측성](../50-operations/50-operations-security-observability.md)
> 관련 문서: [Keycloak 시스템 백서](../WHITEPAPER.md), [프로젝트 개요와 기준 아키텍처](../00-foundation/01-project-overview-and-reference-architecture.md), [서버 런타임과 요청 생명주기](../10-architecture/10-server-runtime-and-request-lifecycle.md), [개발/빌드/테스트 가이드](../40-implementation/40-development-build-test-guide.md)

작성일: 2026-05-16

최신 소스 재검증: 2026-05-16, `/Users/dhsshin/Documents/LLMOps/keycloak` 현재 작업트리 기준

## 목적

이 문서는 현재 분석에서 확정한 사항과 추가 확인이 필요한 열린 질문을 분리해 기록한다. 코드나 운영 정책을 실제로 바꾸기 전 결정해야 할 항목을 추적하기 위한 문서다.

## 확정한 분석 기준

| ID | 결정/판단 | 근거 |
| --- | --- | --- |
| DEC-001 | 현재 서버 배포물의 기준은 `quarkus/` 모듈이다. | `quarkus/README.md`, `quarkus/pom.xml` |
| DEC-002 | 요청 처리의 중심 객체는 `KeycloakSession`이다. | `server-spi/src/main/java/org/keycloak/models/KeycloakSession.java`, `DefaultKeycloakSession.java` |
| DEC-003 | SPI/provider 확장은 `ProviderFactory` 서버 lifecycle과 `Provider` session lifecycle을 기준으로 이해한다. | `server-spi/src/main/java/org/keycloak/provider/ProviderFactory.java` |
| DEC-004 | Public realm endpoint는 `RealmsResource`가 realm을 resolve하고 protocol/account/broker/login-actions로 위임한다. | `services/src/main/java/org/keycloak/services/resources/RealmsResource.java` |
| DEC-005 | OIDC authorization/token 흐름의 중심은 `OIDCLoginProtocolService`, `AuthorizationEndpoint`, `TokenEndpoint`, `TokenManager`다. | `services/src/main/java/org/keycloak/protocol/oidc/` |
| DEC-006 | Admin API는 `AdminRoot`가 bearer token을 인증한 뒤 admin resource로 위임한다. | `services/src/main/java/org/keycloak/services/resources/admin/AdminRoot.java` |
| DEC-007 | 현재 핵심 storage facade는 `DefaultDatastoreProvider`이며 cache/storage manager/JPA/federation provider를 조합한다. | `model/storage-private/src/main/java/org/keycloak/storage/datastore/DefaultDatastoreProvider.java` |
| DEC-008 | 신규 테스트 작성은 `test-framework/`를 우선 기준으로 본다. | `test-framework/docs/README.md`, `testsuite/DEPRECATED.md` |
| DEC-009 | JS UI는 `js/` pnpm workspace와 Maven frontend plugin이 함께 관리한다. | `js/README.md`, `js/pom.xml` |
| DEC-010 | Operator는 기본 서버 runtime과 별도 profile/모듈로 관리되는 Quarkus Operator SDK 기반 코드다. | `operator/README.md`, `operator/pom.xml` |
| DEC-011 | 백서 계층은 `WHITEPAPER.md` 허브와 `whitepaper/ch01`~`ch10` 장별 상세 문서로 둔다. | `docs/custom/WHITEPAPER.md`, `docs/custom/whitepaper/` |
| DEC-012 | 운영 실패 모드는 탐지 신호, 즉시 조치, 검증 기준까지 포함해 기술한다. | `docs/custom/whitepaper/ch10-operations-security-failure-modes.md` |

## 열린 결정 목록

| ID | 우선순위 | 질문 | 선택지 | 닫아야 하는 시점 |
| --- | --- | --- | --- | --- |
| OPEN-001 | 높음 | 이 repository 분석 문서를 upstream general reference로 둘 것인가, 특정 LLMOps 환경용 운영 guide로 확장할 것인가 | upstream 분석 유지, LLMOps 배포 guide 추가, 둘 다 분리 | 실제 배포 작업 시작 전 |
| OPEN-002 | 높음 | `docs/custom` 문서를 Keycloak 공식 docs 구조와 연결할 것인가 | 독립 분석 문서 유지, `docs/` 공식 guide에 link 추가, 별도 README만 link | 문서 공유/리뷰 전 |
| OPEN-003 | 높음 | 운영 기준 DB를 무엇으로 둘 것인가 | PostgreSQL, MariaDB/MySQL, MSSQL, Oracle, vendor별 matrix | production architecture 결정 전 |
| OPEN-004 | 높음 | cache/session 전략을 local Infinispan으로 둘 것인가 remote Infinispan으로 둘 것인가 | local embedded, remote Infinispan, hybrid | multi-pod production 설계 전 |
| OPEN-005 | 높음 | Kubernetes 배포를 Operator 중심으로 할 것인가 직접 manifests/Helm 중심으로 할 것인가 | Operator, Helm/manifests, GitOps hybrid | 배포 자동화 구현 전 |
| OPEN-006 | 중간 | Admin UI/Account UI 개발 workflow를 Maven 중심으로 둘 것인가 pnpm dev server 중심으로 둘 것인가 | Maven-first, pnpm-first, mixed | UI 변경 작업 전 |
| OPEN-007 | 중간 | 신규 테스트를 어디에 둘 것인가 | `test-framework`, `tests`, 기존 `testsuite` 유지보수 | 기능 구현 전 |
| OPEN-008 | 중간 | custom SPI/provider 개발을 이 repository 안에 둘 것인가 외부 provider JAR로 둘 것인가 | in-tree module, external provider repository, examples only | custom extension 개발 전 |
| OPEN-009 | 중간 | event/audit sink를 DB event store만 사용할 것인가 외부 SIEM/log pipeline과 연동할 것인가 | DB event store, log listener, custom event listener, external sink | audit 요구사항 확정 전 |
| OPEN-010 | 중간 | token claim과 mapper 표준을 서비스별로 어떻게 제한할 것인가 | minimal default scope, optional scopes, service-specific client scopes | application onboarding 전 |
| OPEN-011 | 낮음 | 구 `testsuite` 문서와 신규 `test-framework` 문서의 migration map을 별도로 작성할 것인가 | 작성, 필요 시 작성, 작성 안 함 | 테스트 modernization 작업 전 |
| OPEN-012 | 낮음 | 문서 세트를 더 세분화해 lifecycle 하위 문서를 만들 것인가 | 현재 상세 문서 세트 유지, OIDC/Admin/storage별 하위 문서 추가 | 다음 상세 분석 요청 시 |

## 추가 상세 문서 후보

| 후보 문서 | 목적 | 선행 조건 |
| --- | --- | --- |
| `10.1-oidc-authorization-code-flow-detail.md` | authorization endpoint, authentication session, code 발급을 코드 단위로 상세 분석 | OIDC flow 변경 또는 conformance 검토 필요 |
| `10.2-token-endpoint-grants-and-token-manager-detail.md` | grant provider와 token manager/token mapper 흐름 상세 분석 | token/claim 변경 필요 |
| `10.3-admin-api-permission-and-event-detail.md` | Admin API 권한 검사와 admin event 흐름 상세 분석 | Admin API/FGAP 변경 필요 |
| `10.4-storage-cache-jpa-federation-detail.md` | `DefaultDatastoreProvider`, cache, storage manager, JPA/federation 경계 상세 분석 | storage/model 변경 필요 |
| `20.1-authentication-flow-design-guide.md` | realm authentication flow 설계와 custom authenticator guide | 인증 정책 설계 필요 |
| `20.2-token-scope-mapper-design-guide.md` | client scope/protocol mapper/token claim 설계 기준 | application onboarding 필요 |
| `30.1-admin-ui-route-and-admin-client-map.md` | Admin UI route와 Admin REST client map | UI 변경 작업 필요 |
| `30.2-operator-crd-controller-detail.md` | Operator CRD/controller/dependent resource lifecycle 상세 분석 | Operator 변경 필요 |
| `40.1-local-dev-environment-runbook.md` | 로컬 개발 환경 상세 runbook | 실제 onboarding 문서 필요 |
| `50.1-production-kubernetes-runbook.md` | production Kubernetes 운영 runbook | 배포 환경 결정 필요 |
| `whitepaper/ch06.1-authorization-code-code-path.md` | Authorization Code + PKCE를 method 단위로 상세 추적 | OIDC conformance 또는 인증 flow 변경 필요 |
| `whitepaper/ch07.1-infinispan-cache-topology.md` | cache/session별 Infinispan topology, owner, sticky session, remote cache 상세화 | multi-pod 또는 remote Infinispan 설계 필요 |
| `whitepaper/ch09.1-operator-reconciliation-runbook.md` | Operator reconcile, update strategy, status condition 상세 runbook | Kubernetes 운영 방식 확정 필요 |
| `whitepaper/ch10.1-security-hardening-controls.md` | 실패 모드별 hardening control, smoke test, 검증 명령 정리 | production hardening baseline 필요 |

## Decision 기록 방식

향후 실제 결정을 내릴 때는 아래 형식을 사용한다.

```markdown
## DEC-XXX: 제목

상태: proposed | accepted | superseded | rejected

날짜: YYYY-MM-DD

문맥:

결정:

대안:

결과:

관련 문서:
```

## 작업 범위 기록

이 문서는 분석과 문서화만 수행한다. 실제 운영 결정, 배포 방식, DB/cache 선택, 테스트 migration은 별도 작업에서 확정한다.
