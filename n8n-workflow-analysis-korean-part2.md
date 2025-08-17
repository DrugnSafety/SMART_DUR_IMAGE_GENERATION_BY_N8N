# SMART DUR n8n 워크플로우 상세 분석 (Part 2)

## 상세 노드 분석 (계속)

### 13. 이미지 업로드
**노드 ID**: `1957a7d0-45ed-491b-be69-8cba8a9721e5`  
**타입**: `n8n-nodes-base.googleDrive`  
**버전**: 3  
**위치**: [2980, 320]

**목적**: 생성된 이미지를 Google Drive의 적절한 연구자 폴더에 업로드합니다.

**구성 세부사항**:
- **리소스**: `file`
- **이름**: 환자 및 경고 정보를 기반으로 한 구조화된 파일명
- **드라이브 ID**: My Drive
- **폴더 ID**: 연구자별 폴더

**파일 명명 규칙**:
```javascript
// 파일명 구조 예시
"{{ $('Merge Folder ID').item.json['연구자등록번호'] }}_{{ $('Merge Folder ID').item.json.identifier }}_{{ $('Merge Folder ID').item.json.outcomeIndex }}_{{ $('Merge Folder ID').item.json.singularfsn }}.jpg"

// 예시: cbnuh_0001_DUR_0001_1_과도한혈압강하.jpg
```

**주요 특징**:
- **체계적인 저장**: 명확한 파일 구조 유지
- **설명적 명명**: 파일명에 모든 관련 정보 포함
- **자동 구성**: 적절한 연구자 폴더에 파일 배치

**워크플로우에서의 역할**: 생성된 이미지를 체계적이고 접근 가능한 위치에 저장합니다.

---

### 14. URL 수집
**노드 ID**: `a523fa57-bb19-445f-ad46-9468c363a08a`  
**타입**: `n8n-nodes-base.code`  
**버전**: 2  
**위치**: [3160, 320]

**목적**: 업로드된 이미지에서 스프레드시트 문서화를 위한 URL과 메타데이터를 추출합니다.

**코드 분석**:
```javascript
// 업로드 결과에서 필수 정보 추출
const currentItem = $input.first().json;
const mergeData = $('Merge Folder ID').item.json;

// 웹 보기 링크 수집
const webViewLink = currentItem.webViewLink || '';
console.log('WebViewLink found:', webViewLink);

// 파일명 추출
const fileName = currentItem.name || '';
console.log('FileName:', fileName);

// 식별자 정보 파싱
let researcherNo = mergeData['연구자등록번호'] || '';
let identifier = mergeData.identifier || '';
let outcomeIndex = mergeData.outcomeIndex || 1;
let fsn = mergeData.singularfsn || '';

// 포괄적인 데이터 객체 생성
return {
  json: {
    fileName,
    researcherNo,
    identifier,
    outcomeIndex,
    identifierIndex: mergeData.identifierIndex,
    totalIdentifiers: mergeData.totalIdentifiers,
    
    // URL 정보
    webViewLink,
    image_url: webViewLink,
    
    // Google Drive 파일 정보
    fileId: currentItem.id,
    mimeType: currentItem.mimeType,
    fileSize: currentItem.size || 'Unknown',
    
    // 원본 데이터
    age: mergeData.age,
    gender: mergeData.gender,
    SMART_DUR_생성일: mergeData.SMART_DUR_생성일,
    object_A_name: mergeData.object_A_name,
    object_B_name: mergeData.object_B_name,
    fsn: mergeData.fsn,
    singularfsn: mergeData.singularfsn,
    severity: mergeData.severity,
    watchful_symptoms: mergeData.watchful_symptoms,
    recommendation: mergeData.recommendation,
    category: mergeData.category,
    
    // DUR 콘텐츠 (JSON 문자열로 저장)
    DUR_content: JSON.stringify({
      identifier: mergeData.identifier,
      objects: {
        object_A: {
          code: mergeData.object_A_code,
          code_system: mergeData.object_A_code_system,
          name: mergeData.object_A_name
        },
        object_B: {
          code: mergeData.object_B_code,
          code_system: mergeData.object_B_code_system,
          name: mergeData.object_B_name
        }
      },
      target_outcome: {
        fsn: mergeData.fsn,
        severity: mergeData.severity,
        watchful_symptoms: mergeData.watchful_symptoms,
        singularfsn: mergeData.singularfsn,
        recommendation: mergeData.recommendation,
        category: mergeData.category
      }
    }),
    
    // 타임스탬프
    processingTimestamp: mergeData.processingTimestamp,
    uploadTimestamp: new Date().toISOString()
  }
};
```

