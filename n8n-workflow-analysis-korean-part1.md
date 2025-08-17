# SMART DUR n8n 워크플로우 상세 분석 (Part 1)

## 개요
이 문서는 SMART DUR 환자 맞춤형 의약품안심복용 이미지 생성 워크플로우에 대한 포괄적인 분석을 제공합니다. 각 노드는 목적, 구성, 전체 자동화 프로세스에서의 역할을 포함하여 상세히 검토됩니다.

## 워크플로우 아키텍처

### 고수준 흐름
```
Google Sheets Trigger → 데이터 처리 → AI 이미지 생성 → 저장 → 문서화
```

### 노드 카테고리
1. **트리거 노드**: 워크플로우 실행 시작
2. **데이터 처리 노드**: 데이터 파싱, 변환, 준비
3. **AI 생성 노드**: 프롬프트 생성 및 이미지 생성
4. **저장 노드**: 파일 및 폴더 관리
5. **문서화 노드**: 스프레드시트 업데이트 및 진행 상황 추적
6. **오류 처리 노드**: 실패 및 재시도 관리

## 상세 노드 분석

### 1. Google Sheets Trigger
**노드 ID**: `034cab07-43b9-41fb-b56e-50edc6d401de`  
**타입**: `n8n-nodes-base.googleSheetsTrigger`  
**버전**: 1.1  
**위치**: [-60, 260]

**목적**: Google Sheets의 새 데이터 항목을 모니터링하고 워크플로우 실행을 트리거합니다.

**구성 세부사항**:
- **폴링 간격**: 매분 (`everyMinute`)
- **대상 스프레드시트**: `smartdur_image_generation` (ID: `1aVeyFfPfGUz224lN69QPh20zxSyfhEzqU79u3SB0WGI`)
- **대상 시트**: `Sheet1` (gid=0)
- **트리거 이벤트**: `rowAdded` - 새 행이 추가될 때 활성화

**기술적 구현**:
```json
{
  "pollTimes": [{"mode": "everyMinute"}],
  "documentId": "1aVeyFfPfGUz224lN69QPh20zxSyfhEzqU79u3SB0WGI",
  "sheetName": "gid=0",
  "event": "rowAdded"
}
```

**워크플로우에서의 역할**: 이미지 생성이 필요한 새 환자 데이터를 지속적으로 모니터링하는 진입점입니다.

---

### 2. 조건부 필터 (If)
**노드 ID**: `3662a3f3-4712-4e5f-adad-74f79ca7536f`  
**타입**: `n8n-nodes-base.if`  
**버전**: 2.2  
**위치**: [200, 260]

**목적**: 들어오는 데이터를 필터링하여 아직 완료되지 않은 행만 처리합니다.

**구성 세부사항**:
- **조건**: `is_finished` 컬럼이 `"finished"`와 같음
- **타입 검증**: 느슨함 (유연한 타입 매칭 허용)
- **대소문자 구분**: 활성화

**조건 로직**:
```javascript
// 단순화된 조건 표현
if ($json.is_finished === "finished") {
  // 처리 건너뛰기 - 이미 완료됨
} else {
  // 처리 계속 - 새롭거나 불완전한 데이터
}
```

**워크플로우에서의 역할**: 이미 완료된 행의 중복 처리를 방지하여 워크플로우 효율성을 보장합니다.

---

### 3. 행 분할기
**노드 ID**: `311d2dc2-99db-4c7c-965e-db1584b2530c`  
**타입**: `n8n-nodes-base.splitInBatches`  
**버전**: 3  
**위치**: [480, 280]

**목적**: 들어오는 데이터를 처리 가능한 배치로 분할합니다.

**구성 세부사항**:
- **배치 크기**: 배치당 5개 항목 (기본값)
- **재설정**: 비활성화 (실행 간 상태 유지)

**기술적 구현**:
```json
{
  "options": {
    "reset": false
  }
}
```

