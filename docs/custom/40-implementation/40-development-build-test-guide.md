# 개발, 빌드, 테스트 가이드

> 네비게이션: [문서 색인](../README.md) | 이전: [UI, Operator, 테스트와 확장 지점](../30-integration/30-ui-operator-tests-and-extension-points.md) | 다음: [운영, 보안, 관측성](../50-operations/50-operations-security-observability.md)
> 관련 문서: [프로젝트 개요와 기준 아키텍처](../00-foundation/01-project-overview-and-reference-architecture.md), [서버 런타임과 요청 생명주기](../10-architecture/10-server-runtime-and-request-lifecycle.md)

작성일: 2026-05-16

최신 소스 재검증: 2026-05-16, `/Users/dhsshin/Documents/LLMOps/keycloak` 현재 작업트리 기준

## 목적

이 문서는 Keycloak 저장소에서 개발자가 실제로 코드를 수정하고 검증할 때 필요한 빌드 명령, Maven profile, 테스트 실행 방식, 변경 유형별 작업 흐름을 정리한다.

## 사전 요구사항

| 항목 | 기준 | 근거 |
| --- | --- | --- |
| JDK | JDK 17, JDK 21, JDK 25 지원. 더 최신 JDK는 지원되지 않음 | `docs/building.md` |
| Git | repository clone/build에 필요 | `docs/building.md` |
| Maven | 로컬 Maven 대신 루트 `./mvnw` 사용 권장 | `docs/building.md` |
| Maven version | root property 기준 `3.9.8` | `pom.xml` |
| Java release | root compiler release `17` | `pom.xml` |
| Node/Pnpm | JS Maven build가 Node `v24.9.0`, pnpm `10.14.0` 설치 | `js/pom.xml` |
| Docker/Podman | DB, container, Operator, 일부 test-framework/testsuite 실행에 필요 | `test-framework/docs/RUNNING_TESTS.md`, `operator/README.md` |

## 빌드 명령 요약

| 목적 | 명령 | 설명 |
| --- | --- | --- |
| 테스트 생략 전체 빌드 | `./mvnw clean install -DskipTests` | 가장 기본적인 빠른 전체 compile/package 검증 |
| 테스트 포함 전체 빌드 | `./mvnw clean install` | 오래 걸릴 수 있음 |
| distribution profile 포함 | `./mvnw clean install -DskipTests -Pdistribution` | distribution 부가 산출물 포함 |
| 서버만 빌드 | `./mvnw -pl quarkus/deployment,quarkus/dist -am -DskipTests clean install` | Quarkus server distribution 중심 빌드 |
| Quarkus 최초 개발 빌드 | `../mvnw -f ../pom.xml clean install -DskipTestsuite -DskipExamples -DskipTests` | `quarkus/`에서 개발 전 local Maven cache 준비 |
| Quarkus distribution 빌드 | `../mvnw clean install -DskipTests` | `quarkus/`에서 ZIP/TAR 생성 |
| dist만 빌드 | `../mvnw -f dist/pom.xml clean install` | 기존 build 결과를 활용해 dist packaging |
| Quarkus dev mode | `../mvnw -f server/pom.xml compile quarkus:dev -Dkc.config.built=true -Dquarkus.args="start-dev"` | `http://localhost:8080` 개발 서버 |
| Quarkus dev mode suspend debug | `../mvnw -f server/pom.xml -Dsuspend=true compile quarkus:dev -Dkc.config.built=true -Dquarkus.args="start-dev"` | build step 초기 debug |
| distribution debug | `kc.sh --debug start-dev` | 기본 localhost `8787` debug port |
| JS 생략 | `-Dskip.npm` | JS workspace processing skip |
| Operator 포함 | `./mvnw clean install -Poperator -DskipTests` | Operator는 기본 제외 가능성이 있어 profile 필요 |
| proto lock skip | `-DskipProtoLock=true` | proxy 환경 등에서 proto compatibility check 실패 회피 |

## Maven profile 해석

