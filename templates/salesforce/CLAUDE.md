# Salesforce 프로젝트 — Claude Code 컨텍스트

## 프로젝트 개요

Salesforce DX(SFDX) 기반 프로젝트입니다.
Windmillsoft 조직의 GitHub Issues 기반 프로젝트 관리 체계를 따르며,
MCP 서버를 통해 Claude Code가 GitHub 이슈를 직접 조회/생성/관리할 수 있습니다.

## 기술 스택

- **플랫폼**: Salesforce (API v65.0)
- **개발 도구**: Salesforce CLI (`sf`)
- **메타데이터 형식**: Source format
- **소스 경로**: `force-app/main/default/`

## Salesforce 개발 컨벤션

### Apex
- 클래스명: PascalCase (예: `AccountService`, `OpportunityTriggerHandler`)
- 테스트 클래스: `{클래스명}Test` (예: `AccountServiceTest`)
- 트리거: `{오브젝트명}Trigger` (예: `AccountTrigger`)
- 트리거 핸들러: `{오브젝트명}TriggerHandler` (예: `AccountTriggerHandler`)
- 테스트 커버리지: 최소 75% (배포 요건)

### LWC (Lightning Web Components)
- 컴포넌트명: camelCase (예: `accountList`, `opportunityCard`)
- 디렉토리: `force-app/main/default/lwc/{컴포넌트명}/`
- 필수 파일: `.js`, `.html`, `.js-meta.xml`

### 디렉토리 구조

```
force-app/main/default/
├── classes/          # Apex 클래스
├── triggers/         # Apex 트리거
├── lwc/              # Lightning Web Components
├── aura/             # Aura Components
├── objects/          # Custom Objects & Fields
├── permissionsets/   # Permission Sets
├── layouts/          # Page Layouts
├── flexipages/       # Lightning Pages
└── staticresources/  # Static Resources
```

## 자주 쓰는 SF CLI 명령

```bash
sf org login web                          # Org 로그인
sf org create scratch -f config/project-scratch-def.json -a my-scratch  # Scratch Org 생성
sf project deploy start                   # 소스 배포
sf project retrieve start                 # 소스 가져오기
sf apex run test --code-coverage          # 테스트 실행
sf org open                               # Org 열기
```

---

## MCP 도구

이 프로젝트에는 `millworks` MCP 서버가 등록되어 있습니다.
아래 도구들을 사용하여 GitHub 이슈를 관리하세요.

| 도구 | 설명 |
|------|------|
| `get_issue` | 이슈 상세 조회 (번호, 제목, 본문, 라벨, 댓글) |
| `list_issues` | 이슈 목록 조회 (상태, 라벨, 담당자 필터) |
| `list_labels` | 레포지토리 라벨 목록 |
| `get_hierarchy` | 이슈 계층 구조 (Sub-Issues 트리) |
| `create_issue` | 새 이슈 생성 |
| `create_sub_issue` | Sub-Issue 생성 + 부모 연결 |
| `update_issue` | 이슈 수정 (제목, 본문, 상태, 라벨, 담당자) |
| `assign_issue` | 담당자 배정/교체 |
| `assign_iteration` | 이슈를 프로젝트 이터레이션에 배정 (선행 작업 배치) |
| `delete_issues` | 이슈 삭제 (계층 전체 삭제 지원, 테스트 정리용) |

### Sub-Issue 연결 시 주의사항 (gh CLI)

MCP `create_sub_issue` 도구를 우선 사용하되, `gh api`로 직접 연결해야 하는 경우:

```bash
# 1. 이슈 ID 조회 (번호가 아닌 정수 ID 필요)
CHILD_ID=$(gh api repos/{owner}/{repo}/issues/{number} --jq '.id')

# 2. Sub-Issue 연결 (-F 대문자로 정수형 전달)
gh api repos/{owner}/{repo}/issues/{parent_number}/sub_issues \
  -F sub_issue_id=$CHILD_ID
```

**주의**: `-f`(소문자, 문자열) 사용 시 422 에러, 이슈 번호 직접 사용 시 404 에러 발생

### 이슈 대량 생성 규칙

1. **생성과 ID 캡처를 한 번에**: `gh api`로 생성하면 응답에서 `number`와 `id`를 동시 확보
   ```bash
   gh api repos/{owner}/{repo}/issues \
     -f title="제목" -f body="본문" \
     -f labels[]="type:개발항목" -f labels[]="status:backlog" \
     --jq '{number: .number, id: .id}'
   ```
2. **생성 순서**: 공통 모듈 L1 → 공통 L3(선행) → 업무영역 L1 → L2 → L3/L4 → Sub-Issue 연결 → MD 역참조
3. **번호-ID 매핑 유지**: 생성 단계에서 매핑을 변수로 보관하여 연결 시 재조회 방지

