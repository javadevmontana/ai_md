# HTML Entity 이중 인코딩 이슈 분석

## 문제 현상
- `Q&A`가 `Q&amp;A`로 잘못 표시됨
- HTML 소스에서 `Q&amp;amp;A`로 이중 인코딩되어 있음

## 원인 분석

### 발생 흐름
1. 서버에서 메뉴명이 이미 HTML 인코딩된 상태로 전달됨 (`Q&amp;A`)
2. jQuery `.text()` 메서드가 텍스트 설정 시 자동으로 HTML 인코딩 수행
3. 결과적으로 `&amp;` → `&amp;amp;` 이중 인코딩 발생

### 문제 파일 및 라인

| 파일 | 라인 | 코드 | 설명 |
|------|------|------|------|
| `common\js\jquery.mainTab.js` | 168 | `$tabLI.find("span").text(title);` | 탭 제목 업데이트 시 재인코딩 |
| `common\js\jquery.mainTab.js` | 173 | `$button.text(title);` | 버튼 텍스트 업데이트 시 재인코딩 |
| `common\js\jquery.mainTab.js` | 174 | `$tabLI.find("div").text(title);` | 툴팁 텍스트 업데이트 시 재인코딩 |

### 참고: 정상 동작하는 코드
아래 코드는 HTML에 직접 삽입하므로 브라우저가 `&amp;`를 `&`로 해석하여 정상 표시됨:

| 파일 | 라인 | 코드 |
|------|------|------|
| `common\js\jquery.mainTab.js` | 64 | `tabHTML += "<span ...>"+item.title+"</span>";` |
| `common\js\jquery.mainTab.js` | 74 | `tabHTML += "<div ...>" + item.title + "</div>";` |

## 해결 방법

### 방법 1: 클라이언트측 수정 (권장)

`common\js\jquery.mainTab.js` 파일에 HTML 디코드 함수 추가 후 적용:

```javascript
// 파일 상단에 디코드 함수 추가
function decodeHtmlEntity(str) {
    var txt = document.createElement("textarea");
    txt.innerHTML = str;
    return txt.value;
}
```

168, 173, 174 라인 수정:

```javascript
// 168번 라인
if(!isNullStr(title)) $tabLI.find("span").text(decodeHtmlEntity(title));

// 173번 라인
$button.text(decodeHtmlEntity(title));

// 174번 라인
$tabLI.find("div").text(decodeHtmlEntity(title));
```

### 방법 2: 서버측 수정

메뉴명을 클라이언트로 전달할 때 HTML 인코딩하지 않은 원본 텍스트로 전달.
(서버측 메뉴 데이터 처리 로직 확인 필요)

## 권장사항

- **방법 1 (클라이언트측 수정)** 권장
- 서버측 수정 시 다른 곳에서 해당 데이터를 HTML로 직접 출력하는 경우 XSS 취약점 발생 가능성 있음
- 클라이언트에서 디코드 처리하는 것이 안전함
