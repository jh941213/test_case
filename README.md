# Test Agent 시스템 문서

## 개요

Test Agent 시스템은 요구사항을 입력받아 자동으로 테스트 시나리오와 테스트 케이스를 생성하는 AI 기반 테스트 자동화 시스템입니다. 이 시스템은 3개의 전문화된 AI 에이전트가 협력하여 체계적이고 포괄적인 테스트 문서를 생성합니다.

## 시스템 아키텍처

### 핵심 구성 요소

```
agents/
├── __init__.py          # 패키지 초기화
├── agent.py            # 기본 에이전트 클래스 및 전문 에이전트들
├── workflow.py         # 워크플로우 및 오케스트레이션 로직
├── simple_test.py      # 시스템 테스트용 스크립트
├── test.ipynb         # Jupyter 노트북 테스트 환경
├── test_cases.json    # 생성된 테스트 케이스 결과
└── test_scenarios.json # 생성된 테스트 시나리오 결과
```

## 에이전트 구조

### 1. BaseAgent 클래스 (`agent.py`)

모든 전문 에이전트의 기반이 되는 기본 클래스입니다.

**주요 기능:**
- AionuLLMClient를 활용한 AI API 통신
- 스트리밍 모드 지원
- 재시도 메커니즘 (최대 3회)
- 에러 핸들링 및 로깅
- 대화 ID 관리

**핵심 메서드:**
```python
def chat(self, query: str, user: str = None, conversation_id: str = "", inputs: Optional[Dict[str, Any]] = None)
def chat_with_inputs(self, query: str, inputs: Optional[Dict[str, Any]] = None, user: str = None, conversation_id: str = "")
```

### 2. 전문 에이전트들

#### TestPlanner (테스트 계획 수립 에이전트)
- **역할**: 요구사항을 분석하여 테스트 카테고리 계획 수립
- **API Key**: `app-nvTLtHcUofxpVVLlx4KoqyqV`
- **출력**: 테스트 카테고리 목록 (JSON 형태)

**프롬프트 구조:**
- 20년 경력의 시니어 QA 테스트 플래너 역할
- 요구사항을 구조화되고 포괄적이며 추적 가능한 테스트 계획으로 변환
- KT 표준 테스트 시나리오/케이스와 1:1 추적성 보장

**주요 처리 규칙:**
- requirement_id 자동 생성: `REQ_{SYSTEM_ABBR}_{NNN}` 형식
- 중복/유사 요구사항 병합 처리
- 모든 요구사항의 카테고리 매핑 (1:1 관계)
- 우선순위 부여: high/medium/low
- 타입 분류: required/optional

**출력 JSON 구조:**
```json
{
  "system_analysis": {
    "system_name": "시스템명",
    "main_purpose": "시스템의 주요 목적", 
    "core_features": ["핵심 기능1", "핵심 기능2"],
    "complexity_level": "high/medium/low"
  },
  "requirements_mapping": [
    {
      "requirement_id": "REQ_XXX_001",
      "requirement_title": "요구사항 제목",
      "requirement_description": "상세 설명",
      "category": "카테고리명",
      "priority": "high/medium/low", 
      "business_rule": "비즈니스 규칙 요약"
    }
  ],
  "test_categories": [
    {
      "category": "카테고리명",
      "category_abbreviation": "3-4글자 약어",
      "priority": "high/medium/low",
      "type": "required/optional",
      "description": "카테고리 설명",
      "estimated_scenarios": "예상 시나리오 수",
      "key_scenarios": ["대표 시나리오1", "대표 시나리오2"],
      "related_requirements": ["REQ_XXX_001", "REQ_XXX_002"]
    }
  ],
  "functional_keywords": ["키워드1", "키워드2"],
  "testing_strategy": {
    "total_categories_estimate": "총 카테고리 수",
    "complexity_level": "high/medium/low"
  }
}
```