## 이슈 계층 용어 (한국어)

| 계층 | 한국어 | GitHub Label | 설명 |
|------|--------|-------------|------|
| Level 1 | 업무영역 | `type:업무영역` | 최상위 범주 (Epic) |
| Level 2 | 상세기능 | `type:상세기능` | 기능 단위 (User Story) |
| Level 3 | 개발항목 | `type:개발항목` | 구현 항목 (Task) |
| Level 4 | 단위작업 | `type:단위작업` | 최소 작업 단위 (SubTask) |

## 라벨 규칙

### 유형 라벨 (`type:*`)
- `type:업무영역` — 최상위 업무 범주
- `type:상세기능` — 기능 단위 항목
- `type:개발항목` — 구현해야 할 개발 항목
- `type:단위작업` — 최소 단위 작업

### 상태 라벨 (`status:*`)
- `status:backlog` — 백로그
- `status:todo` — 할 일
- `status:in-progress` — 진행 중
- `status:review` — 리뷰 중
- `status:done` — 완료

## 이슈 생성 규칙 (필수)

이슈 생성 시 반드시 `Windmillsoft/.github` 레포의 이슈 템플릿을 따라야 합니다.

### 템플릿 참조 방법

이슈를 생성하기 전에 아래 명령으로 해당 유형의 템플릿 원본을 읽고, 그 필드 구조대로 body를 작성하세요:

```bash
gh api repos/Windmillsoft/.github/contents/.github/ISSUE_TEMPLATE/{유형}.yml --jq '.content' | base64 -d
```

사용 가능한 유형: `업무영역`, `상세기능`, `개발항목`, `단위작업`, `기능요청`, `기술검토`, `버그리포트`, `변경요청`

### 유형별 기본 라벨

| 유형 | 라벨 |
|------|------|
| 업무영역 | `type:업무영역`, `status:backlog` |
| 상세기능 | `type:상세기능`, `status:backlog` |
| 개발항목 | `type:개발항목`, `status:backlog` |
| 단위작업 | `type:단위작업`, `status:todo` |
| 기능요청 | `enhancement` |
| 기술검토 | `tech-review` |
| 버그리포트 | `bug` |
| 변경요청 | `change-request` |

### 규칙
1. **필수**: `create_issue`/`create_sub_issue` 호출 전에 해당 유형의 템플릿 yml을 읽을 것
2. **필수**: 템플릿의 `validations.required: true` 필드는 반드시 채울 것
3. **필수**: 체크리스트(`- [ ]`)는 구체적인 항목으로 채울 것 (플레이스홀더 금지)
4. **필수**: 각 필드를 `## 필드명` 헤더로 구분하여 body 작성
5. **필수**: 부모 이슈의 context를 참고하여 내용을 채울 것
6. 사용자가 별도 body를 지정한 경우 해당 내용 사용 (단, 필수 필드 누락 시 보완 요청)

## 문서 기반 프로젝트 워크플로우

### 폴더 구조 규약

프로젝트의 요구사항 문서는 아래 구조를 따릅니다:

```
docs/
  requirements/
    공통모듈분석.md                  ← Claude 생성 → 설계자 수정 (프로젝트 수준, 전체 업무영역 횡단)
    공통모듈/
      UI-가이드-{패턴명}.md         ← 공통 UI 패턴 가이드라인 (커스텀리스트, 검색필터, 모달폼 등)
    {업무영역}/
      sources/
        YYYY-MM-DD-자료.md          ← 회의록, 분석 자료, 기존 시스템 문서 등
      요구사항정의서.md              ← Claude 생성 → 설계자 수정
      features/
        상세기능-{기능명}.md         ← Claude 생성 → 설계자 수정
```

### Phase 1: 요구사항 정의

#### Step 1 — 자료 등록

설계자가 업무영역별 `sources/` 폴더에 자료 MD 파일을 등록합니다.
회의록, 분석 자료, 기존 시스템 문서 등 다양한 형태의 입력을 지원합니다.
PDF 파일은 업로드 시 자동으로 MD로 변환됩니다.

#### Step 2 — 요구사항정의서 생성

설계자가 Claude에게 요청하면, 해당 업무영역의 모든 자료를 읽고 `요구사항정의서.md`를 생성합니다.

```
사용자: "고객관리 자료를 분석해서 요구사항정의서를 만들어줘"
Claude: sources/ 내 모든 MD 파일 읽기 → 요구사항정의서.md 생성
```

**덮어쓰기 규칙**: 이미 `요구사항정의서.md`가 존재하는 경우, 사용자에게 덮어쓸지 확인한 후 진행합니다. 상세기능 문서(`features/상세기능-*.md`)도 동일합니다.