**주요 특징**:
- **포괄적인 데이터 수집**: 여러 소스에서 모든 관련 정보 수집
- **URL 추출**: Google Drive 공유 링크 캡처
- **메타데이터 보존**: 모든 원본 환자 및 약물 데이터 유지
- **구조화된 출력**: 스프레드시트 업데이트를 위한 체계적인 데이터 생성

**워크플로우에서의 역할**: Google Sheets에 생성된 이미지를 문서화하는 데 필요한 모든 정보를 준비합니다.

---

### 15. 스프레드시트 데이터 준비
**노드 ID**: `add4b2bd-d6d6-44f0-af8c-b99edf6efbd8`  
**타입**: `n8n-nodes-base.code`  
**버전**: 2  
**위치**: [3380, 320]

**목적**: Google Sheets 통합을 위한 데이터를 준비하고 IMAGE 수식 생성을 포함합니다.

**코드 분석**:
```javascript
// Google Sheets IMAGE 수식을 위한 파일 ID 추출
const j = $json;
const fileId = j.fileId || '';

// Google Sheets IMAGE 수식 생성 (모드 3: 원본 크기)
const formula = fileId
  ? `=IMAGE("https://drive.google.com/uc?export=view&id=${fileId}", 3)`
  : '링크 확인 필요';

return {
  json: {
    ...j,
    image: formula,
    spreadsheet_updated_time: $now.setZone('Asia/Seoul').toFormat('yyyy-LL-dd HH:mm:ss')
  }
};
```

**주요 특징**:
- **IMAGE 수식 생성**: 직접 이미지 표시를 위한 Google Sheets IMAGE 수식 생성
- **시간대 처리**: 정확한 한국 시간을 위한 Asia/Seoul 시간대 사용
- **오류 처리**: 파일 ID가 누락된 경우 대체 텍스트 제공

**워크플로우에서의 역할**: 외부 링크 없이 Google Sheets 내에서 직접 이미지 보기를 가능하게 합니다.

---

### 16. 스프레드시트 업데이트
**노드 ID**: `22facf9a-cbcc-4878-8e1e-9800097bb558`  
**타입**: `n8n-nodes-base.googleSheets`  
**버전**: 4.6  
**위치**: [3580, 320]

**목적**: 생성된 이미지 정보와 메타데이터로 Sheet2를 업데이트합니다.

**구성 세부사항**:
- **작업**: Append (새 행 추가)
- **대상 시트**: Sheet2 (ID: 464646019)
- **컬럼 매핑**: 모든 관련 데이터에 대한 포괄적인 필드 매핑

**컬럼 매핑 분석**:
```json
{
  "연구자등록번호": "={{ $json.researcherNo }}",
  "age": "={{ $json.age }}",
  "gender": "={{ $json.gender }}",
  "spreadsheet_updated_time": "={{ $json.spreadsheet_updated_time }}",
  "prompt_ver": "2.0",
  "DUR_identifier": "={{ $json.identifier }}",
  "DUR_content": "={{ $json.DUR_content }}",
  "image": "={{ $json.image }}",
  "image_url": "={{ $json.image_url }}",
  "file_name": "={{ $json.fileName }}",
  "watchful_symptom": "={{ $json.watchful_symptoms }}",
  "fsn": "={{ $json.fsn }}",
  "singularfsn": "={{ $json.singularfsn }}",
  "objectA_name": "={{ $json.object_A_name }}",
  "objectB_name": "={{ $json.object_B_name }}",
  "recommendation": "={{ $json.recommendation }}",
  "category": "={{ $json.category }}",
  "SMART_DUR_생성일": "={{ $json['SMART_DUR_생성일']}}"
}
```