#### TestScenario (테스트 시나리오 작성 에이전트)
- **역할**: 카테고리별 상세 테스트 시나리오 생성
- **API Key**: `app-iud2v9FBgdgObdT9wuo7u2bf`
- **출력**: 구조화된 테스트 시나리오 (JSON 형태)

**입력 템플릿:**
```
## 원본 요구사항 정의서 : {requirements}
## 카테고리 정보 : {category}
```

**처리 방식:**
- TestPlanner에서 생성된 각 카테고리별로 개별 호출
- 원본 요구사항과 특정 카테고리 정보를 조합하여 프롬프트 생성
- 카테고리에 맞는 구체적인 테스트 시나리오 도출

**예상 출력 구조:**
```json
[
  {
    "scenario_id": "TS_TERM_001",
    "scenario_name": "약관 미동의 시 회원가입 차단 검증",
    "scenario_description": "상세 시나리오 설명",
    "requirement_id": "REQ_MEMB_005",
    "steps": ["단계1", "단계2", "단계3"],
    "expected_result": "예상 결과",
    "preconditions": "전제 조건",
    "test_data": {...},
    "priority": "high"
  }
]
```

#### TestCase (테스트 케이스 작성 에이전트)
- **역할**: 시나리오를 기반으로 실행 가능한 테스트 케이스 생성
- **API Key**: `app-tlvSXnjz9OPDHk6BW1dc6r1b`
- **출력**: 상세한 테스트 케이스 (JSON 형태)

**입력 템플릿:**
```
## 테스트 시나리오 정의서 : {scenario}
## 외부 참조문서 : {reference_doc}
```

**처리 방식:**
- TestScenario에서 생성된 각 시나리오별로 개별 호출
- 시나리오 정보와 선택적 참조 문서를 조합하여 프롬프트 생성
- 실행 가능한 상세 테스트 케이스 생성 (단계별 액션/검증 포함)

**예상 출력 구조:**
```json
[
  {
    "testcase_id": "TC_LOGIN_001_01",
    "scenario_id": "LOGIN_001", 
    "screen_id": "로그인 화면",
    "name": "정상 로그인 - 올바른 이메일과 비밀번호 입력",
    "description": "상세 테스트 케이스 설명",
    "precondition": "전제 조건",
    "steps": [
      {
        "seq": 1,
        "action": "실행할 액션",
        "expected": "예상 결과"
      }
    ],
    "input": {
      "email": "user@example.com",
      "password": "ValidPass123!"
    },
    "expected_result": "최종 예상 결과",
    "type": "positive|negative|boundary",
    "status": "pending"
  }
]
```

## 프롬프트 엔지니어링

### 프롬프트 설계 원칙

각 에이전트의 프롬프트는 다음 원칙에 따라 설계되었습니다:

1. **역할 기반 설계**: 각 에이전트는 명확한 전문 역할을 가짐
2. **구조화된 출력**: 일관된 JSON 스키마로 출력 보장
3. **추적성 보장**: 요구사항-시나리오-케이스 간 1:1 매핑
4. **품질 기준**: KT 표준 테스트 문서 품질 수준 준수

### 프롬프트 최적화 전략

#### TestPlanner 프롬프트 특징
- **상세한 역할 정의**: "20년 경력의 시니어 QA 테스트 플래너"
- **명확한 출력 제약**: JSON만 반환, 한국어 작성, 열거형 값 제한
- **일관성 규칙**: 요구사항-카테고리 매핑 검증 로직
- **추정 가이드라인**: 시나리오 수 산정을 위한 구체적 기준

#### TestScenario 프롬프트 특징
- **간결한 입력**: 요구사항과 카테고리 정보만 제공
- **맥락 유지**: 원본 요구사항 전체를 유지하여 일관성 보장
- **카테고리 집중**: 특정 카테고리에 대한 집중적 시나리오 생성