#### Step 3 — 설계자 검토/수정

설계자가 `요구사항정의서.md`를 확인하고 수정 완료합니다.

#### Step 4 — 상세기능 문서 생성

설계자가 Claude에게 요청하면, **요구사항정의서 + sources/ 원본 자료를 모두 읽고** `features/` 폴더에 상세기능 MD 파일들을 생성합니다.

```
사용자: "고객관리 요구사항정의서 기반으로 상세기능 문서를 만들어줘"
Claude:
  1. 요구사항정의서.md 읽기
  2. sources/ 내 모든 MD 파일 읽기 (원본 자료 참조)
  3. features/상세기능-{기능명}.md × N 생성
```

**필수**: 상세기능 생성 시 반드시 sources/ 원본 자료도 함께 읽어야 합니다. 요구사항정의서에서 추상화되며 빠진 세부 사항을 원본에서 보완하기 위함입니다.

#### 상세기능 문서 필수 구성

각 상세기능 MD에는 반드시 **프로세스 맵** 섹션을 포함해야 합니다.
동일 기능이라도 프로필/역할에 따라 프로세스가 달라질 수 있으므로, 역할별 행동 흐름을 명시합니다.

```markdown
## 프로세스 맵

| 단계 | 영업담당 | 관리자 | 시스템 |
|------|---------|--------|--------|
| 1. 고객 정보 입력 | ● 입력 폼 작성 | | |
| 2. 중복 검증 | | | ● 전화번호/사업자번호 기준 |
| 3. 등급 심사 요청 | ● 심사 요청 버튼 | | |
| 4. 등급 승인/반려 | | ● 승인 화면에서 결정 | |
| 5. 고객 확정 | | | ● 자동 확정 + 알림 |
```

**규칙**:
1. **필수**: 해당 기능에 관여하는 모든 역할(프로필)을 열로 나열
2. **필수**: 각 단계에서 누가 무엇을 하는지 구체적으로 명시 (● + 행동 설명)
3. **필수**: 시스템 자동 처리 단계도 포함 (Trigger, Flow, 알림 등)
4. **권장**: 분기/조건이 있는 경우 별도 행으로 표시 (예: "4a. 승인 시", "4b. 반려 시")

#### Step 4-1 — 커버리지 검증

상세기능 문서 생성 완료 후, sources/ 원본 자료 대비 커버리지를 검증합니다.

```
Claude:
  1. sources/ 각 파일의 핵심 요구사항/기능 항목을 추출
  2. 생성된 상세기능 문서들에서 해당 항목이 다뤄졌는지 대조
  3. 누락/미반영 항목을 체크리스트로 사용자에게 보고
```

출력 형식:
```markdown
## 커버리지 검증 결과

### 반영 완료
- [항목] → 상세기능-{기능명}.md

### 누락/검토 필요
- [항목] (출처: {source파일명}) → 미반영 사유 또는 보완 제안
```

설계자가 누락 항목을 확인하고, 필요 시 추가 생성을 요청합니다.

#### Step 5 — 설계자 검토/수정

설계자가 각 상세기능 MD 파일을 확인하고 수정 완료합니다.

### Phase 1.5: 공통 모듈 분석

상세기능 문서가 확정된 후, 이슈 생성 전에 **전체 업무영역을 횡단**하여 공통 요소를 분석합니다.
분석 결과는 프로젝트 수준의 `docs/requirements/공통모듈분석.md` 파일로 저장합니다.

```
사용자: "공통 모듈을 분석해줘"
Claude:
  1. docs/requirements/*/features/*.md 전체 읽기 (모든 업무영역)
  2. 업무영역 간 중복/공통 요소 횡단 분석 (아래 순서 준수)
  3. docs/requirements/공통모듈분석.md 파일 생성
  4. 사용자에게 요약 보고
```

#### SF 데이터 모델 분석 순서 (필수)

공통 오브젝트/필드 식별 시 반드시 아래 순서를 따릅니다:

1. **Standard Object 우선 확인**: 요구사항을 충족하는 Standard Object가 있는지 먼저 확인 (Account, Contact, Opportunity, Contract, Case, Lead, Product2, Order 등)
2. **Standard Object + Custom Field**: Standard Object에 Custom Field 추가로 해결 가능한지 확인
3. **Custom Object 생성**: Standard로 커버 불가능한 경우에만 Custom Object 생성 (정당성을 공통모듈분석.md에 명시)

#### 분석 대상 분류