**워크플로우에서의 역할**: 관리 가능한 처리 부하를 보장하고 API 속도 제한 문제를 방지합니다.

---

### 4. JSON 데이터 파서
**노드 ID**: `176022a7-1e82-4a60-95e4-6e37eeb4ac0a`  
**타입**: `n8n-nodes-base.code`  
**버전**: 2  
**위치**: [760, 340]

**목적**: SMART_DUR_json 컬럼의 복잡한 JSON 데이터를 파싱하고 각 DUR 경고와 결과 조합에 대한 개별 처리 항목을 생성합니다.

**코드 분석**:
```javascript
// 주요 기능 분석:

// 1. 기본 환자 정보 추출
const researcherNo = originalRow['연구자등록번호'] || originalRow.연구자등록번호 || '';
const age = originalRow.age || originalRow.Age || '';
const gender = originalRow.gender || originalRow.Gender || '';

// 2. SMART_DUR_json 컬럼 파싱
let arr;
try {
  let cleanedJson = rawJson;
  if (typeof rawJson === 'string') {
    // 이중 따옴표 제거 및 JSON 문자열 정리
    cleanedJson = rawJson.replace(/^\"|\"$/g, '').replace(/\"\"/g, '\"');
  }
  arr = typeof cleanedJson === 'string' ? JSON.parse(cleanedJson) : cleanedJson;
} catch (e) {
  throw new Error('SMART_DUR_json 컬럼의 JSON 파싱 실패: ' + e.message);
}

// 3. 각 식별자와 결과 조합 처리
for (let identIdx = 0; identIdx < arr.length; identIdx++) {
  const entry = arr[identIdx];
  const identifier = entry.identifier || '';
  
  // 각 target_outcome 처리
  for (let outcomeIdx = 0; outcomeIdx < targetOutcomeArr.length; outcomeIdx++) {
    const outcome = targetOutcomeArr[outcomeIdx];
    
    // 구조화된 결과 객체 생성
    const result = {
      json: {
        identifier,
        identifierIndex: identIdx + 1,
        totalIdentifiers: arr.length,
        outcomeIndex: outcomeIdx + 1,
        totalOutcomes: targetOutcomeArr.length,
        age,
        gender,
        // ... 추가 필드
      }
    };
    
    results.push(result);
  }
}
```

**주요 특징**:
- **강력한 JSON 파싱**: 따옴표 이스케이핑으로 잘못된 JSON 문자열 처리
- **다단계 처리**: 환자당 여러 식별자와 식별자당 여러 결과 지원
- **포괄적인 데이터 추출**: 모든 관련 환자 및 약물 정보 캡처
- **오류 처리**: 상세한 오류 메시지와 함께 우아한 실패

**출력 구조**: 다음을 포함하는 객체 배열을 생성합니다:
- 환자 인구통계 (나이, 성별)
- DUR 식별자 정보
- 약물 세부사항 (object_A, object_B)
- 목표 결과 데이터 (증상, 심각도, 권장사항)
- 처리 메타데이터 (인덱스, 타임스탬프)

**워크플로우에서의 역할**: 원시 스프레드시트 데이터를 구조화되고 처리 가능한 항목으로 변환하는 핵심 데이터 변환 노드입니다.

---

### 5. 식별자 분할기
**노드 ID**: `88588455-efe3-4f1d-b686-24928fc6b048`  
**타입**: `n8n-nodes-base.splitInBatches`  
**버전**: 3  
**위치**: [980, 420]

**목적**: 파싱된 데이터를 더욱 분할하여 각 식별자를 개별적으로 처리합니다.

**구성 세부사항**:
- **배치 크기**: 배치당 5개 항목
- **재설정**: 비활성화

**워크플로우에서의 역할**: 각 DUR 경고가 별도로 처리되도록 보장하여 데이터 무결성을 유지하고 개별 오류 처리를 허용합니다.

---

