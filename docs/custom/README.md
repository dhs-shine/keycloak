# Keycloak custom 문서 색인

## 1. 개요

이 문서는 Keycloak repository를 처음 접하는 사람이 **제품 목적, 서버 런타임, 정책 모델, 통합 지점, 개발/운영 계약**을 빠르게 찾기 위한 색인입니다.

`litellm-governance` 문서 체계와 동일하게, 문서는 두 층으로 나눕니다.

| 계층 | 문서 | 역할 |
| --- | --- | --- |
| 백서 | [Keycloak Identity Governance 시스템 백서](WHITEPAPER.md) | 신규 독자를 위한 배경 설명, 설계 철학, tradeoff, 실패 모드, 로드맵 |
| 참조 문서 | `00-foundation` ~ `90-decisions` | 구현자와 운영자가 바로 확인할 수 있는 계약, code path, 명령, 체크리스트 |

---

## 2. 핵심 방향

현재 문서 세트가 전제로 삼는 핵심 방향은 다음과 같습니다.

| 주제 | 결론 |
| --- | --- |
| **제품 해석** | Keycloak은 로그인 화면이 아니라 Identity Control Plane입니다. |
| **서버 기준** | 현재 서버 배포물의 중심은 `quarkus/`이며, build-time augmentation과 runtime 실행 경계를 분리합니다. |
| **요청 중심 객체** | REST resource, authentication flow, token 발급, storage 접근은 `KeycloakSession`을 중심으로 움직입니다. |
| **정책 모델** | `Realm`, `Client`, `User`, `Role`, `Group`, `Client Scope`, `Protocol Mapper`를 신뢰 계약과 권한 claim 모델로 해석합니다. |
| **상태 저장** | DB는 정책과 장기 상태의 장부이고, Infinispan은 cache/session의 빠른 상태 저장소입니다. |
| **확장 모델** | SPI/provider는 강력한 확장 지점이지만 server process 내부 신뢰 경계로 들어오는 코드입니다. |
| **운영 모델** | Operator는 Kubernetes desired state를 관리하지만 DB, cache, IdP, DNS/TLS, image build state까지 모두 소유하지 않습니다. |

---

## 3. 문서 구성 상태

현재 `docs/custom`의 구성은 다음과 같습니다.

| 영역 | 상태 |
| --- | --- |
| Whitepaper | `litellm-governance` 스타일의 3 Volume / 8 Chapter 구조로 재작성했습니다. |
| Foundation | 제품 목적, 범위, non-goals, 기준 아키텍처, source layout 계약을 정리합니다. |
| Architecture | bootstrap, request lifecycle, OIDC/token/admin/storage/event 흐름을 계약 문서로 정리합니다. |
| Policy | realm/client/user/role/scope/token/session 정책 의미론과 validation rule을 정리합니다. |
| Integration | JS UI, theme, Operator, test framework, SPI extension points를 표준 통합 표면으로 정리합니다. |
| Implementation | 빌드, 테스트, profile, 변경 유형별 workflow, release gate를 runbook 형태로 정리합니다. |
| Operations | DB/cache/proxy/Kubernetes/observability/security/failure mode 운영 기준을 정리합니다. |
| Decisions | 확정된 분석 기준과 아직 닫지 않은 운영 결정을 분리합니다. |

---

## 4. 디렉터리 구조

문서는 책임 영역별로 분리합니다.