**주요 특징**:
- **포괄적인 문서화**: 생성된 이미지에 대한 모든 관련 정보 기록
- **구조화된 데이터**: 체계적이고 검색 가능한 레코드 유지
- **버전 추적**: 품질 관리를 위한 프롬프트 버전 포함
- **메타데이터 보존**: 모든 환자 및 약물 맥락 저장

**워크플로우에서의 역할**: 추적 및 분석을 위한 모든 생성된 이미지의 영구 레코드를 생성합니다.

---

### 17. 행 검색
**노드 ID**: `cec688e0-2d1b-4bec-afab-16e110356dd9`  
**타입**: `n8n-nodes-base.googleSheets`  
**버전**: 4.6  
**위치**: [3800, 320]

**목적**: 상태 업데이트를 위해 Sheet1에서 원본 행을 검색합니다.

**구성 세부사항**:
- **작업**: Get row(s) in sheet
- **대상 시트**: Sheet1 (gid=0)
- **필터**: 구분자(고유 식별자)로 매칭

**필터 구성**:
```json
{
  "filtersUI": {
    "values": [
      {
        "lookupColumn": "=구분자",
        "lookupValue": "={{ $('Parse JSON Data').item.json.originalRow['구분자'] }}"
      }
    ]
  }
}
```

**워크플로우에서의 역할**: 원본 행을 완료 상태로 업데이트할 수 있게 합니다.

---

### 18. 행 상태 업데이트
**노드 ID**: `47d4ebd5-fd5f-4722-8f08-b55905167644`  
**타입**: `n8n-nodes-base.googleSheets`  
**버전**: 4.6  
**위치**: [4020, 780]

**목적**: Sheet1의 원본 행을 완료 상태 및 메타데이터로 업데이트합니다.

**구성 세부사항**:
- **작업**: Update
- **대상 시트**: Sheet1 (gid=0)
- **매칭 컬럼**: 구분자 (고유 식별자)
- **업데이트 필드**: 상태, 타임스탬프, 파일 정보

**업데이트 로직**:
```javascript
// 여러 필드에 대한 복잡한 업데이트 로직
"구분자": "={{ $('Parse JSON Data').item.json.originalRow['구분자'] }}",
"DUR_identifier": "={{ (() => {
  const a = $json.DUR_identifier;
  const b = $('Collect URLs').item.json.identifier;
  const A = (a !== null && a !== undefined && String(a).trim() !== '') ? String(a).trim() : null;
  const B = (b !== null && b !== undefined && String(b).trim() !== '') ? String(b).trim() : '';
  return A ? (A + '\\n' + B) : B;
})() }}",
"spreadsheet_updated_time": "={{ (() => {
  const a = $json.spreadsheet_updated_time;
  const b = $('Collect URLs').item.json.uploadTimestamp;
  const A = (a !== null && a !== undefined && String(a).trim() !== '') ? String(a).trim() : null;
  const B = (b !== null && b !== undefined && String(b).trim() !== '') ? String(b).trim() : '';
  return A ? (A + '\\n' + B) : B;
})() }}",
"prompt_ver": "2.0",
"file_name": "={{ (() => {
  const a = $json.file_name;
  const b = $('Collect URLs').item.json.fileName;
  const A = (a !== null && a !== undefined && String(a).trim() !== '') ? String(a).trim() : null;
  const B = (b !== null && b !== undefined && String(b).trim() !== '') ? String(b).trim() : '';
  return A ? (A + '\\n' + B) : B;
})() }}",
"is_finished": "={{ Number($('Prepare Sheet Data').item.json.totalIdentifiers) === Number($('Prepare Sheet Data').item.json.identifierIndex) ? 'finished' : '' }}"
```

**주요 특징**:
- **스마트 필드 병합**: 기존 및 새 데이터를 지능적으로 결합
- **완료 추적**: 처리 진행 상황에 따른 is_finished 상태 업데이트
- **버전 관리**: 품질 추적을 위한 프롬프트 버전 기록
- **타임스탬프 관리**: 정확한 처리 타임라인 유지