### 6. 폴더 검색
**노드 ID**: `f000384a-b4a2-4ba2-9759-d9b14293bc85`  
**타입**: `n8n-nodes-base.googleDrive`  
**버전**: 3  
**위치**: [1400, 520]

**목적**: Google Drive에서 기존 연구자별 폴더를 검색합니다.

**구성 세부사항**:
- **리소스**: `fileFolder` (검색 작업)
- **쿼리**: 파싱된 데이터의 연구자 등록 번호
- **검색 범위**: 특정 상위 폴더 (`2025_08_DUR_이미지생성`)
- **필터**: 폴더만

**기술적 구현**:
```json
{
  "resource": "fileFolder",
  "queryString": "={{ $json['연구자등록번호'] }}",
  "returnAll": true,
  "filter": {
    "folderId": "1tnSEtkvGKgvZ365BHN42OKlRmEg-RhAU",
    "whatToSearch": "folders"
  }
}
```

**워크플로우에서의 역할**: 연구자를 위한 새 폴더를 생성할지 또는 기존 폴더를 사용할지 결정합니다.

---

### 7. 폴더 존재 확인
**노드 ID**: `084d3588-3da5-400f-ad90-71b4f6453f55`  
**타입**: `n8n-nodes-base.if`  
**버전**: 2.2  
**위치**: [1880, 500]

**목적**: 연구자 폴더가 이미 존재하는지에 따라 워크플로우를 라우팅합니다.

**구성 세부사항**:
- **조건**: 폴더 ID가 존재하는지 확인
- **타입 검증**: 엄격함
- **대소문자 구분**: 활성화

**조건 로직**:
```javascript
// 폴더 검색이 유효한 폴더 ID를 반환했는지 확인
if ($json.id exists and is not empty) {
  // 기존 폴더 사용
} else {
  // 새 폴더 생성
}
```

**워크플로우에서의 역할**: 기존 폴더를 사용할지 또는 새 폴더를 생성할지에 대한 조건부 라우팅입니다.

---

### 8. 폴더 생성
**노드 ID**: `4c7a3673-5dff-48ae-8d91-002840fd3cde`  
**타입**: `n8n-nodes-base.googleDrive`  
**버전**: 3  
**위치**: [2180, 600]

**목적**: 존재하지 않을 때 Google Drive에 새 연구자별 폴더를 생성합니다.

**구성 세부사항**:
- **리소스**: `folder`
- **이름**: 연구자 등록 번호
- **상위 폴더**: `2025_08_DUR_이미지생성` (ID: `1tnSEtkvGKgvZ365BHN42OKlRmEg-RhAU`)

**기술적 구현**:
```json
{
  "resource": "folder",
  "name": "={{ $('Split Identifiers').item.json['연구자등록번호'] }}",
  "driveId": "My Drive",
  "folderId": "1tnSEtkvGKgvZ365BHN42OKlRmEg-RhAU"
}
```

**워크플로우에서의 역할**: 각 연구자를 위한 전용 폴더를 생성하여 적절한 파일 구성을 보장합니다.

---

### 9. 데이터 병합기
**노드 ID**: `044ba1e8-2dfc-49c1-8fd3-c64bd0b77200`  
**타입**: `n8n-nodes-base.code`  
**버전**: 2  
**위치**: [2180, 420]

**목적**: 파싱된 환자 데이터를 폴더 정보와 결합하여 처리합니다.

**코드 분석**:
```javascript
// 폴더 ID를 파싱된 데이터와 병합
const folderId = $json.id;
const parseData = $('Split Identifiers').item.json;

return {
  json: {
    ...parseData,
    folderId: folderId,
    // 처리 메타데이터 추가
    processingTimestamp: new Date().toISOString(),
    workflowExecutionId: $executionId
  }
};
```

**주요 특징**:
- **데이터 통합**: 환자 데이터를 폴더 정보와 결합
- **메타데이터 추가**: 추적을 위한 타임스탬프 및 실행 ID 추가
- **깔끔한 데이터 구조**: 체계적인 데이터 흐름 유지