| 영역 | 책임 | 문서 |
| --- | --- | --- |
| [WHITEPAPER.md](WHITEPAPER.md) | 설계 철학과 전체 narrative | [백서 색인](WHITEPAPER.md) |
| [whitepaper](whitepaper/ch01-design-philosophy.md) | 3 Volume / 8 Chapter 백서 본문 | [Ch.1](whitepaper/ch01-design-philosophy.md), [Ch.8](whitepaper/ch08-security-audit-and-roadmap.md) |
| [00-foundation](00-foundation/) | 요구사항, 범위, 기준 아키텍처, non-goals | [프로젝트 개요와 기준 아키텍처](00-foundation/01-project-overview-and-reference-architecture.md) |
| [10-architecture](10-architecture/) | 서버 bootstrap, request lifecycle, storage/event 흐름 | [서버 런타임과 요청 생명주기](10-architecture/10-server-runtime-and-request-lifecycle.md) |
| [20-policy](20-policy/) | realm/client/user/role/scope/token/session 정책 모델 | [Realm, Client, User 정책 모델](20-policy/20-realm-client-user-policy-model.md) |
| [30-integration](30-integration/) | UI, theme, Operator, test framework, extension points | [UI, Operator, 테스트와 확장 지점](30-integration/30-ui-operator-tests-and-extension-points.md) |
| [40-implementation](40-implementation/) | 로컬 개발, 빌드, 테스트, 변경 유형별 workflow | [개발, 빌드, 테스트 실행 계약](40-implementation/40-development-build-test-guide.md) |
| [50-operations](50-operations/) | 운영, 보안, 관측성, backup, failure mode | [운영, 보안, 관측성 계약](50-operations/50-operations-security-observability.md) |
| [90-decisions](90-decisions/) | 열린 결정과 후속 문서 후보 | [열린 결정 기록](90-decisions/90-open-decision-register.md) |

---

## 5. 빠른 탐색 경로

역할별 권장 읽기 순서는 다음과 같습니다.

| 독자 | 추천 읽기 순서 |
| --- | --- |
| **신규 리뷰어** | [WHITEPAPER](WHITEPAPER.md) -> [README](README.md) -> [Foundation](00-foundation/01-project-overview-and-reference-architecture.md) -> [Architecture](10-architecture/10-server-runtime-and-request-lifecycle.md) |
| **IAM 설계자** | [WHITEPAPER Ch.1~5](WHITEPAPER.md) -> [Policy](20-policy/20-realm-client-user-policy-model.md) -> [Operations](50-operations/50-operations-security-observability.md) -> [Decisions](90-decisions/90-open-decision-register.md) |
| **서버 개발자** | [Architecture](10-architecture/10-server-runtime-and-request-lifecycle.md) -> [Implementation](40-implementation/40-development-build-test-guide.md) -> [Integration](30-integration/30-ui-operator-tests-and-extension-points.md) |
| **UI 개발자** | [Integration](30-integration/30-ui-operator-tests-and-extension-points.md) -> [Implementation](40-implementation/40-development-build-test-guide.md) -> [Policy](20-policy/20-realm-client-user-policy-model.md) |
| **Operator 개발자** | [WHITEPAPER Ch.7](WHITEPAPER.md) -> [Integration](30-integration/30-ui-operator-tests-and-extension-points.md) -> [Operations](50-operations/50-operations-security-observability.md) |
| **운영자/SRE** | [WHITEPAPER Ch.7~8](WHITEPAPER.md) -> [Operations](50-operations/50-operations-security-observability.md) -> [Implementation](40-implementation/40-development-build-test-guide.md) -> [Decisions](90-decisions/90-open-decision-register.md) |

---

## 6. 백서와 참조 문서의 관계

| 백서 챕터 | 참조 문서 |
| --- | --- |
| Ch.1 설계 철학 | [00 Foundation](00-foundation/01-project-overview-and-reference-architecture.md) |
| Ch.2 신뢰 경계 | [00 Foundation](00-foundation/01-project-overview-and-reference-architecture.md), [50 Operations](50-operations/50-operations-security-observability.md) |
| Ch.3 정책 모델 | [20 Policy](20-policy/20-realm-client-user-policy-model.md) |
| Ch.4 인증/Token/Session | [10 Architecture](10-architecture/10-server-runtime-and-request-lifecycle.md), [20 Policy](20-policy/20-realm-client-user-policy-model.md) |
| Ch.5 Federation/Brokering | [20 Policy](20-policy/20-realm-client-user-policy-model.md), [30 Integration](30-integration/30-ui-operator-tests-and-extension-points.md) |
| Ch.6 SPI/Quarkus | [10 Architecture](10-architecture/10-server-runtime-and-request-lifecycle.md), [40 Implementation](40-implementation/40-development-build-test-guide.md) |
| Ch.7 Release/Operator/Ops | [30 Integration](30-integration/30-ui-operator-tests-and-extension-points.md), [40 Implementation](40-implementation/40-development-build-test-guide.md), [50 Operations](50-operations/50-operations-security-observability.md) |
| Ch.8 Security/Roadmap | [50 Operations](50-operations/50-operations-security-observability.md), [90 Decisions](90-decisions/90-open-decision-register.md) |