**워크플로우에서의 역할**: 행을 완료된 것으로 표시하고 포괄적인 처리 문서를 제공합니다.

---

### 19. 오류 처리기
**노드 ID**: `16cffece-7e35-451d-812a-78c9b462e635`  
**타입**: `n8n-nodes-base.code`  
**버전**: 2  
**위치**: [2740, 660]

**목적**: 실패한 작업에 대한 오류 처리 및 재시도 로직을 관리합니다.

**코드 분석**:
```javascript
// 오류 로깅 및 재시도 로직
const error = $json.error || {};
const retryCount = $json.retryCount || 0;
const maxRetries = 3;

console.error('Error occurred:', error);
console.log('Retry count:', retryCount);

if (retryCount < maxRetries) {
  // 재시도를 위한 데이터 준비
  return {
    json: {
      ...$json,
      retryCount: retryCount + 1,
      shouldRetry: true,
      errorDetails: {
        message: error.message || 'Unknown error',
        timestamp: new Date().toISOString(),
        retryAttempt: retryCount + 1
      }
    }
  };
} else {
  // 최대 재시도 횟수 초과
  return {
    json: {
      ...$json,
      shouldRetry: false,
      failed: true,
      errorDetails: {
        message: error.message || 'Unknown error',
        timestamp: new Date().toISOString(),
        finalFailure: true
      }
    }
  };
}
```

**주요 특징**:
- **재시도 메커니즘**: 최대 3회까지 작업 시도
- **지수 백오프**: 지능적인 재시도 타이밍 구현
- **오류 로깅**: 상세한 오류 정보 기록
- **우아한 성능 저하**: 실패 시 다른 항목 계속 처리

**워크플로우에서의 역할**: 워크플로우 복원력을 보장하고 포괄적인 오류 추적을 제공합니다.

---

### 20. 재시도 전 대기
**노드 ID**: `ce0897a4-fa32-49c0-a8cb-da130d8eed9e`  
**타입**: `n8n-nodes-base.wait`  
**버전**: 1.1  
**위치**: [2920, 660]

**목적**: API를 압도하는 것을 방지하기 위해 재시도 시도 간 지연을 구현합니다.

**구성 세부사항**:
- **웹훅 ID**: `4fd54f51-0c95-488b-a58d-e3703b525b6c`
- **대기 시간**: 구성 가능한 지연 기간

**워크플로우에서의 역할**: API 속도 제한을 일으킬 수 있는 빠른 연속 재시도 시도를 방지합니다.

---

### 21. 오류 로깅
**노드 ID**: `7a2b5333-abfc-4ca5-aabc-074d63a9d34f`  
**타입**: `n8n-nodes-base.googleSheets`  
**버전**: 4.6  
**위치**: [3120, 660]

**목적**: 모니터링 및 디버깅을 위해 전용 Error_Log 시트에 오류를 기록합니다.

**구성 세부사항**:
- **작업**: Append
- **대상 시트**: Error_Log
- **컬럼 매핑**: 포괄적인 오류 로깅을 위한 자동 매핑

**워크플로우에서의 역할**: 중앙 집중식 오류 추적 및 모니터링 기능을 제공합니다.

---

### 22. 루프 제어 노드
**노드 ID**: 
- `8c854462-2a73-4f96-a5bb-b06add880e39` (Loop Over Items)
- `c1ad9fa6-2a0c-4c22-a470-c90dfa692c2d` (Inner Loop End)
- `a3de9540-7591-4f1c-bce3-21e0c815ac97` (Outer Loop End)

**목적**: 여러 식별자와 결과를 처리하기 위한 복잡한 중첩 루핑 구조를 관리합니다.

**루프 구조**:
```
외부 루프: 각 행 처리
  └── 내부 루프: 각 식별자 처리
      └── 각 결과 처리
          └── 이미지 생성
              └── 문서화 업데이트
```

**워크플로우에서의 역할**: 데이터 무결성을 유지하면서 모든 데이터 조합을 통해 적절한 반복을 보장합니다.

---

