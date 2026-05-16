# 열린 결정 기록

## 1. 개요

이 문서는 `docs/custom` 분석 문서에서 이미 확정한 판단과 아직 닫지 않은 결정을 분리하는 decision register입니다. 목적은 문서 안에서 운영 결정을 암묵적으로 확정하지 않고, 실제 배포/개발 작업 전에 선택해야 할 항목을 추적하는 것입니다.

| 항목 | 값 |
| --- | --- |
| 기준 repository | `/Users/dhsshin/Documents/LLMOps/keycloak` |
| 문서 기준 | `docs/custom` |
| 정리일 | 2026-05-17 |
| 문서 스타일 | `litellm-governance` reference-doc style |
| 작업 범위 | 분석과 문서화 |
| Runtime 변경 | 없음 |

---

## 2. Register 계약

| 구분 | 의미 | 처리 방식 |
| --- | --- | --- |
| Accepted baseline | 코드/문서 분석으로 현재 확정 가능한 사실 | 참조 문서와 백서의 전제로 사용합니다. |
| Open decision | 환경, 배포 전략, 운영 정책이 필요해 아직 확정할 수 없는 항목 | 실제 구현/배포 작업 전에 owner와 due point를 정합니다. |
| Follow-up document | 결정이 닫힌 뒤 별도 문서화가 필요한 deep dive | 필요해질 때 `00`~`90` 또는 `whitepaper` 하위 문서로 분리합니다. |
| Superseded decision | 이후 결정으로 대체된 항목 | register에 남기되 근거 문서를 연결합니다. |

---

## 3. Accepted Baseline

| ID | 판단 | 근거 | 영향 문서 |
| --- | --- | --- | --- |
| DEC-001 | 현재 서버 배포물의 기준은 `quarkus/` 모듈입니다. | `quarkus/README.md`, `quarkus/pom.xml` | Architecture, Implementation |
| DEC-002 | 요청 처리의 중심 context는 `KeycloakSession`입니다. | `server-spi/src/main/java/org/keycloak/models/KeycloakSession.java`, `services/src/main/java/org/keycloak/services/DefaultKeycloakSession.java` | Architecture |
| DEC-003 | SPI 확장은 `ProviderFactory` server lifecycle과 `Provider` session lifecycle로 해석합니다. | `server-spi/src/main/java/org/keycloak/provider/ProviderFactory.java` | Architecture, Integration |
| DEC-004 | Public realm endpoint는 `RealmsResource`가 realm을 resolve하고 protocol/account/broker/login-actions로 위임합니다. | `services/src/main/java/org/keycloak/services/resources/RealmsResource.java` | Architecture |
| DEC-005 | OIDC authorization/token 흐름의 중심은 `OIDCLoginProtocolService`, `AuthorizationEndpoint`, `TokenEndpoint`, `TokenManager`입니다. | `services/src/main/java/org/keycloak/protocol/oidc/` | Architecture, Policy |
| DEC-006 | Admin API는 `AdminRoot`가 bearer token을 인증한 뒤 admin resource로 위임합니다. | `services/src/main/java/org/keycloak/services/resources/admin/AdminRoot.java` | Architecture, Policy |
| DEC-007 | 핵심 storage facade는 `DefaultDatastoreProvider`이며 cache/storage manager/JPA/federation provider를 조합합니다. | `model/storage-private/src/main/java/org/keycloak/storage/datastore/DefaultDatastoreProvider.java` | Architecture, Operations |
| DEC-008 | 신규 테스트 작성은 가능한 `test-framework/`를 우선합니다. | `test-framework/docs/README.md`, `testsuite/DEPRECATED.md` | Integration, Implementation |
| DEC-009 | JS UI는 `js/` pnpm workspace와 Maven frontend build가 함께 관리합니다. | `js/README.md`, `js/pom.xml` | Integration, Implementation |
| DEC-010 | Operator는 기본 서버 runtime과 별도 profile/module로 관리되는 Quarkus Operator SDK 기반 코드입니다. | `operator/README.md`, `operator/pom.xml` | Integration, Operations |
| DEC-011 | 백서 계층은 `WHITEPAPER.md` 허브와 `whitepaper/ch01`~`ch08` 장별 문서로 둡니다. | `docs/custom/WHITEPAPER.md`, `docs/custom/whitepaper/` | Whitepaper |
| DEC-012 | 백서 문체는 `litellm-governance`의 3 Volume 구조와 문제 상황 중심 narrative를 따릅니다. | `/Users/dhsshin/Documents/LLMOps/litellm-governance/docs/WHITEPAPER.md` | Whitepaper, README |
| DEC-013 | 참조 문서는 contract-first 흐름을 기준으로 하며, decision register는 table-first 형식으로 변형합니다. | `docs/custom/README.md` | 모든 reference docs |

---

## 4. Open Decision Register