#### TestCase 프롬프트 특징
- **시나리오 기반**: 생성된 시나리오를 기반으로 구체적 케이스 도출
- **참조 문서 지원**: 외부 문서를 통한 추가 컨텍스트 제공
- **실행 가능성**: 단계별 액션과 검증 포인트 명시

### 프롬프트 템플릿 관리

```python
# 시나리오 템플릿
def scenario_template(requirements: str, category: str) -> str:
    return """
    ## 원본 요구사항 정의서 : {requirements}
    ## 카테고리 정보 : {category}
    """.format(requirements=requirements, category=category)

# 테스트 케이스 템플릿  
def test_case_template(scenario, reference_doc: str = None) -> str:
    return """
    ## 테스트 시나리오 정의서 : {scenario}
    ## 외부 참조문서 : {reference_doc}
    """.format(scenario=scenario, reference_doc=reference_doc)
```

## 워크플로우 시스템 (`workflow.py`)

### 1. 시나리오 생성 워크플로우

```python
def scenario_workflow(requirement: str, inputs: Dict[str, Any] = None) -> List[Dict[str, Any]]
```

**처리 단계:**
1. **계획 수립**: TestPlanner가 요구사항을 분석하여 테스트 카테고리 생성
2. **병렬 처리**: 각 카테고리별로 TestScenario가 동시에 시나리오 생성
3. **데이터 통합**: 생성된 시나리오들을 통합하고 메타데이터 추가
4. **결과 저장**: `test_scenarios.json` 파일로 저장

**병렬 처리 특징:**
- ThreadPoolExecutor 사용 (최대 5개 워커)
- Thread-safe 데이터 수집 (threading.Lock 사용)
- 개별 카테고리 처리 실패 시에도 전체 프로세스 계속 진행

### 2. 테스트 케이스 생성 워크플로우

```python
def case_workflow(scenarios: List[Dict[str, Any]], reference_doc: str = None, inputs: Dict[str, Any] = None) -> Dict[str, Any]
```

**처리 단계:**
1. **시나리오 분석**: 입력받은 시나리오 목록 검증
2. **병렬 케이스 생성**: 각 시나리오별로 TestCase 에이전트가 동시에 테스트 케이스 생성
3. **재시도 메커니즘**: JSON 파싱 실패 시 최대 3회 재시도
4. **데이터 정규화**: 생성된 케이스들에 순번 및 메타데이터 추가
5. **결과 저장**: `test_cases.json` 파일로 저장

## 데이터 구조

### 테스트 시나리오 구조
```json
{
  "scenario_id": "TS_TERM_001",
  "category": {
    "category": "약관 동의",
    "category_abbreviation": "TERM",
    "priority": "high",
    "type": "required"
  },
  "scenario_name": "약관 미동의 시 회원가입 차단 검증",
  "scenario_description": "상세 설명...",
  "requirement_id": "REQ_MEMB_005",
  "steps": ["단계1", "단계2", "단계3"],
  "expected_result": "예상 결과",
  "preconditions": "전제 조건",
  "test_data": {...},
  "priority": "high",
  "sno": 1
}
```

### 테스트 케이스 구조
```json
{
  "testcase_id": "TC_LOGIN_001_01",
  "scenario_id": "LOGIN_001",
  "screen_id": "로그인 화면",
  "name": "정상 로그인 - 올바른 이메일과 비밀번호 입력",
  "description": "상세 설명...",
  "precondition": "전제 조건",
  "steps": [
    {
      "seq": 1,
      "action": "실행할 액션",
      "expected": "예상 결과"
    }
  ],
  "input": {
    "email": "user@example.com",
    "password": "ValidPass123!"
  },
  "expected_result": "최종 예상 결과",
  "type": "positive|negative|boundary",
  "status": "pending",
  "sno": 1
}
```

## 핵심 기능

### 1. 병렬 처리
- **성능 최적화**: 여러 에이전트가 동시에 작업하여 처리 시간 단축
- **확장성**: ThreadPoolExecutor를 통한 동시 처리 수 조절 가능
- **안정성**: Thread-safe 데이터 수집 및 에러 격리