| Profile/조건 | 동작 | 주의점 |
| --- | --- | --- |
| `!skipTestsuite` | `testsuite` module 포함 | 테스트 실행을 skip해도 module aggregation 자체는 커질 수 있음 |
| `!skipAdapters` | `adapters` module 포함 | adapter 관련 build가 필요 없으면 skip flag 고려 |
| `!skipDocs` | `docs` module 포함 | 문서 빌드가 필요 없으면 skip flag 고려 |
| `-Pdistribution` | `distribution` module 포함 | Quarkus dist와 별개로 SAML adapters/Galleon/license/downloads 성격 포함 |
| `-Doperator` | `operator` module 포함 | IDE에서 Operator build 전 필요 |
| `-Doperator-prod` | Operator production 성격 build | container/image 설정과 함께 검토 |
| `-Dfips140-2` | FIPS crypto artifact 선택 | crypto provider와 운영 JDK/security provider 조건 확인 필요 |

## 개발 서버 실행

### Quarkus dev mode

```bash
cd quarkus
../mvnw -f server/pom.xml compile quarkus:dev -Dkc.config.built=true -Dquarkus.args="start-dev"
```

특징:

| 항목 | 내용 |
| --- | --- |
| 기본 URL | `http://localhost:8080` |
| debug | port `5005` attach 가능 |
| 자동 반영 | `quarkus/deployment`, `quarkus/runtime`, `quarkus/server` resource 변경 중심 |
| 한계 | `keycloak-services` 같은 non-Quarkus module Java 변경은 Hot Swap에 의존 |
| build-time option | `quarkus.args`만으로 반영되지 않으며 `-D` 또는 env와 재실행 필요 |

### Distribution 실행

```bash
bin/kc.sh start-dev
```

또는 Windows:

```bat
bin\kc.bat start-dev
```

debug 실행:

```bash
kc.sh --debug start-dev
```

보안 주의:

| 설정 | 주의 |
| --- | --- |
| `--debug` | 기본은 localhost `8787` |
| `--debug 0.0.0.0:8787` | 모든 network interface에 debug port가 노출될 수 있음 |
| `DEBUG_SUSPEND=y` | early bootstrap debug에 유용하지만 운영에서는 사용 금지 |

## 테스트 실행 경로

### 신규 test-framework

| 목적 | 명령/설정 |
| --- | --- |
| 기본 test-framework test | 해당 module에서 `mvn test` |
| embedded server | `KC_TEST_SERVER=embedded mvn test` |
| distribution server | `KC_TEST_SERVER=distribution mvn test` |
| remote server | `KC_TEST_SERVER=remote mvn test` |
| Postgres DB | `KC_TEST_DATABASE=postgres mvn test` |
| Chrome browser | `KC_TEST_BROWSER=chrome mvn test` |
| server reuse | `KC_TEST_SERVER_REUSE=true` |
| hot deploy | `KC_TEST_SERVER_HOT_DEPLOY=true` |

설정 우선순위:

| 순서 | Source |
| --- | --- |
| 1 | System properties |
| 2 | Environment variables |
| 3 | `.env.test` |
| 4 | classpath `keycloak-test.properties` |
| 5 | `KC_TEST_CONFIG`로 지정한 properties 파일 |

### Legacy testsuite

| 목적 | 명령 |
| --- | --- |
| integration-arquillian 전체 | `mvn -f testsuite/integration-arquillian/pom.xml clean install` |
| 단일 테스트 | `mvn -f testsuite/integration-arquillian/pom.xml clean install -Dtest=LoginTest` |
| Quarkus production mode | `mvn -f testsuite/integration-arquillian/pom.xml -Pauth-server-quarkus clean install` |
| Quarkus embedded | `mvn -f testsuite/integration-arquillian/pom.xml -Pauth-server-quarkus-embedded clean install -Dtest=LoginTest` |
| browser 지정 | `-Dbrowser=firefox` 또는 `-Dbrowser=chrome` |

주의:

| 항목 | 설명 |
| --- | --- |
| deprecated | `testsuite/DEPRECATED.md`에 따라 신규 테스트는 test-framework 우선 |
| production mode 테스트 | 코드 변경 후 distribution rebuild 필요 가능 |
| DB/browser 의존성 | Docker/Podman, Selenium/WebDriver, Chrome/Firefox 환경 영향 |