**워크플로우에서의 역할**: AI 생성 단계를 위한 통합 데이터를 준비합니다.

---

### 10. 프롬프트 생성 에이전트
**노드 ID**: `38143541-4225-46bf-9875-bd3be5d3dabc`  
**타입**: `@n8n/n8n-nodes-langchain.agent`  
**버전**: 2  
**위치**: [2420, 420]

**목적**: OpenAI의 GPT-5를 사용하여 의료 일러스트레이션 생성을 위한 상세하고 구조화된 프롬프트를 생성합니다.

**구성 세부사항**:
- **모델**: OpenAI Chat Model (AI Language Model을 통해 연결)
- **프롬프트 타입**: Define (사용자 정의 프롬프트 템플릿)
- **최대 반복**: 10

**프롬프트 템플릿 분석**:
```text
당신은 의료 교육 일러스트레이션을 위한 이미지 프롬프트 생성기입니다.

당신의 목표:  
환자 사례에 대한 다음 입력 정보가 주어지면, 고령자 환자가 이해하기 쉽고 친근한 단순하고 도식적인 단일 패널 만화 스타일 의료 일러스트레이션을 위한 **안정적이고 일관되고 명확한 이미지 생성 프롬프트**를 생성하세요.

### 고정 스타일 요구사항:
- 스타일: 단순하고 도식적이며 단일 패널 만화, 플랫 벡터, 부드러운 파스텔 색상, 두꺼운 깔끔한 윤곽선, 최소한의 음영, 복잡한 배경 없음.
- 구성: 중앙에 환자 캐릭터, 문제의 원인이 되는 객체(들)을 단순하고 인식 가능한 아이콘으로 명확하게 표시, 주의해야 할 증상을 환자 주변의 작은 아이콘이나 말풍선으로 표시, 증상과 일치하는 표정과 몸짓, 하단에 권장사항이 포함된 작은 안내판이나 말풍선.
- 톤: 친근하고 이해하기 쉽고 위협적이지 않음.
- 초점: 환자와 핵심 객체, 불필요한 세부사항 피하기.
- 식별자 라벨 추가: {{ $json.identifier }}를 우상단 모서리에 표시

### 가변 입력 (제공된 데이터로 플레이스홀더 교체):
- 나이: {{ $json.age }}
- 성별: {{ $json.gender }}
- 객체 A: {{ $json.object_A_name }}
- 객체 B: {{ $json.object_B_name }}
- 주의해야 할 증상: {{ $json.watchful_symptoms.join(', ') }}
- FSN: {{ $json.singularfsn }}
- 권장사항: {{ $json.recommendation[0]?.guidance || $json.recommendation[0]?.action || '' }}
- 식별자: {{ $json.identifier }}

### 출력 프롬프트 구조:
단순하고 도식적인 단일 패널 만화 스타일 의료 일러스트레이션.  
플랫 벡터 스타일, 부드러운 파스텔 색상, 두꺼운 깔끔한 윤곽선, 최소한의 음영, 깨끗한 흰색 배경.  
식별자 "{{ $json.identifier }}"를 우상단 모서리에 작은 라벨로 표시.
중앙에 {age}세 {gender} 환자를 묘사.  
{objectA}{ 및 {objectB}가 제공된 경우}를 문제의 명확한 원인으로 단순하고 인식 가능한 아이콘으로 표시.  
환자 주변에 주의해야 할 증상을 나타내는 작은 아이콘이나 말풍선 표시: {watchful_symptoms}.  
표정과 몸짓은 증상과 일치해야 함.  
{fsn}으로 설명된 의료 문제를 시각적으로 암시하되, 방해가 되는 이미지는 표시하지 않음.  
하단에 권장사항이 포함된 작은 안내판: "{recommendation}".  
고령자 환자 교육을 위한 친근하고 이해하기 쉬운 스타일로 환자와 핵심 객체에 초점을 맞춘 깔끔한 레이아웃을 보장.
```