| ID | 우선순위 | 결정 영역 | 선택지 | 닫아야 하는 시점 |
| --- | --- | --- | --- | --- |
| OPEN-001 | 높음 | `docs/custom`를 upstream 분석 reference로 둘지 LLMOps 배포 guide로 확장할지 결정 | upstream 분석 유지, LLMOps 배포 guide 추가, 둘 다 분리 | 문서 공유/리뷰 전 |
| OPEN-002 | 높음 | `docs/custom` 문서를 Keycloak 공식 docs 구조와 연결할지 결정 | 독립 유지, 공식 `docs/`에서 link, 별도 README에서만 link | 문서 공개 전 |
| OPEN-003 | 높음 | 운영 기준 DB vendor 결정 | PostgreSQL, MariaDB/MySQL, MSSQL, Oracle, vendor별 matrix | production architecture 결정 전 |
| OPEN-004 | 높음 | cache/session topology 결정 | local embedded Infinispan, remote Infinispan, hybrid | multi-pod production 설계 전 |
| OPEN-005 | 높음 | Kubernetes 배포 방식 결정 | Operator, Helm/manifests, GitOps hybrid | 배포 자동화 구현 전 |
| OPEN-006 | 높음 | hostname/TLS/proxy 기준 결정 | single public hostname, admin hostname 분리, private ingress 분리 | 외부 노출 설계 전 |
| OPEN-007 | 높음 | realm key와 secret rotation 절차 결정 | 수동 runbook, scheduled rotation, external secret manager 연동 | production go-live 전 |
| OPEN-008 | 중간 | Admin UI/Account UI 개발 workflow 결정 | Maven-first, pnpm-first, mixed | UI 변경 작업 전 |
| OPEN-009 | 중간 | 신규 테스트 위치 결정 | `test-framework`, `tests`, 기존 `testsuite` 유지보수 | 기능 구현 전 |
| OPEN-010 | 중간 | custom SPI/provider 배치 결정 | in-tree module, external provider repository, examples only | custom extension 개발 전 |
| OPEN-011 | 중간 | event/audit sink 결정 | DB event store, log listener, custom event listener, external SIEM sink | audit 요구사항 확정 전 |
| OPEN-012 | 중간 | token claim과 mapper 표준 결정 | minimal default scope, optional scopes, service-specific client scopes | application onboarding 전 |
| OPEN-013 | 중간 | Operator가 관리할 CR 범위 결정 | Keycloak only, RealmImport 포함, OIDC/SAML client CR 포함 | GitOps 설계 전 |
| OPEN-014 | 낮음 | legacy `testsuite`와 신규 `test-framework` migration map 작성 여부 결정 | 작성, 필요 시 작성, 작성 안 함 | 테스트 modernization 작업 전 |
| OPEN-015 | 낮음 | 백서 8장 아래 deep dive 하위 문서 추가 여부 결정 | 현재 구조 유지, OIDC/Admin/storage별 하위 문서 추가 | 다음 상세 분석 요청 시 |

---

## 5. Decision Close Criteria

| 결정 유형 | 닫기 위한 증거 | 기록 위치 |
| --- | --- | --- |
| 배포 방식 | architecture diagram, manifest/Operator strategy, rollback path | Operations, Implementation, follow-up runbook |
| DB/cache 선택 | vendor/version, HA topology, backup/restore drill, performance target | Operations |
| 보안 정책 | threat model, secret/key rotation, token/session policy, audit retention | Policy, Operations |
| 테스트 전략 | target module, framework location, server/db/browser matrix, CI gate | Implementation, Integration |
| UI 개발 방식 | pnpm/Maven workflow, dev server 기준, release packaging path | Integration, Implementation |
| 문서 공개 방식 | link target, owner, review path, upstream/docs 분리 기준 | README, this register |

---

## 6. Follow-up Document 후보

| 후보 문서 | 목적 | 선행 결정 |
| --- | --- | --- |
| `10.1-oidc-authorization-code-flow-detail.md` | authorization endpoint, authentication session, code 발급을 코드 단위로 상세 분석 | OIDC flow 변경 필요 |
| `10.2-token-endpoint-grants-and-token-manager-detail.md` | grant provider와 token manager/token mapper 흐름 상세 분석 | token/claim 변경 필요 |
| `10.3-admin-api-permission-and-event-detail.md` | Admin API 권한 검사와 admin event 흐름 상세 분석 | Admin API/FGAP 변경 필요 |
| `10.4-storage-cache-jpa-federation-detail.md` | `DefaultDatastoreProvider`, cache, storage manager, JPA/federation 경계 상세 분석 | storage/model 변경 필요 |
| `20.1-authentication-flow-design-guide.md` | realm authentication flow 설계와 custom authenticator guide | 인증 정책 설계 필요 |
| `20.2-token-scope-mapper-design-guide.md` | client scope/protocol mapper/token claim 설계 기준 | application onboarding 필요 |
| `30.1-admin-ui-route-and-admin-client-map.md` | Admin UI route와 Admin REST client map | UI 변경 작업 필요 |
| `30.2-operator-crd-controller-detail.md` | Operator CRD/controller/dependent resource lifecycle 상세 분석 | Operator 변경 필요 |
| `40.1-local-dev-environment-runbook.md` | 로컬 개발 환경 상세 runbook | onboarding 문서 필요 |
| `50.1-production-kubernetes-runbook.md` | production Kubernetes 운영 runbook | 배포 방식 결정 필요 |
| `whitepaper/ch04.1-authorization-code-code-path.md` | Authorization Code + PKCE를 method 단위로 상세 추적 | OIDC conformance 검토 필요 |
| `whitepaper/ch04.2-infinispan-cache-topology.md` | cache/session별 Infinispan topology, owner, sticky session, remote cache 상세화 | cache topology 결정 필요 |
| `whitepaper/ch07.1-operator-reconciliation-runbook.md` | Operator reconcile, update strategy, status condition 상세 runbook | Kubernetes 운영 방식 결정 필요 |
| `whitepaper/ch08.1-security-hardening-controls.md` | failure mode별 hardening control, smoke test, 검증 명령 정리 | production hardening baseline 필요 |

---

## 7. Decision 기록 형식

향후 실제 결정을 닫을 때는 아래 형식을 사용합니다.

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

---

## 8. 작업 범위 기록

이 문서는 분석과 문서화만 수행합니다. 실제 운영 결정, 배포 방식, DB/cache 선택, 테스트 migration, 보안 정책은 별도 작업에서 확정합니다.