| 분류 | 유형 | 선행 여부 | 설명 |
|------|------|-----------|------|
| 플랫폼/인프라 | Multi-Currency 활성화 | 최선행 | 통화 코드별 환율 관리 (멀티 권역 시) |
| 플랫폼/인프라 | Region 필드 (Account 등) | 최선행 | 권역 판별 기준 필드 |
| 플랫폼/인프라 | RegionConfig__mdt | 최선행 | 권역별 Service 매핑/분기 기준 |
| 플랫폼/인프라 | BaseTriggerHandler | 최선행 | 권역 분기 공통 프레임워크 |
| 데이터 모델 | Custom Object | 선행 필수 | Standard Object로 불가한 경우만 |
| 데이터 모델 | Custom Field (Standard Obj에 추가) | 선행 필수 | 여러 기능이 공유하는 필드 |
| 데이터 모델 | Custom Field (Custom Obj에 추가) | 선행 필수 | Custom Object 생성과 함께 |
| 보안/권한 | Permission Set | 선행 필수 | 오브젝트/필드 접근 권한 |
| 보안/권한 | Record Type | 조건부 선행 | 2개+ 기능이 의존하는 경우만 선행 |
| 개발 | Apex 클래스 (Selector/Service/Utility) | 선행 필수 | 공통 비즈니스 로직 |
| 개발 | LWC 컴포넌트 | 선행 필수 | 공통 UI 컴포넌트 |
| 개발 | Trigger/TriggerHandler | 선행 필수 | 공통 트리거 로직 + 권역 분기 |
| 설정 | Custom Metadata Type | 선행 필수 | 권역별 세율/할인율/분기 매핑 등 |
| 설정 | Validation Rule, Flow | 조건부 선행 | 2개+ 기능이 의존하는 경우만 선행 |
| 설정 | Page Layout | 기능별 처리 | 보통 단일 기능 전용이므로 해당 이슈 내 처리 |
| UI 가이드라인 | 공통 UI 패턴 (커스텀 리스트, 검색, 모달 등) | 선행 필수 | 2개+ 기능에서 동일 UI 패턴 사용 시 |

> **원칙**: 플랫폼/인프라는 최선행(모든 기능의 기반), 2개 이상 기능이 의존하면 공통 선행, 단일 기능 전용이면 해당 기능 이슈 내 처리

#### 출력 파일: `docs/requirements/공통모듈분석.md`

```markdown
## 공통 모듈 분석 결과

### 멀티 권역 인프라 (최선행)
| 항목 | 유형 | 설명 | 비고 |
|------|------|------|------|
| Multi-Currency 활성화 | Platform 설정 | 통화 코드별 환율 관리 | 활성화 후 비활성화 불가, 신중히 결정 |
| Region__c 필드 | Custom Field (Account 등) | 권역 판별 기준 필드 | 모든 트리거 분기의 기반 |
| RegionConfig__mdt | Custom Metadata Type | 권역별 Service 클래스 매핑 | RegionCode__c, ServiceClass__c |
| RegionTaxRate__mdt | Custom Metadata Type | 권역별 세율 | RegionCode__c, TaxRate__c, EffectiveDate__c |
| BaseTriggerHandler | Apex 추상 클래스 | 권역 분기 공통 프레임워크 | 모든 TriggerHandler가 상속 |

> **참고**: 멀티 권역이 아닌 프로젝트는 이 섹션을 생략합니다.

### 데이터 모델 (선행)

#### Standard Object 활용
| Standard Object | 추가 Custom Field | 설명 | 사용 상세기능 |
|-----------------|-------------------|------|--------------|
| Account | CustomerType__c, Grade__c | 고객 유형/등급 관리 | 고객등록, 고객조회, 고객수정 |

#### Custom Object (Standard 불가 사유 포함)
| Custom Object | 사유 | 설명 | 사용 상세기능 |
|---------------|------|------|--------------|
| ConsultationLog__c | 상담 이력은 Standard에 해당 오브젝트 없음 | 상담 기록 관리 | 상담등록, 상담이력조회 |

### 보안/권한 (선행)
| 항목 | 유형 | 설명 | 사용 상세기능 |
|------|------|------|--------------|
| 고객관리_기본 | Permission Set | Account + Custom Field 접근 | 고객등록, 고객조회, 고객수정 |
| Account - 법인/개인 | Record Type | 고객 유형별 구분 | 고객등록, 고객조회 |

### 공통 Apex 클래스 (선행)
| 클래스명 | 패턴 | 설명 | 사용 상세기능 |
|----------|------|------|--------------|
| AccountSelector | Selector | Account SOQL 공통 조회 | 고객등록, 고객조회, 고객수정 |
| ValidationUtil | Utility | 전화번호, 이메일 등 공통 검증 | 고객등록, 고객수정 |

### 공통 LWC 컴포넌트 (선행)
| 컴포넌트명 | 설명 | 사용 상세기능 |
|------------|------|--------------|
| searchFilter | 목록 화면 공통 검색 UI | 고객조회, 이력조회 |

### Custom Metadata Type (선행)
| Metadata Type | 용도 | 주요 필드 | 사용 상세기능 |
|---------------|------|-----------|--------------|
| RegionConfig__mdt | 권역별 비즈니스 분기 매핑 | RegionCode__c, ServiceClass__c | 전체 (TriggerHandler 분기) |
| RegionTaxRate__mdt | 권역별 세율 | RegionCode__c, TaxRate__c, EffectiveDate__c | 주문처리, 정산 |

### 공통 설정 항목 (조건부)
| 항목 | 유형 | 선행 여부 | 설명 | 사용 상세기능 |
|------|------|-----------|------|--------------|
| 고객등급 검증 | Validation Rule | 선행 | 등급 변경 시 공통 검증 | 고객등록, 고객수정 |
| 고객 상세 레이아웃 | Page Layout | 기능별 | 고객 상세 화면 | 고객조회 |

### 공통 UI 가이드라인 (선행)
| UI 패턴 | 가이드라인 문서 | 사용 상세기능 | 주요 규격 |
|---------|---------------|--------------|----------|
| 커스텀 리스트 | `공통모듈/UI-가이드-커스텀리스트.md` | 고객조회, 이력조회, 자산관리 | 컬럼 스타일, 정렬, 페이징, 행 액션 |
| 검색 필터 | `공통모듈/UI-가이드-검색필터.md` | 고객조회, 이력조회 | 필터 UI, 검색 UX, 초기화 동작 |

> **식별 기준**: 2개+ 상세기능에서 동일 UI 패턴을 LWC로 커스텀 개발하는 경우, 가이드라인 문서를 먼저 작성.
> Standard List View로 충분한 경우는 가이드라인 불필요.
> 가이드라인 문서는 `docs/requirements/공통모듈/UI-가이드-{패턴명}.md`에 저장.
```