---

## 7. 기술 참조 계약

구현과 운영에서는 아래 계약을 기준으로 판단합니다.

| 주제 | 보존해야 할 계약 |
| --- | --- |
| Runtime entrypoint | `quarkus/runtime/src/main/java/org/keycloak/quarkus/runtime/KeycloakMain.java`가 CLI와 Quarkus lifecycle을 연결합니다. |
| Build-time augmentation | `quarkus/deployment/src/main/java/org/keycloak/quarkus/deployment/KeycloakProcessor.java`가 provider discovery, persistence, REST, Infinispan 구성을 build-time에 연결합니다. |
| Request context | `server-spi/src/main/java/org/keycloak/models/KeycloakSession.java`가 realm/client/session/provider context의 중심입니다. |
| Public realm root | `services/src/main/java/org/keycloak/services/resources/RealmsResource.java`가 realm을 resolve하고 protocol/account/broker/login-actions로 위임합니다. |
| OIDC flow | `OIDCLoginProtocolService`, `AuthorizationEndpoint`, `TokenEndpoint`, `TokenManager`가 authorization/token lifecycle의 중심입니다. |
| Admin API | `services/src/main/java/org/keycloak/services/resources/admin/AdminRoot.java`가 bearer token을 인증하고 admin resource로 위임합니다. |
| Storage facade | `model/storage-private/src/main/java/org/keycloak/storage/datastore/DefaultDatastoreProvider.java`가 cache, storage manager, JPA, federation provider를 조합합니다. |
| JS workspace | `js/package.json`, `js/pnpm-workspace.yaml`, `js/pom.xml`이 pnpm workspace와 Maven frontend build를 연결합니다. |
| Operator | `operator/src/main/java/org/keycloak/operator/controllers/KeycloakController.java`가 Keycloak CR reconciliation의 중심입니다. |
| Test framework | `test-framework/core/src/main/java/org/keycloak/testframework/KeycloakIntegrationTestExtension.java`가 신규 JUnit 5 test lifecycle의 중심입니다. |

현재 작업 범위의 기준은 다음과 같습니다.

| 영역 | 기준 |
| --- | --- |
| Source path | `/Users/dhsshin/Documents/LLMOps/keycloak` 현재 작업트리 |
| Reference style | `/Users/dhsshin/Documents/LLMOps/litellm-governance/docs` 문서 구조, 톤앤매너, 표준 구성요소 |
| Runtime code | 수정하지 않습니다. 문서만 정리합니다. |
| Validation | Markdown placeholder, stale link, code path sanity를 우선 확인합니다. |

---

## 8. 문서 작성 규칙

| 규칙 | 설명 |
| --- | --- |
| 한국어 우선 | 설명 문장은 한국어를 기본으로 합니다. |
| 기술 식별자 유지 | class, method, config key, command, file path, protocol 이름은 원문을 유지합니다. |
| 백서는 narrative | 백서는 배경과 문제 상황에서 시작합니다. |
| 참조 문서는 contract | 참조 문서는 `개요 -> 계약 -> 구조/흐름 -> 검증/참조 -> 작업 범위` 흐름을 기준으로 하며, decision register는 table-first 형식으로 변형합니다. |
| 표와 Mermaid 사용 | 책임 경계, sequence, failure policy, 운영 기준은 표와 Mermaid로 정리합니다. |
| Non-goals 명시 | 현재 문서가 하지 않는 일을 분리합니다. |
| Open decisions 분리 | 실제 운영 결정을 문서 안에서 암묵적으로 확정하지 않고 [90 Decisions](90-decisions/90-open-decision-register.md)로 보냅니다. |

---

## 9. 작업 범위 기록

이 단계에서는 `docs/custom` 하위 분석 문서와 백서 구조만 재정비합니다. Java, TypeScript, Maven, Operator, 테스트 runtime 코드는 수정하지 않습니다.