### Test utilities

| 목적 | 명령 |
| --- | --- |
| test Keycloak server | `mvn exec:java -Pkeycloak-server` from `testsuite/utils` |
| realm import 포함 | `mvn exec:java -Pkeycloak-server -Dimport=testrealm.json` |
| resources live edit | `mvn exec:java -Pkeycloak-server -Dresources` |
| test mail server | `./mvnw -f testsuite/utils/pom.xml exec:java -Pmail-server` |
| test LDAP server | `./mvnw -f testsuite/utils/pom.xml exec:java -Pldap` |
| test Kerberos server | `mvn exec:java -Pkerberos` |

## JS/UI 개발

| 작업 | 명령/위치 |
| --- | --- |
| JS workspace build | `cd js && pnpm build` |
| Maven에서 JS build 포함 | 루트 Maven build 기본 경로 |
| Maven에서 JS skip | `-Dskip.npm` |
| Admin UI dev server | `js/apps/admin-ui`, Vite port `5174` |
| Account UI dev server | `js/apps/account-ui`, Vite port `5173` |
| UI 개발용 Keycloak server | `js/apps/keycloak-server` |
| Admin dev 연동 | `pnpm start --admin-dev` |
| Account dev 연동 | `pnpm start --account-dev` |
| local dist 연동 | `pnpm start --local` |

개발 server 환경 변수 예:

```text
KC_ACCOUNT_VITE_URL=http://localhost:5173
KC_ADMIN_VITE_URL=http://localhost:5174
KC_FEATURES=login:v2,account:v3,admin-fine-grained-authz,transient-users,oid4vc-vci
```

## Operator 개발과 테스트

| 목적 | 명령 |
| --- | --- |
| Operator image build | `mvn clean package -Dquarkus.container-image.build=true` in `operator/` |
| Minikube quick start | `minikube start --addons ingress --cni cilium --cpus=max` |
| target manifest 적용 | `kubectl create namespace keycloak && kubectl apply -k target` |
| default namespace overlay | `kubectl apply -k overlays/default-namespace` |
| remote operator tests | `mvn clean verify -Dquarkus.container-image.build=true -Dquarkus.container-image.tag=test -Dquarkus.kubernetes.image-pull-policy=IfNotPresent -Dtest.operator.deployment=remote` |
| custom image tests | `./scripts/build-testing-docker-images.sh [SOURCE KEYCLOAK IMAGE TAG] [SOURCE KEYCLOAK IMAGE]` 후 `-Dtest.operator.custom.image=custom-keycloak:latest` |

Operator test mode:

| Mode | 설명 |
| --- | --- |
| `local_apiserver` | 기본값. jenvtest controlled API server와 local operator. 빠르지만 모든 테스트 불가 |
| `local` | local cluster에 resource 배포, operator는 cluster 밖 local 실행 |
| `remote` | operator image를 생성해 cluster 내부에 배포 |

## 변경 유형별 작업 흐름

### OIDC/token endpoint 변경

| 단계 | 작업 |
| --- | --- |
| 1 | `services/src/main/java/org/keycloak/protocol/oidc/`에서 endpoint/grant/mapper 영향 범위 확인 |
| 2 | `AuthorizationEndpoint`, `TokenEndpoint`, `TokenManager`, grant provider 중 실제 변경 지점 식별 |
| 3 | client policy, CORS, event, token mapper, session 영향 확인 |
| 4 | unit/integration test를 test-framework 또는 기존 testsuite 위치에 추가/수정 |
| 5 | 변경 module 중심 Maven test와 필요한 distribution test 실행 |

### Authentication flow 변경

| 단계 | 작업 |
| --- | --- |
| 1 | `services/src/main/java/org/keycloak/authentication/`에서 authenticator/required action 위치 확인 |
| 2 | `AuthenticationProcessor` lifecycle과 `LoginActionsService` action URL 영향을 확인 |
| 3 | event, brute force protector, user session attach, required action 상태를 검토 |
| 4 | browser/direct grant/required action 별 테스트 추가 |
| 5 | UI/theme 변경이 필요한 경우 `themes/` 또는 `js/` 영향 확인 |