### 2. 에러 핸들링
- **재시도 메커니즘**: API 호출 실패 시 자동 재시도
- **Graceful Degradation**: 일부 실패 시에도 전체 프로세스 계속 진행
- **상세 로깅**: 각 단계별 상세한 로그 기록

### 3. JSON 파싱 및 검증
- **유연한 파싱**: 다양한 JSON 구조 지원
- **데이터 검증**: 필수 필드 존재 여부 확인
- **타입 안전성**: 예상치 못한 데이터 구조에 대한 방어 코드

## 사용 방법

### 1. 기본 사용법
```python
from workflow import scenario_workflow, case_workflow

# 1단계: 테스트 시나리오 생성
scenarios = scenario_workflow("회원가입 기능을 테스트하고 싶습니다.")

# 2단계: 테스트 케이스 생성
test_cases = case_workflow(scenarios)
```

### 2. 간단한 테스트 실행
```bash
cd swe/test_agent/agents
python simple_test.py
```

### 3. Jupyter 노트북 환경
`test.ipynb` 파일을 통해 대화형 환경에서 테스트 가능

### 4. 실제 사용 예시

**입력 요구사항:**
```
회원가입 기능을 테스트하고 싶습니다.
- 이메일 인증 필요
- 약관 동의 필수
- 중복 이메일 차단
```

**처리 과정:**
1. **TestPlanner**: 요구사항 분석 → 카테고리 도출 (인증, 약관동의, 접근제어 등)
2. **TestScenario**: 각 카테고리별 시나리오 생성 (병렬 처리)
3. **TestCase**: 각 시나리오별 상세 테스트 케이스 생성 (병렬 처리)

**출력 결과:**
- `test_scenarios.json`: 4개 시나리오 생성
- `test_cases.json`: 18개 테스트 케이스 생성

```

## 출력 파일

### test_scenarios.json
- 생성된 모든 테스트 시나리오
- 카테고리별 분류 및 우선순위 정보
- 요구사항 추적성 정보

### test_cases.json
- 실행 가능한 상세 테스트 케이스
- 단계별 액션 및 예상 결과
- 테스트 데이터 및 전제 조건

## 로깅 및 모니터링

시스템은 다음과 같은 로그 정보를 제공합니다:
- 워크플로우 진행 상황
- 각 에이전트의 처리 결과
- 에러 발생 시 상세 정보
- 성능 메트릭 (처리 시간, 생성된 항목 수)

## 확장성 및 커스터마이징

### 새로운 에이전트 추가
`BaseAgent`를 상속받아 새로운 전문 에이전트 생성 가능

### 워크플로우 수정
`workflow.py`의 함수들을 수정하여 처리 로직 변경 가능

### 템플릿 커스터마이징
`scenario_template()`, `test_case_template()` 함수를 통해 프롬프트 템플릿 수정 가능

## 주의사항

1. **API 키 관리**: 각 에이전트별로 다른 API 키 사용
2. **네트워크 의존성**: AI API 호출을 위한 안정적인 네트워크 연결 필요
3. **리소스 사용량**: 병렬 처리로 인한 메모리 및 네트워크 사용량 고려
4. **JSON 파싱**: AI 응답의 JSON 형식이 일관되지 않을 수 있어 파싱 로직 강화 필요

## 문제 해결

### 일반적인 문제들
1. **JSON 파싱 에러**: AI 응답 형식 불일치 → 재시도 메커니즘 활용
2. **API 호출 실패**: 네트워크 또는 API 키 문제 → 로그 확인 및 재시도
3. **빈 결과**: 프롬프트 템플릿 또는 입력 데이터 검토 필요

### 디버깅 팁
- 로그 레벨을 DEBUG로 설정하여 상세 정보 확인
- 개별 에이전트 테스트를 통한 문제 격리
- JSON 파일 출력 결과 직접 검증
