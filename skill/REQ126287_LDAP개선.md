# LDAP 조직도 동기화 데몬 서비스 개선 문서

## 📋 목차
1. [개요](#1-개요)
2. [LDAP 디렉토리 구조](#2-ldap-디렉토리-구조)
3. [서비스 흐름도](#3-서비스-흐름도)
4. [단계별 상세 설명](#4-단계별-상세-설명)
5. [기존 대비 변경 사항](#5-기존-대비-변경-사항)
6. [성능 비교](#6-성능-비교)
7. [DB 테이블 구조](#7-db-테이블-구조)
8. [데이터 흐름](#8-데이터-흐름)
9. [의존 라이브러리](#9-의존-라이브러리)
10. [소스 파일 목록](#10-소스-파일-목록)
11. [배포 절차 및 확인사항](#11-배포-절차-및-확인사항)
12. [참고사항](#12-참고사항)

---

## 1. 개요

LDAP 서버에서 전자정부 표준 조직도 데이터를 주기적으로 조회하여 DB에 동기화하는 배치 데몬 서비스입니다.
문서유통 시스템에서 수신처 선택 트리, 발신명의(authName) 표시 등에 사용됩니다.

---

## 2. LDAP 디렉토리 구조

### 2.1 DN(Distinguished Name) 계층 구조

```
c=kr                                          ← 국가
└── o=government of korea                     ← 최상위 조직
    └── ou=대학                               ← 분류
        └── ou=전북대학교                      ← 기관
            ├── ou=인문과학대학               ← 단과대학
            │   ├── ou=국어국문학과           ← 학과
            │   └── ou=영어영문학과
            └── ou=자연과학대학
                └── ou=수학과
```

### 2.2 LDAP 엔트리 속성 (objectClass=ucOrg2)

| 속성명 | 설명 | 예시 |
|--------|------|------|
| `ou` | 부서명 | 인문과학대학 |
| `ouCode` | 부서코드 (고유 식별자) | 7000941 |
| `ouOrder` | 정렬순서 | 10 |
| `ouLevel` | 계층 레벨 | 3 |
| `ucOrgFullName` | 조직 전체명 | 전북대학교 인문과학대학 |
| `parentouCode` | 상위 부서코드 | 7006699 |
| `ouReceiveDocumentYN` | 문서 수신 가능 여부 | Y/N |
| `ouSendOutDocumentYN` | 대외문서 발신 가능 여부 | Y/N |
| `ucChieftitle` | 기관장 직책명 | 학장 |
| `repouCode` | 대표부서코드 | 7000001 |
| `entryDn` | LDAP DN (조회 시 자동 포함) | ou=인문과학대학,ou=전북대학교,... |

### 2.3 검색 필터

```
(&(objectClass=ucOrg2)
  (!(ou=people))
  (!(objectClass=organizationalUnit))
  (!(ou=GPKI))
  (!(ou=Group Of EDI))
  (!(ou=Group Of Server)))
```

- `ucOrg2` objectClass를 가진 조직 엔트리만 대상
- `people`, `GPKI`, `Group Of EDI`, `Group Of Server` 등 비조직 OU 제외
- `organizationalUnit`(일반 OU 컨테이너) 제외

### 2.4 upSendChieftitle (상위기관 발신명의) 규칙

```
if (ouReceiveDocumentYN == "Y" && ouSendOutDocumentYN == "N") {
    // 수신 가능하지만 자체 대외발신 불가 → 상위 발신기관장 필요
    authName = upSendChieftitle + "(" + ucChieftitle + ")"
    // 예: "총장(학장)"
} else {
    authName = ucChieftitle
    // 예: "총장"
}
```

**탐색 로직:** 현재 조직의 `parentouCode`를 따라 상위로 올라가면서 `ouSendOutDocumentYN="Y"`인 기관을 찾고, 해당 기관의 `ucChieftitle`을 반환합니다.

---

## 3. 서비스 흐름도

### 3.1 전체 실행 흐름

```
시작 (데몬 스케줄)
    ↓
[1] AppLdapTreeMakeRunner.doRun()
    └─ ThreadPool에 Worker 등록
        ↓
[2] AppLdapTreeMakeWorker.doWork()
    ├─ 시작 시간 기록
    └─ doProcess() 호출
        ↓
[3] AppLdapTreeMakeWorker.doProcess()
    ├─ AppLdapMgr 싱글톤 인스턴스 획득
    ├─ LDAP 동기화 시작 로그
    └─ LDAP 데이터 동기화 시작
        ↓
[4] AppLdapMgr.getLdapRcvTreeAll()  ← 핵심 메소드
    ├─ 전역 카운터/캐시 초기화
    ├─ LdapPagedSearch 인스턴스 생성
    ├─ UnboundID LDAP SDK로 연결
    └─ 재귀적 트리 탐색 시작
        ↓
[5] AppLdapMgr.getLdapTreeListWithPaging()  ← 재귀 호출
    ├─ LDAP Base DN부터 시작
    ├─ LdapPagedSearch.searchWithPagingUsingConnection() 호출
    │  └─ RFC 2696 페이징으로 직계 하위만 조회 (ONE level)
    ├─ 글로벌 캐시에 저장 (deep copy)
    ├─ upSendChieftitle 계산 (캐시 기반 + DN 폴백)
    ├─ 조회 결과를 DB 저장용 모델로 변환 (setLdapModel)
    ├─ DB 슬레이브 테이블에 INSERT
    ├─ 각 하위 조직에 대해 자신을 재귀 호출
    └─ 모든 조직도 트리 탐색 완료
        ↓
[6] 실패 항목 재시도 (2차)
    ├─ failedEntryDnList 순회
    └─ 3초 간격으로 재시도
        ↓
[7] 동기화 건수 검증
    ├─ AppLdapMgr.selectSlaveCount() - DB에 저장된 건수 조회
    └─ 메모리 카운트 vs DB 건수 비교
        ↓
[8] 마스터/슬레이브 테이블 교체
    ├─ 건수 일치 + 0이 아님 확인
    ├─ AppLdapMgr.reverseLdapMasterSlave() - 테이블 교체
    └─ 동기화 완료 로그
        ↓
종료 (소요 시간 기록)
```

### 3.2 upSendChieftitle 캐시 조회 흐름

```
findUpSendChieftitleFromGlobalCache(parentouCode, ouSendOutDocYn, entryDn)
    │
    │ ouSendOutDocYn == "Y" → 자체 발신 가능 → return ""
    │
    ▼
┌─── 반복 (최대 20회) ──────────────────────────────────────────────┐
│                                                                    │
│  [1차 시도] globalOrgCache.get(parentouCode)                      │
│       │                                                            │
│       ├── 히트 → 상위 기관 정보 획득                               │
│       │         ouSendOutDocYN="Y" → return ucChieftitle          │
│       │         ouSendOutDocYN="N" → 더 상위로 반복               │
│       │                                                            │
│       └── 미스 → [2차 시도] DN 기반 폴백                          │
│                   │                                                │
│                   ▼                                                │
│            extractParentDn(currentEntryDn)                         │
│            "ou=A,ou=B,ou=C,c=kr" → "ou=B,ou=C,c=kr"             │
│                   │                                                │
│                   ▼                                                │
│            globalDnCache.get(parentDn)                             │
│                 ├── 히트 → 상위 기관으로 처리                      │
│                 └── 미스 → 더 상위 DN으로 반복                     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 4. 단계별 상세 설명

### [Step 1] AppLdapTreeMakeRunner.doRun()

### [Step 2] AppLdapTreeMakeWorker.doWork()

### [Step 3] AppLdapTreeMakeWorker.doProcess()

**역할:** LDAP 동기화 처리 로직 조율

**처리 순서:**

#### (1) AppLdapMgr 싱글톤 인스턴스 획득

#### (2) LDAP 동기화 시작
```java
int orgLdapCnt = appLdapMgr.getLdapRcvTreeAll();
```
- **반환값:** LDAP에서 조회한 조직 개수

#### (3) DB에 실제 저장된 건수 확인
```java
int realLdapCnt = appLdapMgr.selectSlaveCount();
```
- 정말로 DB에 저장되었는지 검증

#### (4) 건수 검증 후 테이블 교체
```java
if (orgLdapCnt == realLdapCnt && realLdapCnt != 0) {
    appLdapMgr.reverseLdapMasterSlave();
}
```

### [Step 4] AppLdapMgr.getLdapRcvTreeAll()

**초기화 단계:**
```java
// 1. 카운터 초기화
setLdapCnt(0);

// 2. 실패 목록 초기화
failedEntryDnList.clear();

// 3. 글로벌 캐시 초기화
globalOrgCache.clear();
globalDnCache.clear();

// 4. LDAP 연결 준비
com.unboundid.ldap.sdk.LDAPConnection unboundIdConn = null;

// 5. Base DN 조회
String entryDn = ConfigServiceManager.getInstance()
    .getValue(Constants.XCLICK_SYSTEM, "//app/gcc/ldap-base-dn");
```

**1차 동기화 (재귀 탐색):**
```java
getLdapTreeListWithPaging(mapList, unboundIdConn, entryDn);
```

**2차 처리 (실패 항목 재시도):**
```java
if (failedEntryDnList.size() > 0) {
    for (String failedDn : retryList) {
        Thread.sleep(3000);  // 3초 대기
        getLdapTreeListWithPaging(..., failedDn);
    }
}
```

### [Step 5] AppLdapMgr.getLdapTreeListWithPaging()

**역할:** 페이징 방식으로 LDAP 트리 재귀 탐색

**페이징이 필요한 이유:**
- LDAP 서버는 보통 한 번에 조회할 수 있는 결과 개수 제한 (sizeLimit)
- 조직이 1000개 이상이면 제한 초과로 실패
- 페이징: 500개씩 여러 번 조회


**LdapPagedSearch에서 수행되는 작업:**
```
▼ 페이지 1 (500개)
  ├─ RFC 2696 페이징 쿠키 생성
  ├─ SearchScope.ONE (직계 하위만 조회)
  ├─ LDAP 서버에서 최대 500개 조회
  └─ 쿠키 반환 (다음 페이지 있음?)

▼ 페이지 2 (500개)
  ├─ 이전 쿠키 사용
  └─ 다음 500개 조회

▼ 페이지 3 (250개)
  ├─ 마지막 페이지
  └─ 쿠키 없음 (종료)
```

#### 글로벌 캐시에 저장 (deep copy)
```java
for(Map<String, String> org : ldapList) {
    String ouCode = org.get("ouCode");
    String orgEntryDn = org.get("entryDn");
    if(ouCode != null && !ouCode.trim().isEmpty()) {
        Map<String, String> cacheCopy = new HashMap<String, String>(org);
        globalOrgCache.put(ouCode, cacheCopy);
        if(orgEntryDn != null && !orgEntryDn.trim().isEmpty()) {
            globalDnCache.put(orgEntryDn, cacheCopy);
        }
    }
}
```

#### LDAP 원시 데이터를 DB 저장용으로 변환
```java
List<Map> mapList = setLdapModel(ldapList);
```

#### 각 하위 조직에 대해 재귀 호출
```java
for (Map map : mapList) {
    String srchEntryDn = (String)map.get("entryDn");
    getLdapTreeListWithPaging(mapAllList, unboundIdConn, srchEntryDn);
}
```

**재귀 예시:**
```
[1단계] o=company,c=kr 조회
        ├─ ou=dept1,o=company,c=kr
        ├─ ou=dept2,o=company,c=kr
        └─ ou=dept3,o=company,c=kr

[2단계] ou=dept1,o=company,c=kr 조회 (재귀 호출)
        ├─ ou=team1,ou=dept1,o=company,c=kr
        └─ ou=team2,ou=dept1,o=company,c=kr

[3단계] ou=team1,ou=dept1,o=company,c=kr 조회 (재귀 호출)
        ├─ ou=member1,...
        └─ ou=member2,...
```

---

## 5. 기존 대비 변경 사항

### 5.1 LDAP 라이브러리 교체

| 항목 | 기존 (Old) | 변경 (New) |
|------|------------|------------|
| 라이브러리 | Netscape LDAP SDK | UnboundID LDAP SDK 7.0.4 |
| 페이징 지원 | 미지원 (`setMaxResults(0)`) | RFC 2696 Simple Paged Results Control |
| sizeLimit 문제 | 서버 제한 초과 시 데이터 누락 가능 | 페이지 단위 조회로 제한 없음 |
| 연결 관리 | `LDAPConnection.disconnect()` | `LDAPConnection.close()` |
| 타임아웃 설정 | 미설정 | 연결 30초, 응답 120초 |

### 5.2 upSendChieftitle 계산 방식 변경

| 항목 | 기존 (Old) | 변경 (New) |
|------|------------|------------|
| 조회 방식 | 매 기관마다 LDAP SUBTREE 검색 | 메모리 캐시 (HashMap) O(1) 조회 |
| 캐시 범위 | 없음 (매번 LDAP 질의) | 전체 동기화 동안 누적되는 글로벌 캐시 |
| 폴백 전략 | 없음 | DN 기반 폴백 (parentouCode 미스 시) |
| 변수 누수 버그 | `upSendChieftitle` 루프 외부 선언, 리셋 안됨 | 매 기관별 독립 계산 |
| N+1 문제 | 약 9만건 × 평균 3회 = ~27만회 LDAP 쿼리 | 0회 추가 LDAP 쿼리 |

**기존 코드의 버그 (GccLdapList.getSrchOrgList):**
```java
// 루프 바깥에 선언 → 이전 기관의 값이 다음 기관으로 누수
String upSendChieftitle = "";  // line 337

while(e.hasMoreElements()) {   // line 340
    // upSendChieftitle이 리셋되지 않음
    mapData.put("upSendChieftitle", upSendChieftitle);
}
```

**수정된 코드 (AppLdapMgr.findUpSendChieftitleFromGlobalCache):**
- 매 기관별 독립적으로 계산
- `ouSendOutDocYN="Y"`인 기관은 빈 문자열 반환 (정상 동작)

### 5.3 동시성 및 안정성 개선

| 항목 | 기존 | 변경 |
|------|------|------|
| 카운터 | `static int ldapCnt` | `AtomicInteger ldapCnt` |
| 실패 관리 | 없음 (실패 시 전체 중단) | `CopyOnWriteArrayList<String> failedEntryDnList` |
| 재시도 | 연결 끊김 시 3회 시도 | 전체 완료 후 실패 항목 개별 재시도 (3초 간격) |
| 연결 재시도 | `Thread.sleep(1000*(3*tryCount))` | LdapPagedSearch 내부에서 새 연결 자동 생성 |
| 캐시 오염 방지 | 없음 (Map 참조 공유) | `new HashMap<>(org)` deep copy |

### 5.4 로깅 개선

- 동기화 시작/종료 로깅 (소요시간 포함)
- 각 단계별 건수 추적 (조회 건수, 변환 건수, 누적 건수)
- DB INSERT 후 즉시 건수 검증 (메모리 카운트 vs DB 카운트)
- 실패 항목 상세 로깅 및 재시도 결과 로깅

---

## 6. 성능 비교

### 6.1 속도

| 항목 | 기존 | 변경 | 개선율 |
|------|------|------|--------|
| LDAP 쿼리 횟수 | ~27만회 (N+1 문제) | ~9만회 (DFS 재귀만) | **약 66% 감소** |
| upSendChieftitle 계산 | LDAP SUBTREE 검색 (네트워크 I/O) | HashMap O(1) 조회 (메모리) | **네트워크→메모리** |
| 페이징 | 미지원 (서버 제한에 의존) | 500건 단위 페이징 | 안정적 대용량 처리 |
| 검색 범위 | SUBTREE (전체 하위 트리) | ONE level (직계 하위만) | 재귀+페이징으로 안정적 |

**속도 개선 핵심:**
- 기존: 각 기관마다 상위 기관을 LDAP 서버에 직접 질의 (평균 3 depth × 9만건 = ~27만회 네트워크 I/O)
- 변경: DFS 탐색하면서 글로벌 캐시 구축 → O(1) 메모리 조회로 계산 완료
- 네트워크 레이턴시 제거로 전체 동기화 시간 대폭 감소

### 6.2 메모리 사용량

| 항목 | 크기 (약 9만 기관 기준) |
|------|------------------------|
| `globalOrgCache` (ouCode→Map) | ~12MB |
| `globalDnCache` (entryDn→Map) | ~15MB |
| 동시 보유 최대 메모리 | ~27MB (동기화 완료 후 GC 대상) |

**메모리 사용 상세:**
- 1개 기관 Map: 약 10개 필드 × 평균 30byte = ~300byte
- deep copy로 캐시 저장: 기관당 ~300byte × 9만건 = ~27MB
- `globalDnCache`는 `globalOrgCache`와 동일 객체 참조 (추가 메모리 ~3MB, 키 문자열만)
- 동기화 종료 후 `clear()` 호출로 GC 대상

### 6.3 안정성

| 시나리오 | 기존 | 변경 |
|----------|------|------|
| sizeLimit 1000 초과 | 데이터 누락 가능 | 페이징으로 전량 조회 |
| 중간 연결 끊김 | 전체 실패 | 해당 DN만 failedEntryDnList에 추가, 나머지 계속 |
| parentouCode 미존재 기관 | LDAP 검색 실패 → 빈 값 | DN 기반 폴백으로 상위 기관 탐색 |
| 동시성 문제 | `static int` 경합 가능 | `AtomicInteger` 사용 |

---

## 7. DB 테이블 구조

### 7.1 T_APP_GCCLDAPTREE / XGCCT_APP_GCCLDAPTREE2 (동기화 대상 테이블)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| LDAPID | VARCHAR | PK, UUID |
| OU | VARCHAR | 부서명 |
| OUCODE | VARCHAR | 부서코드 |
| OUORDER | NUMBER | 정렬순서 |
| OULEVEL | NUMBER | 계층레벨 |
| UCORGFULLNAME | VARCHAR | 조직 전체명 |
| PARENTOUCODE | VARCHAR | 상위 부서코드 |
| OURECEIVEDOCUMENTYN | CHAR(1) | 수신 가능 여부 |
| OUSENDOUTDOCUMENTYN | CHAR(1) | 대외발신 가능 여부 |
| UCCHIEFTITLE | VARCHAR | 기관장 직책 |
| UPSENDCHIEFTITLE | VARCHAR | 상위 발신기관장 직책 |
| AUTHNAME | VARCHAR | 발신명의 |
| CHILDCNT | NUMBER | 하위 기관 존재 여부 (0/1) |
| ENTRYDN | VARCHAR | LDAP DN |
| REPOUCODE | VARCHAR | 대표부서코드 |
| REGDATE | VARCHAR(8) | 등록일자 (yyyyMMdd) |

---

## 8. 데이터 흐름

### 8.1 입력 데이터
```
LDAP 서버
├─ o=company,c=kr (Base DN)
├─ ou=dept1 (1차)
│  ├─ ou=team1 (2차)
│  │  ├─ ou=member1 (3차)
│  │  └─ ou=member2 (3차)
│  └─ ou=team2 (2차)
└─ ou=dept2 (1차)
   └─ ou=team3 (2차)
```

### 8.2 처리 과정
```
[LDAP 서버]
    ↓ (LdapPagedSearch로 페이징 조회)
[원시 데이터 (Map)]
    ou: "dept1"
    ouCode: "D001"
    ...
    ↓ (글로벌 캐시 저장 + upSendChieftitle 계산)
    ↓ (setLdapModel로 변환)
[DB 저장용 모델]
    ou: "dept1"
    ouCode: "D001"
    authName: "대표이사(부장)" ← 추가됨
    ldapId: "APP_xxx..." ← 생성됨
    regdate: "20260122" ← 생성됨
    ...
    ↓ (INSERT)
[DB 슬레이브 테이블]
    XGCC_LDAP_TREE2 ← 여기에 저장
```

### 8.3 출력 데이터
```
[동기화 결과]
✓ 메모리 건수: 63527개
✓ DB 건수: 63527개
✓ 건수 일치 확인
✓ 테이블 교체 완료

[로그]
[결과] 총 건수: 63527, 소요시간: 120000ms (0시간 2분 0초)
```

---

## 9. 의존 라이브러리

| 라이브러리 | 버전 | 용도 |
|-----------|------|------|
| UnboundID LDAP SDK | 7.0.4 | LDAP 서버 연결 및 RFC 2696 페이징 검색 |
| ~~Netscape LDAP SDK~~ | (제거됨) | ~~기존 LDAP 연결 라이브러리~~ |

---

## 10. 소스 파일 목록

### 10.1 주요 수정 파일

#### ① AppLdapMgr.java (핵심 수정)
**경로:** `\src\bizwell\xclick\gw\app\service\AppLdapMgr.java`

**수정 내용:**
- UnboundID LDAP SDK로 라이브러리 교체
- 글로벌 캐시 추가 (`globalOrgCache`, `globalDnCache`)
- upSendChieftitle 계산 로직 개선 (DN 폴백 방식)
- RFC 2696 페이징 지원
- 동시성 개선 (`AtomicInteger`, `CopyOnWriteArrayList`)
- 실패 항목 자동 재시도 로직
- 상세 로깅 추가

**주요 변경 메소드:**
- `getLdapRcvTreeAll()` - UnboundID SDK 사용, 실패 재시도 추가
- `getLdapTreeListWithPaging()` - 신규 페이징 메소드 (기존 `getLdapTreeList` 대체)
- `findUpSendChieftitleFromGlobalCache()` - 신규 캐시 기반 조회 메소드
- `extractParentDn()` - 신규 DN 파싱 메소드

#### ② LdapPagedSearch.java (신규 생성)
**경로:** `\src\bizwell\xclick\gw\app\gcc\LdapPagedSearch.java`

**역할:**
- UnboundID LDAP SDK 래퍼 클래스
- RFC 2696 Simple Paged Results Control 구현
- LDAP 페이징 검색 전담
- 연결 관리 및 타임아웃 설정

### 10.2 기존 파일 (더 이상 사용 안함)

#### GccLdapList.java
**경로:** `\src\bizwell\xclick\gw\app\gcc\GccLdapList.java`
- Netscape LDAP SDK 사용 (구버전)
- `AppLdapMgr.java`에서 더 이상 참조하지 않음
- 변수 누수 버그 있었음 (`upSendChieftitle` 루프 외부 선언)

#### GccLdapSearch.java
**경로:** `\src\bizwell\xclick\gw\app\gcc\GccLdapSearch.java`
- Netscape LDAP SDK 사용 (구버전)
- SUBTREE 검색 방식
- 더 이상 사용 안함

### 10.3 의존 라이브러리 변경

#### 추가된 라이브러리
```
unboundid-ldapsdk-7.0.4.jar
```

#### 제거 가능한 라이브러리 (선택)
```
ldapjdk.jar (Netscape LDAP SDK)
```
**주의:** 다른 모듈에서 사용 중일 수 있으므로 전체 검색 후 제거 권장

### 10.4 전달 파일 체크리스트

```
✅ 소스 파일 (3개)
   ├─ AppLdapMgr.java (수정)
   └─ LdapPagedSearch.java (신규)

✅ 라이브러리
   └─ unboundid-ldapsdk-7.0.4.jar

✅ 문서
   └─ LDAP_개선_060123.md (현재 문서)
```

---

## 11. 배포 절차 및 확인사항

### 11.1 배포 전 확인사항

✅ **unboundid-ldapsdk-7.0.4.jar 요구사항**
- JDK 1.8 이상 지원 (JDK 25 버전까지 호환)

**다운로드:**
- 📋 GitHub: https://github.com/pingidentity/ldapsdk/releases/download/7.0.4/unboundid-ldapsdk-7.0.4.zip
- 📋 Maven: https://repo1.maven.org/maven2/com/unboundid/unboundid-ldapsdk/7.0.4/unboundid-ldapsdk-7.0.4.jar
- 📋 Gradle: `implementation 'com.unboundid:unboundid-ldapsdk:7.0.4'`

**Java 버전별 지원 여부:**

| Java 버전 | UnboundID 7.0.4 지원 |
|-----------|---------------------|
| JDK 1.6 (Java 6) | ❌ 불가 |
| JDK 1.7 (Java 7) | ❌ 불가 (UnboundID 6.0.11 사용 가능) |
| JDK 1.8 (Java 8) | ✅ 가능 |
| JDK 11 | ✅ 가능 |
| JDK 17 | ✅ 가능 |
| JDK 21 | ✅ 가능 |
| JDK 25 | ✅ 가능 (예상) |

✅ **UnboundID 6.0.11 다운로드 (JDK 1.7용)**
- https://github.com/pingidentity/ldapsdk/releases/tag/6.0.11

**기존 데이터 백업:**
```sql
-- 동기화 건수 확인 [2026.01.23 기준: 97,139건]
-- 기존 사용하는 테이블 백업 (추후 삭제 필요)
CREATE TABLE T_APP_GCCLDAPTREE_BAK AS
SELECT * FROM T_APP_GCCLDAPTREE;

SELECT COUNT(*) FROM T_APP_GCCLDAPTREE_BAK;
```

### 11.2 배포 후 확인사항

```sql
-- 동기화 건수 확인
SELECT COUNT(*) FROM T_APP_GCCLDAPTREE;

-- upSendChieftitle 데이터 확인
SELECT COUNT(*) FROM T_APP_GCCLDAPTREE WHERE UPSENDCHIEFTITLE IS NOT NULL;

-- 특정 기관 확인 (예: ouCode=7000941)
SELECT OUCODE, OU, UCCHIEFTITLE, UPSENDCHIEFTITLE, AUTHNAME
FROM T_APP_GCCLDAPTREE
WHERE OUCODE = '7000941';
```

**로그 확인:**
```
[정상 로그]
========== LDAP 동기화 시작 (UnboundID SDK 페이징 사용) ==========
[시작] LDAP Base DN: o=government of korea,c=kr
[라이브러리] UnboundID LDAP SDK 7.0.4 (RFC 2696 페이징 지원)
...
[결과] 총 건수: 63527, 소요시간: 120000ms (0시간 2분 0초)
[최종실패] 동기화 실패 항목 수: 0
```

---

## 12. 참고사항

| 항목 | 값 |
|------|-----|
| 페이지 크기 | 500개 |
| 페이징 재시도 대기 | 3초 |
| LDAP 연결 타임아웃 | 30초 |
| LDAP 응답 타임아웃 | 120초 |
| 검색 범위 | ONE level (직계 하위만) |