설계자가 공통 모듈 목록을 확인/수정하면 Phase 2에서 이를 반영하여 이슈를 생성합니다.

**필수**: 공통 모듈 분석 시 Salesforce 플랫폼 특성을 반영하여, 설정(Configuration)과 개발(Development)을 구분합니다.
**필수**: Custom Object 생성 시 Standard Object로 대체 불가능한 사유를 반드시 명시합니다.

### Phase 2: 이슈 생성 (MCP)

설계자가 Claude에게 요청하면, 업무영역 MD + 상세기능 MD + 공통 모듈 분석 결과를 기반으로 GitHub 이슈를 생성합니다.

```
사용자: "문서 기반으로 이슈를 생성해줘"
Claude:
  1. 공통모듈분석.md 읽기
  2. create_issue → "공통 모듈" L1 업무영역 이슈 생성
  3. create_sub_issue × N → 공통 모듈 L3 이슈 생성 (선행 작업)

  4. 요구사항정의서.md + features/*.md 읽기
  5. create_issue → 업무영역 L1 이슈 생성
  6. create_sub_issue × N → 상세기능 L2 이슈 생성
  7. create_sub_issue × N → L3/L4 이슈 생성 (기능 내 의존성 순서 + 공통 모듈 의존성 명시)
```

**공통 모듈 이슈 계층**:
```
L1: 공통 모듈 (type:업무영역, status:backlog)
  L3: Account Custom Fields (type:개발항목)      ← 선행
  L3: BaseTriggerHandler (type:개발항목)          ← 선행
  L3: RegionConfig__mdt (type:개발항목)           ← 선행
  L3: AccountSelector (type:개발항목)             ← 선행
  ...
```

> **참고**: 공통 모듈 L1에는 L2(상세기능)를 만들지 않고, 바로 L3(개발항목)를 하위로 연결합니다.
> 공통 모듈 L1 이슈가 이미 존재하는 경우, 새로 만들지 않고 기존 이슈에 L3를 추가합니다.

#### L3/L4 이슈 생성 규칙

개발항목(L3) 및 단위작업(L4) 이슈 생성 시, Salesforce 플랫폼 특성을 반영해야 합니다.

1. **필수**: 설정(Configuration) vs 개발(Development) 구분 — 설정으로 해결 가능한 것은 개발하지 않음
2. **필수**: 공통 모듈이 식별된 경우, 개별 개발항목 body에 의존성 명시
3. **필수**: 공통 모듈 이슈에 선행 작업 표시를 하여 우선 개발 유도
4. **필수**: 상세기능의 프로세스 맵 전체 단계를 L3/L4가 빠짐없이 커버해야 함 — 프로세스 맵의 각 단계 × 역할을 기준으로 이슈 생성
5. **필수**: 기능 내에서도 L3/L4 간 의존성 순서를 명시 — 아래 순서를 따름