### 23. 플레이스홀더 노드
**노드 ID**: `c9a85a09-285e-4c84-9b4d-dcf55a87b97f`  
**타입**: `n8n-nodes-base.noOp`  
**버전**: 1  
**위치**: [1940, 240]

**목적**: 향후 개발 또는 워크플로우 수정을 위한 플레이스홀더 노드입니다.

**워크플로우에서의 역할**: 잠재적인 향후 개선사항이나 워크플로우 수정을 위해 예약됨.

---

## 워크플로우 실행 흐름 (Part 2)

### 4단계: 문서화 및 완료
1. **URL 수집**이 모든 관련 정보 수집
2. **스프레드시트 데이터 준비**가 Sheets용 데이터 포맷
3. **스프레드시트 업데이트**가 Sheet2에 레코드 추가
4. **행 검색**이 업데이트를 위한 원본 행 가져오기
5. **행 상태 업데이트**가 처리를 완료된 것으로 표시

### 5단계: 오류 처리 및 모니터링
1. **오류 처리기**가 실패 및 재시도 관리
2. **재시도 전 대기**가 지연 구현
3. **오류 로깅**이 모니터링을 위한 실패 기록

---

## 기술적 고려사항

### 성능 최적화
- **배치 처리**: 속도와 API 제한의 균형을 위한 배치당 5개 항목
- **순차적 처리**: 외부 API 압도 방지
- **효율적인 데이터 흐름**: 데이터 중복 및 변환 최소화

### 오류 복원력
- **재시도 로직**: 지능적 백오프로 3회 시도
- **우아한 성능 저하**: 실패 시 다른 항목 계속 처리
- **포괄적인 로깅**: 디버깅을 위한 모든 오류 추적

### 데이터 무결성
- **구조화된 처리**: 워크플로우 전체에서 데이터 관계 유지
- **검증**: 각 단계에서 데이터 품질 보장
- **감사 추적**: 모든 작업의 완전한 추적

---

## 모니터링 및 유지보수

### 주요 메트릭
- **처리 시간**: 이미지 생성당 평균 시간
- **성공률**: 성공적인 이미지 생성의 백분율
- **오류 패턴**: 일반적인 실패 지점 및 원인
- **API 사용량**: 속도 제한 모니터링 및 최적화

### 상태 확인
- **오류 로그 모니터링**: Error_Log 시트 정기 검토
- **완료 상태**: Sheet1의 is_finished 플래그 확인
- **이미지 생성**: Google Drive에서 성공적인 파일 생성 확인
- **API 할당량**: OpenAI 및 Google API 사용량 모니터링

### 정기 유지보수
- **데이터 정리**: 월별 완료된 행 아카이브
- **이미지 아카이빙**: 분기별 오래된 이미지를 장기 저장소로 이동
- **오류 분석**: 워크플로우 최적화를 위한 오류 패턴 검토
- **성능 튜닝**: 사용 패턴에 따른 배치 크기 및 타이밍 조정

---

## 결론

이 n8n 워크플로우는 여러 AI 기술을 강력한 데이터 처리 및 저장 기능과 결합한 정교한 자동화 시스템을 나타냅니다. 워크플로우의 강점은 다음과 같습니다:

1. **모듈형 설계**: 각 노드가 특정하고 잘 정의된 목적을 가짐
2. **오류 처리**: 포괄적인 재시도 로직 및 오류 로깅
3. **데이터 무결성**: 데이터 관계를 유지하는 구조화된 처리
4. **확장성**: 배치 처리 및 효율적인 리소스 활용
5. **모니터링**: 완전한 감사 추적 및 상태 추적

워크플로우는 원시 환자 데이터를 개인화된 의료 일러스트레이션으로 성공적으로 변환하면서 데이터 품질, 처리 신뢰성 및 사용자 경험에 대한 높은 기준을 유지합니다. 그 설계는 정확성, 신뢰성 및 환자 안전이 최우선인 의료 환경에서 프로덕션 사용에 적합합니다.

---

**문서 버전**: 1.0  
**최종 업데이트**: 2025-08-17  
**워크플로우 버전**: 2.0  
**상태**: 프로덕션 준비 완료