**주요 특징**:
- **구조화된 프롬프팅**: 일관된 의료 일러스트레이션 스타일 보장
- **환자별 맞춤 콘텐츠**: 개별 환자 특성 통합
- **의학적 정확성**: 환자 친화적이면서도 임상적 관련성 유지
- **스타일 일관성**: 모든 생성된 이미지에 대한 균일한 시각적 접근법 강제

**워크플로우에서의 역할**: DALL-E가 적절한 의료 일러스트레이션을 생성하도록 안내하는 지능적이고 맥락을 인식하는 프롬프트를 생성합니다.

---

### 11. OpenAI Chat Model
**노드 ID**: `f706bcb8-1aef-43ba-bd40-77dde6e2379d`  
**타입**: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
**버전**: 1.2  
**위치**: [2420, 660]

**목적**: 프롬프트 생성을 위한 언어 모델 인터페이스를 제공합니다.

**구성 세부사항**:
- **모델**: GPT-5
- **자격 증명**: OpenAI API 계정

**워크플로우에서의 역할**: 고급 언어 이해 기능으로 프롬프트 생성 에이전트에 힘을 실어줍니다.

---

### 12. 이미지 생성
**노드 ID**: `5ae92208-14c9-4537-95aa-c0cf3d12ecea`  
**타입**: `@n8n/n8n-nodes-langchain.openAi`  
**버전**: 1.8  
**위치**: [2760, 320]

**목적**: 생성된 프롬프트를 기반으로 OpenAI의 DALL-E를 사용하여 의료 일러스트레이션을 생성합니다.

**구성 세부사항**:
- **리소스**: `image`
- **모델**: `gpt-image-1` (DALL-E)
- **프롬프트**: 프롬프트 생성 에이전트의 출력
- **옵션**:
  - 품질: 고품질
  - 크기: 1024x1024

**기술적 구현**:
```json
{
  "resource": "image",
  "model": "gpt-image-1",
  "prompt": "={{ $json.output }}",
  "options": {
    "quality": "high",
    "size": "1024x1024"
  }
}
```

**주요 특징**:
- **고품질 출력**: 명확한 의료 일러스트레이션을 위한 1024x1024 해상도
- **AI 기반 생성**: DALL-E의 고급 이미지 생성 기능 활용
- **프롬프트 기반**: 일관된 스타일과 콘텐츠를 보장하는 구조화된 프롬프트 사용

**워크플로우에서의 역할**: 최종 의료 일러스트레이션을 생성하는 핵심 이미지 생성 노드입니다.

---

## 워크플로우 실행 흐름 (Part 1)

### 1단계: 트리거 및 초기 처리
1. **Google Sheets Trigger**가 새 행 모니터링
2. **조건부 필터**가 이미 처리된 행인지 확인
3. **행 분할기**가 데이터를 관리 가능한 배치로 분할
4. **JSON 파서**가 환자 데이터 추출 및 구조화

### 2단계: 폴더 관리
1. **식별자 분할기**가 각 DUR 경고를 개별적으로 처리
2. **폴더 검색**이 기존 연구자 폴더 찾기
3. **폴더 존재 확인**이 라우팅 결정
4. **폴더 생성** (필요시) 새 폴더 생성
5. **데이터 병합기**가 환자 데이터를 폴더 정보와 결합

### 3단계: AI 이미지 생성
1. **프롬프트 생성 에이전트**가 GPT-5를 사용하여 구조화된 프롬프트 생성
2. **이미지 생성**이 DALL-E를 사용하여 의료 일러스트레이션 생성

---

**문서 버전**: 1.0  
**최종 업데이트**: 2025-08-17  
**워크플로우 버전**: 2.0  
**상태**: 프로덕션 준비 완료