##### 기능 내 L3/L4 의존성 순서

단일 상세기능(L2) 안에서도 L3/L4 이슈 간 실행 순서가 있습니다. 기능 전용 오브젝트라도 데이터 모델이 먼저 정의되어야 합니다.

```
① 데이터 모델 (Object/Field 정의)     ← 기능 내 선행
② 보안/권한 (Permission Set 등)       ← ①에 의존
③ 비즈니스 로직 (Apex, Flow, VR)      ← ①②에 의존
④ UI (LWC, Page Layout)              ← ①②③에 의존
```

**규칙**: L3 이슈 body의 `## 의존성` 섹션에 공통 모듈 의존성, 기능 내 의존성, UI 가이드라인을 모두 명시합니다.

```markdown
## 의존성
- 선행 (공통): #30 AccountSelector, #31 ValidationUtil
- 선행 (기능 내): #40 Asset Object/Field 정의, #41 Asset Permission Set
- UI 가이드라인: `공통모듈/UI-가이드-커스텀리스트.md` (#35 공통 UI: 커스텀 리스트)
```

##### 공통모듈 UI 가이드라인 자동 참조

Phase 2 이슈 생성 시, `docs/requirements/공통모듈/` 폴더에 `UI-가이드-*.md` 파일이 존재하면 자동으로 관련 L3/L4 이슈에 참조합니다.

1. **필수**: 이슈 생성 전 `docs/requirements/공통모듈/` 폴더를 스캔
2. **필수**: `UI-가이드-{패턴명}.md` 파일의 `## 적용 대상` 섹션을 읽어 어떤 기능에 적용되는지 파악
3. **필수**: 해당 기능의 L3/L4 이슈(특히 UI 관련) body에 가이드라인 문서를 `## 의존성`에 명시
4. **필수**: 가이드라인에 대응하는 공통 모듈 L3 이슈가 없으면 생성 (예: "공통 UI: 커스텀 리스트")
5. 설계자가 수동으로 만든 가이드라인도 동일하게 스캔 대상에 포함됨

> **참고**: 가이드라인 문서는 Claude가 자동 생성하거나, 설계자가 Extension의 "공통모듈 → 새 가이드라인" 기능으로 직접 등록할 수 있습니다.

##### Salesforce 구현 방식 분류

| 요구사항 유형 | 설정 (Configuration) | 개발 (Development) |
|-------------|---------------------|-------------------|
| 데이터 모델 | Custom Object/Field, Relationship | — |
| UI 화면 | Lightning App Builder, Page Layout | LWC 커스텀 컴포넌트 |
| 비즈니스 로직 (단순) | Validation Rule, Flow, Process Builder | — |
| 비즈니스 로직 (복잡) | — | Apex Class (Service 패턴) |
| 트리거 | — | Apex Trigger + TriggerHandler 패턴 |
| 데이터 조회 | — | Apex Selector 패턴 |
| 외부 연동 | Named Credential 설정 | Apex Callout + Integration 패턴 |
| 권한 | Permission Set, Profile | — |
| 환경/권역별 설정값 | Custom Metadata Type | — |
| 보고서/대시보드 | Report Type, Dashboard | — |

##### Salesforce 개발 패턴 규칙

- **Trigger**: 오브젝트당 1개 트리거, 반드시 TriggerHandler 패턴 사용
- **Selector**: SOQL은 Selector 클래스로 분리 (예: `AccountSelector`)
- **Service**: 비즈니스 로직은 Service 클래스로 분리 (예: `AccountService`)
- **Test**: 모든 Apex 클래스에 테스트 클래스 필수 (75% 이상 커버리지)
- **LWC**: 재사용 가능한 UI는 공통 LWC로 분리

##### 멀티 법인/권역 Trigger 분기 패턴

하나의 Org에 여러 법인(권역)의 비즈니스 로직이 공존하는 경우, 아래 분기 패턴을 필수로 적용합니다.

```
Trigger → TriggerHandler → 권역 판별 → RegionService 분기
```

**구조 예시**:
```
AccountTrigger (1개)
  └→ AccountTriggerHandler
       ├→ 권역 판별: record.Region__c 또는 Custom Metadata로 매핑
       ├→ AccountService_KR (한국 법인 로직)
       ├→ AccountService_JP (일본 법인 로직)
       └→ AccountService_US (미국 법인 로직)
```

**규칙**:
1. **필수**: TriggerHandler에서 레코드의 지역/법인 정보를 기준으로 해당 권역 Service를 호출
2. **필수**: 권역 공통 로직은 기본 Service 클래스에, 권역 특화 로직은 `{Object}Service_{Region}` 클래스에 분리
3. **필수**: 권역 판별 기준(필드명, 매핑 규칙)은 Custom Metadata Type으로 관리 (하드코딩 금지)
4. **권장**: 새 권역 추가 시 Service 클래스만 추가하면 되도록 설계 (Trigger/Handler 수정 불필요)