### Storage/model 변경

| 단계 | 작업 |
| --- | --- |
| 1 | `server-spi` model interface 변경 여부를 먼저 판단 |
| 2 | `model/jpa` adapter/entity/provider 변경 필요 여부 확인 |
| 3 | `model/storage-private` storage manager/datastore/cache 경유 여부 확인 |
| 4 | DB schema 변경 시 `docs/updating-database-schema.md` 확인 |
| 5 | cache invalidation, migration, test DB matrix 검증 |

### Admin UI 변경

| 단계 | 작업 |
| --- | --- |
| 1 | `js/apps/admin-ui/src`의 기능 디렉토리와 route 확인 |
| 2 | `js/libs/keycloak-admin-client` resource method 필요 여부 확인 |
| 3 | i18n message와 PatternFly component usage 확인 |
| 4 | Vite dev server 또는 UI test 실행 |
| 5 | Maven JS build와 theme packaging 영향 확인 |

### Operator 변경

| 단계 | 작업 |
| --- | --- |
| 1 | CRD spec/status 변경인지 controller/dependent resource 변경인지 구분 |
| 2 | `operator/src/main/java/org/keycloak/operator/crds/`와 `controllers/` 영향 확인 |
| 3 | generated CRD/manifests update 필요 여부 확인 |
| 4 | `local_apiserver`, `local`, `remote` test mode 중 적절한 mode 선택 |
| 5 | status reconciliation과 idempotency를 검증 |

## IDE와 generated code 주의점

| 주의점 | 설명 |
| --- | --- |
| Maven build 선행 | 일부 코드는 Maven plugin으로 생성되므로 IDE build만 먼저 실행하면 compile error 가능 |
| IntelliJ rebuild 주의 | 전체 rebuild가 generated classes를 삭제할 수 있음 |
| Operator profile | IDE에서 Operator를 인식하려면 `-Poperator` build가 필요 |
| JS lockfile | `pnpm install --frozen-lockfile`로 lockfile 불일치가 build 실패를 유발할 수 있음 |
| Proxy 환경 | proto lock compatibility check가 실패하면 `-DskipProtoLock=true` 검토 |

## 검증 matrix

| 변경 영역 | 최소 검증 | 추가 검증 |
| --- | --- | --- |
| Quarkus bootstrap/config | 관련 module Maven test, dev mode startup | distribution build, `kc.sh start-dev` |
| OIDC/token | endpoint/grant unit/integration test | OIDC conformance, browser flow test |
| Authentication | authenticator/required action test | browser UI flow, brute force, event assertion |
| Storage/JPA | model/provider tests, DB migration test | Postgres/MySQL/MariaDB/MSSQL/Oracle matrix |
| Infinispan/session | session tests | clustering tests, remote Infinispan |
| Admin API | Admin REST tests | Admin UI regression |
| Admin UI/Account UI | pnpm build/test, Playwright if available | Maven root build with JS enabled |
| Theme | theme verifier | manual browser check |
| Operator | unit/local_apiserver tests | local/remote cluster tests |
| Docs only | Markdown link/path sanity | no runtime test required unless docs include generated output |

## 파일 참조 색인

| 문서/파일 | 용도 |
| --- | --- |
| `docs/building.md` | source build, JDK, Maven wrapper, IDE 주의점 |
| `docs/tests.md` | 기존 testsuite/test utils 실행 |
| `quarkus/README.md` | Quarkus development mode, dist build, debug |
| `test-framework/docs/README.md` | 신규 test framework index |
| `test-framework/docs/RUNNING_TESTS.md` | test-framework server/database/browser 설정 |
| `test-framework/docs/CONFIG.md` | test-framework 설정 우선순위 |
| `testsuite/integration-arquillian/HOW-TO-RUN.md` | legacy integration-arquillian 실행 |
| `js/README.md` | JS workspace 구조 |
| `js/pom.xml` | Maven frontend build, Node/Pnpm version |
| `operator/README.md` | Operator build/test/Minikube guide |

## 작업 범위 기록

이 문서는 분석과 문서화만 수행한다. 빌드 스크립트, Maven 설정, 테스트 코드는 수정하지 않는다.