##### Custom Metadata Type 활용 규칙

환경별/권역별로 달라지는 설정값은 반드시 Custom Metadata Type을 사용합니다.

| 용도 | Custom Metadata 예시 | 하드코딩 금지 사유 |
|------|---------------------|-------------------|
| 권역별 세율 (Tax) | `RegionTaxRate__mdt` | 권역 추가/세율 변경 시 배포 없이 수정 가능 |
| 권역별 할인율 | `RegionDiscount__mdt` | 프로모션/정책 변경 빈도 높음 |
| 권역별 비즈니스 규칙 매핑 | `RegionConfig__mdt` | TriggerHandler 분기 기준 |
| 외부 시스템 엔드포인트 | `IntegrationEndpoint__mdt` | 환경(Sandbox/Prod)별 상이 |
| 공통 코드값/설정값 | `AppConfig__mdt` | 운영 중 변경 가능성 있는 상수 |

**규칙**:
1. **필수**: 세율, 할인율, 수수료율 등 비즈니스 수치는 Custom Metadata Type 사용
2. **필수**: 권역/법인별 분기 매핑 정보는 Custom Metadata Type 사용
3. **필수**: Apex 코드에 비즈니스 수치나 권역 코드를 하드코딩하지 않음
4. **권장**: Custom Metadata Type은 `{도메인}{용도}__mdt` 네이밍 규칙 적용
5. **참고**: Custom Metadata Type vs Custom Setting 선택 기준 — 배포 가능(CI/CD 포함) + 테스트에서 DML 가능 → Custom Metadata Type 우선

#### L3 이슈 body 예시 (개발항목 — 개발 유형)

```markdown
## 설명
고객 등록 화면에서 입력된 데이터를 Account 오브젝트에 저장하는 Apex Service 구현.
AccountSelector를 통해 중복 고객 조회 후, ValidationUtil로 입력값 검증.

## 기술 사항
- **구현 유형**: 개발 (Development)
- **변경 파일**: `force-app/main/default/classes/AccountRegistrationService.cls`
- **사용 기술**: Apex Service 패턴
- **의존성**:
  - 선행: #20 Account Custom Field (CustomerType__c, Grade__c)
  - 선행: #21 AccountSelector (공통 SOQL 조회)
  - 선행: #22 ValidationUtil (공통 입력 검증)
  - 선행: #23 고객관리_기본 Permission Set

## 구현 참고
- AccountSelector.findByPhone()으로 전화번호 중복 조회
- ValidationUtil.validatePhone(), validateEmail() 활용
- DML은 Service 내에서 처리, Selector는 조회 전용

## 완료 조건
- [ ] AccountRegistrationService 클래스 구현
- [ ] AccountRegistrationServiceTest 작성 (75%+ 커버리지)
- [ ] AccountSelector.findByPhone() 연동 확인
- [ ] ValidationUtil 연동 확인

## 출처
`docs/requirements/고객관리/features/상세기능-고객등록.md`
```

#### L3 이슈 body 예시 (개발항목 — 설정 유형)

```markdown
## 설명
고객 등록 시 필수 필드 누락 및 형식 오류를 Validation Rule로 검증.

## 기술 사항
- **구현 유형**: 설정 (Configuration)
- **변경 파일**: `force-app/main/default/objects/Account/validationRules/`
- **사용 기술**: Validation Rule
- **의존성**:
  - 선행: #20 Account Custom Field (CustomerType__c, Grade__c)

## 구현 참고
- CustomerType__c: 필수, 허용값 = 법인, 개인, 단체
- Grade__c: CustomerType__c가 '법인'일 때 필수
- 전화번호 형식: REGEX(Phone, "^01[016789]-?\\d{3,4}-?\\d{4}$")

## 완료 조건
- [ ] Validation Rule 생성 (Account_CustomerType_Required)
- [ ] Validation Rule 생성 (Account_Grade_Required_For_Corp)
- [ ] Sandbox 테스트 통과

## 출처
`docs/requirements/고객관리/features/상세기능-고객등록.md`
```

#### L4 이슈 body 예시 (단위작업)

```markdown
## 작업 내용
AccountSelector 클래스에 전화번호 기반 고객 조회 메서드 추가.
고객 등록 시 중복 확인에 사용.

## 구현 방법
- **파일**: `force-app/main/default/classes/AccountSelector.cls`
- **패턴**: Selector 패턴 (SOQL 전용 클래스)
- **메서드**: `findByPhone(String phone)` → `List<Account>`
- **활용할 공통 모듈**: AccountSelector 기본 골격 (#21)
- **SOQL 필드**: Id, Name, Phone, CustomerType__c, Grade__c

## 완료 조건
- [ ] findByPhone 메서드 구현
- [ ] AccountSelectorTest에 테스트 케이스 추가
- [ ] 75%+ 커버리지 확인
```

### 추적성 규칙 (필수)

문서와 이슈 간 양방향 참조를 유지해야 합니다.

#### 이슈 → 문서 (이슈 body에 출처 기록)

이슈 생성 시 body에 반드시 출처 문서 경로를 포함합니다:

```markdown
## 출처
`docs/requirements/고객관리/features/상세기능-고객등록.md`
```

#### 문서 → 이슈 (MD 파일에 이슈 번호 역참조)

이슈 생성 후, 해당 MD 파일 상단에 관련 이슈 섹션을 추가/업데이트합니다:

```markdown
## 관련 이슈
- 업무영역: #10 고객관리
- 개발항목: #15 고객 정보 입력 폼, #16 유효성 검증
- 단위작업: #17 입력 폼 UI, #18 API 연동
```

### Phase 3: 변경 관리

상세기능 문서가 변경된 경우, Claude가 영향 분석을 수행합니다.

```
사용자: "상세기능-고객등록.md가 변경되었어. 영향받는 이슈를 확인해줘"
Claude:
  1. MD 파일의 "관련 이슈" 섹션에서 이슈 번호 추출
  2. get_issue로 각 이슈 현재 상태 확인
  3. MD 변경 내용과 비교하여 영향 분석:
     - 변경된 이슈 → update_issue로 body 수정
     - 삭제된 요구사항 → update_issue로 close + change-request 라벨
     - 추가된 요구사항 → create_sub_issue로 신규 생성
  4. MD 파일의 "관련 이슈" 섹션 업데이트
```

변경 시 반드시:
1. **필수**: 변경 전 기존 이슈 상태를 먼저 확인할 것
2. **필수**: 변경 계획을 사용자에게 제시하고 승인받을 것
3. **필수**: 이슈 변경 후 MD 파일의 관련 이슈 섹션을 동기화할 것
4. **필수**: 삭제된 요구사항의 이슈는 close하되, `change-request` 라벨을 추가하여 이력 보존
5. **필수**: 이슈 close 시 사유 코멘트를 남길 것 (예: "요구사항 변경으로 불필요해짐. 출처: 상세기능-고객등록.md 2025-02-01 변경")

---

## 작업 흐름 예시

### 예시 1: 자료 → 요구사항 → 이슈 (정규 프로세스)

1. 설계자: `docs/requirements/고객관리/sources/` 에 자료 등록 (회의록, PDF 등)
2. 설계자: "자료를 분석해서 요구사항정의서를 만들어줘"
3. Claude: 자료 읽기 → `요구사항정의서.md` 생성
4. 설계자: 검토 후 수정 완료
5. 설계자: "요구사항정의서 기반으로 상세기능 문서를 만들어줘"
6. Claude: 요구사항정의서 + sources/ 읽기 → `features/상세기능-{기능명}.md` × N 생성
7. Claude: sources 대비 커버리지 검증 → 누락 항목 체크리스트 보고
8. 설계자: 누락 항목 확인 후 추가 생성 요청 또는 불필요 판단
9. 설계자: 각 상세기능 검토 후 수정 완료
10. 설계자: "공통 모듈을 분석해줘"
11. Claude: 전체 업무영역의 features/*.md 횡단 분석 → `docs/requirements/공통모듈분석.md` 생성
12. 설계자: 공통 모듈 확인/수정
13. 설계자: "문서 기반으로 이슈를 생성해줘"
14. Claude: 공통 모듈 이슈(선행) + 상세기능 이슈 + 개발항목/단위작업 이슈 생성 + 양방향 참조 기록

### 예시 2: 변경 관리

1. 설계자: `상세기능-고객등록.md` 수정 (요구사항 추가/변경)
2. 설계자: "변경된 내용으로 이슈를 업데이트해줘"
3. Claude: 관련 이슈 조회 → 영향 분석 → 변경 계획 제시
4. 설계자: 승인
5. Claude: 이슈 업데이트/생성/닫기 + MD 역참조 동기화

### 예시 3: 테스트 이슈 정리

1. 설계자: "이슈 #10과 하위 이슈를 모두 삭제해줘"
2. Claude: `delete_issues(root_issue_number: 10)` → 계층 전체 삭제
3. Claude: 도구가 반환한 "역참조 정리 필요" 목록의 MD 파일에서 삭제된 이슈 번호 제거
4. Claude: 해당 MD 파일의 `## 관련 이슈` 섹션 정리 (삭제된 이슈 행 제거)
