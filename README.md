# 어그로 제목 계산기

키워드 하나를 입력하면 국내 커뮤니티 제목의 품사 구조 패턴과 단어별 주목도 가중치를 조합해
클릭률 높은 제목 후보를 생성하고 0~100점으로 점수화하는 정적 단일 페이지 웹 서비스입니다.

- 서버 없음 (정적 파일만으로 동작)
- 외부 API 호출 / 크롤링 없음
- 모든 생성 로직은 브라우저(클라이언트)에서 실행
- 외부 라이브러리 0개 (순수 HTML/CSS/JS)

## 파일 구성

| 파일 | 설명 |
|---|---|
| `index.html` | CSS/JS가 인라인 포함된 단일 페이지 앱 — **배포되는 건 사실상 이 파일 + patterns.json + ads.json + robots.txt + sitemap.xml 뿐** |
| `patterns.json` | 제목 구조/단어 가중치/점수 규칙/슬롯 호환성 규칙 데이터 (코딩 없이 수정 가능) |
| `ads.json` | 상단/사이드바/하단 쿠팡파트너스 배너 데이터 (코딩 없이 수정 가능) |
| `robots.txt` | 검색엔진 크롤링 허용 규칙 |
| `sitemap.xml` | 사이트맵 |
| `package.json` | `npm run verify`(CSP 해시 검증) / `npm run quality`(품질 검증) 스크립트 정의. 의존성 없음 |
| `tools/verify-csp.js` | 배포 직전 필수 점검. index.html의 인라인 스크립트 해시가 CSP meta와 일치하는지 확인 |
| `tools/validate-quality.js` | `generateTitles()`를 Node에서 직접 호출해 조사 오류율·점수 분포·중복률을 정량 측정하는 검증 스크립트 |
| `reports/` | 검증 스크립트 실행 결과(CSV/JSON/로그) |

> `tools/`와 `reports/`는 로컬 개발/QA용이라 `.gitignore`에 올려 GitHub 저장소에는 포함하지
> 않았습니다. 배포되는 정적 사이트와는 무관하고, 로컬 사본에는 그대로 남아 있어 위 명령들은
> 계속 실행할 수 있습니다. 저장소에 함께 버전관리하고 싶다면 `.gitignore`에서 해당 줄을
> 지우고 `git add -f tools reports`로 강제 추가하면 됩니다.

## 로컬에서 실행하기

`index.html`은 `fetch()`로 `patterns.json`을 불러오기 때문에, `file://`로 직접 열면
브라우저 보안 정책(CORS) 때문에 데이터 로드가 실패할 수 있습니다(그 경우 내장 폴백 데이터로
동작하며 화면 하단에 안내 문구가 표시됩니다). 정상적으로 확인하려면 로컬 웹 서버로 열어주세요.

```bash
python -m http.server 8000
```

이후 브라우저에서 `http://localhost:8000/` 접속.

Node.js가 설치되어 있다면 대신 다음 명령도 사용할 수 있습니다.

```bash
npx serve .
```

## patterns.json 수정해서 패턴 추가하기 (코딩 불필요)

`patterns.json` 하나만 편집하면 코드를 건드리지 않고 제목 구조와 단어 사전을 확장할 수 있습니다.
슬롯 이름은 동적으로 해석되므로, `slots`에 새 이름을 추가하고 `structures[].slots`에서
그 이름을 참조하기만 하면 바로 동작합니다. (단, `TOPIC`, `TOPIC_EUN`, `TOPIC_I`, `TOPIC_REUL`은
사용자가 입력한 키워드 자리를 뜻하는 예약어이므로 `slots`에 직접 정의하지 않습니다.)

### 1) 새 단어 추가하기

예: `EVAL` 슬롯에 "레벨업"이라는 단어를 추가하고 싶다면:

```json
"EVAL": [
  { "w": "근황", "weight": 443889 },
  { "w": "레벨업", "weight": 52000 }
]
```

`weight`가 클수록 해당 단어가 더 자주 뽑히고, 점수 계산에도 더 크게 기여합니다.
(다른 단어들과 비교해서 상대적인 크기로 정하면 됩니다.)

### 2) 새 슬롯(품사) 추가하기

예: 지역명을 넣는 `PLACE` 슬롯을 새로 만들고 싶다면, `slots`에 추가:

```json
"PLACE": [
  { "w": "강남", "weight": 25000 },
  { "w": "부산", "weight": 18000 }
]
```

그리고 `structures`에서 이 슬롯을 참조하는 구조를 하나 추가합니다:

```json
{ "id": "s19", "slots": ["PLACE", "TOPIC", "EVAL", "PUNCT", "EXT"], "weight": 1500, "style": "both" }
```

이렇게 저장 후 새로고침하면 코드 수정 없이 바로 "강남 [키워드] 근황...jpg" 형태의 제목이 생성됩니다.

### 3) 구조(structures) 필드 설명

| 필드 | 설명 |
|---|---|
| `id` | 고유 식별자 (문자열) |
| `slots` | 순서대로 나열되는 슬롯 이름 배열. `TOPIC` 계열은 정확히 1번만 포함되어야 합니다 |
| `weight` | 이 구조가 뽑힐 가중치(클수록 자주 등장, 점수에도 반영) |
| `style` | `"concept"`(개념글) / `"normal"`(일반글) / `"both"`(둘 다) |
| `example` | (선택) 사람이 보기 위한 예시 문자열, 로직에는 사용되지 않음 |

`PUNCT`, `EXT`, `END`, `EMOTE` 슬롯은 바로 앞 단어에 공백 없이 붙습니다(예: `근황` + `...` + `jpg` → `근황...jpg`).
그 외 슬롯은 공백으로 구분됩니다.

> `PUNCT` 슬롯은 8개(`...`/`..`/`.`/`…`/`!`/`?!`/`..?`/`~`)입니다. 겹마침표(‥)나 가운뎃점
> 3연속(···) 같은 실제로 안 쓰이는 표기는 뺐고, 대신 커뮤니티에서 실제로 보이는 변형만
> 채웠습니다. 새 변형을 추가할 때도 개수를 채우는 것보다 실제로 쓰이는 표기인지를 우선하세요.
> 참고로 `?!`/`..?`처럼 `?`를 포함한 항목은 `EXT`와 결합되면(`...jpg` 자리) `has_question`
> 보너스도 함께 붙습니다 — 의도한 부수 효과입니다(말줄임표+확장자 조합인데 물음표가 섞이면
> 더 자극적인 제목으로 보는 것이 자연스럽다고 판단했습니다).

> 새 슬롯이 `PUNCT`/`EXT`/`END`/`EMOTE`처럼 "바로 앞 단어에 공백 없이 붙어야" 한다면(예:
> `NUMSUFFIX`: `3` + `년만에` → `3년만에`), `index.html`의 `ATTACH_SLOTS` 상수에 슬롯 이름을
> 추가해야 합니다 — 이건 patterns.json만으로는 안 되는 유일한 예외이자 필요한 코드 수정
> 지점입니다. 공백으로 구분되는 슬롯(대부분의 경우)은 이 작업이 필요 없습니다.

### 4) 점수/가산점 조정하기

`scoring` 객체의 `base`, `bonus.*`, `penalty.over_length`, `max` 값을 수정하면
바로 점수 계산 방식이 바뀝니다. 코드 수정이 필요 없습니다.

### 5) 검증

JSON 문법 오류가 있으면 자동으로 감지되어(스키마 최소 검증 포함) 내장 폴백 데이터로 전환되고
화면 하단에 안내 문구가 표시됩니다. 저장 후 반드시 브라우저에서 새로고침해 정상 동작을 확인하세요.
(JSON 유효성은 온라인 JSON validator나 `python -m json.tool patterns.json` 명령으로도 확인할 수 있습니다.)

패턴을 추가/수정한 뒤에는 `node tools/validate-quality.js`를 실행해 조사 오류율·점수 분포·
중복률에 회귀가 없는지 확인하는 것을 권장합니다.

### 6) 슬롯 호환성 규칙 (의미 충돌 방지)

`patterns.json`의 `keywordHeuristics`와 `compatibilityRules`로, 특정 키워드에는 특정 슬롯을
아예 쓰지 않도록 코드 수정 없이 제한할 수 있습니다. 가중치 조정으로는 해결되지 않는 문제(예:
"이강인 대학생...", "고딩 손흥민..."처럼 인명 키워드에 무관한 GROUP 단어가 붙는 경우)를 막기
위한 장치입니다.

```json
"keywordHeuristics": {
  "personName": { "pattern": "^[가-힣]{2,4}$" }
},
"compatibilityRules": [
  { "id": "personname-no-group", "when": { "keywordMatches": "personName" }, "excludeSlots": ["GROUP"] }
]
```

- `keywordHeuristics.<이름>.pattern`은 키워드 전체에 대한 정규식(전체 일치)입니다. 매치되면 그
  이름의 휴리스틱이 "적용됨"으로 판정됩니다.
- `compatibilityRules[].when.keywordMatches`가 위 휴리스틱 이름을 가리키면, 매치되는 키워드에
  한해 `excludeSlots`에 나열된 슬롯을 쓰는 구조가 후보에서 전부 제외됩니다.
- `when`을 아예 생략하면 키워드와 무관하게 항상 적용되는 규칙이 됩니다(특정 슬롯을 완전히
  끄고 싶을 때).
- 여러 규칙이 동시에 매치되면 배제 목록은 합집합으로 누적됩니다.
- 규칙 적용 후 후보 구조가 하나도 안 남으면 안전장치로 규칙을 무시하고 원래 후보군을 그대로
  씁니다 — 화면에 "결과 없음"이 뜨는 상황을 방지하기 위함입니다.
- `personName` 휴리스틱은 완벽한 인명 판별기가 아닙니다. "2~4자 한글이 공백 없이 단독으로
  등장"이라는 대략적인 규칙이라, "국밥"·"월요일"처럼 실제로는 인명이 아닌 키워드도 함께
  걸러집니다. 애매할 때는 보수적으로(GROUP을 못 쓰게) 판정하도록 일부러 그렇게 설계했습니다.

## 결과 배치 크기 / 더 보기

한 번에 보여주는 결과 개수는 `index.html`의 `RESULT_BATCH_SIZE`(현재 8)로 정해집니다.
"더 보기" 버튼은 같은 키워드로 `generateTitles()`를 다시 호출하되, 이미 화면에 보인 모든
`structureId`(`avoidStructureIds`)와 제목(`excludeTitles`)을 넘겨 회피/제외시킨 뒤 결과에
이어 붙입니다. "다시 뽑기"는 반대로 화면을 통째로 새 배치로 교체합니다(역시 직전에 보인
구조는 회피). 두 옵션 모두 `generateTitles(keyword, options, patterns)`의 `options`에 얹는
평범한 인자라서, 함수 자체는 여전히 DOM에 접근하지 않는 순수 함수입니다.

## 구조 선택 알고리즘 튜닝 (코드, patterns.json 범위 밖)

`index.html`의 `pickStructure()` 위에 있는 두 상수로 "다시 뽑기"의 체감 다양성을 조절합니다.

```js
var STRUCTURE_DECAY_BASE = 15;    // 한 배치 안에서 이미 쓴 구조는 weight / 15^(쓴 횟수)로 감쇠
var CROSS_RUN_AVOID_FACTOR = 0.08; // 직전 배치에서 쓴 구조는 weight에 0.08을 곱해 회피
```

이 두 값은 서로 반대 방향으로 당기는 목표를 조절합니다: `STRUCTURE_DECAY_BASE`를 올리면 한
배치 안의 구조 중복은 줄지만, 그만큼 한 배치가 구조 풀을 더 많이 소진해버려서 바로 다음
배치가 재사용할 수밖에 없는 구조가 늘어나 재실행 간 다양성(Jaccard)은 오히려 나빠집니다.

**수학적 배경 (2026-08-24 해소됨):** 구조 풀 크기가 N일 때, 한 배치가 서로 다른 구조 k개를
쓰면 연속된 두 배치의 구조 집합 Jaccard 유사도는 `(2k-N)/N`보다 작아질 수 없습니다(포함-배제
원리). 배치 크기가 12·풀이 13개(개념글/일반글)이던 시절에는 중복률 15% 이하(k≥11)와
Jaccard 0.35 이하를 동시에 만족하는 k가 존재하지 않았습니다. 지금은 **개념글·일반글 풀을
각각 25개로 늘리고 배치 크기를 8로 줄여**(`k`가 최대 8, N=25) 이 부등식 자체가 여유 있게
풀립니다 — 실측 재실행 Jaccard 0.16대, 배치 내 중복률 2~3%대로 두 목표 모두 달성했습니다.
`structures`에 새 스타일 구조를 추가하면 N이 더 커지므로 여유는 계속 늘어나지만, 반대로
`RESULT_BATCH_SIZE`(k)를 다시 키우면 이 한계가 재발할 수 있다는 점을 기억하세요. 자세한
계산은 품질 감사 리포트의 "00 · NEW"(1차) / 이어지는 업데이트(2차) 섹션을 참고하세요.

## 광고 코드 삽입 위치

`index.html` 안에 아래 3곳이 주석으로 표시되어 있습니다.

1. 상단 광고 (728x90 / 모바일 320x100) — `<div class="ad-slot ad-top" id="adTop">` 내부
2. 우측 사이드바 광고 (300x250) — `<div class="ad-slot ad-side" id="adSide">` 내부 (768px 미만에서는 본문 하단으로 자동 이동)
3. 하단 광고 (728x90) — `<div class="ad-slot ad-bottom" id="adBottom">` 내부

세 자리 모두 현재 **쿠팡파트너스 이미지 배너**가 `ads.json`을 통해 자동으로 채워지고 있습니다
(아래 "쿠팡 배너 수정하는 법" 참고). 애드센스 등 스크립트 기반 광고로 바꾸려면 이 `div`들의
내용을 직접 교체하면 됩니다.

각 `div`는 레이아웃 흔들림(CLS) 방지를 위해 `height`(+`min-height`)가 고정돼 있으니, 실제
광고 태그를 넣을 때도 이 크기를 벗어나지 않도록 유지하는 것을 권장합니다. (이미지 배너처럼
`height:100%` 자식 요소를 쓰는 경우, 부모에 `min-height`만 있고 고정 `height`가 없으면 이미지의
원본 크기만큼 컨테이너가 커져버리는 문제가 있어 `height`를 함께 고정해뒀습니다.)

### 쿠팡 배너 수정하는 법 (코딩 불필요)

세 자리의 상품/링크/이미지는 전부 저장소 루트의 `ads.json` 하나로 관리됩니다. 이 파일만
고치면 코드를 건드리지 않고 배너를 교체할 수 있습니다.

```json
{
  "top":     { "productName": "코비츠 스마트폰 저주파마사지기", "url": "https://link.coupang.com/a/gss5N40Lro", "imageUrl": "https://gi.esmplus.com/suuuuuuu/marketing/bt-701.png", "enabled": true },
  "sidebar": { "productName": "꿀숨밴드", "url": "https://link.coupang.com/a/gsteYgsu8y", "imageUrl": "https://gi.esmplus.com/suuuuuuu/marketing/ggulsum%20300x250.png", "enabled": true },
  "bottom":  { "productName": "무지외반증교정기", "url": "https://link.coupang.com/a/gsvetfIZ9o", "imageUrl": "https://gi.esmplus.com/suuuuuuu/marketing/mooji.png", "enabled": true }
}
```

- `top`/`sidebar`/`bottom` 키는 위 세 광고 슬롯에 그대로 대응합니다.
- `productName`: 이미지 로드 실패 시 대신 표시되는 텍스트이자 `alt`/`aria-label`에 쓰이는 상품명.
- `url`: 클릭 시 이동할 쿠팡파트너스 딥링크. **`link.coupang.com` 또는 `coupang.com` 도메인만
  허용**됩니다 — 다른 도메인이면 검증에서 걸러져 배너 자체가 렌더링되지 않고 기존
  placeholder("광고 영역 ...")가 그대로 남습니다(브라우저 콘솔에 경고 로그).
- `imageUrl`: 배너 이미지 주소. **`gi.esmplus.com` 도메인만 허용**됩니다. 다른 이미지 호스트를
  쓰려면 `index.html`의 `AD_ALLOWED_IMAGE_HOSTS` 배열에 도메인을 추가하고, CSP `img-src`에도
  같은 도메인을 추가해야 합니다(둘 다 안 하면 검증 실패 또는 CSP 차단으로 이미지가 안 뜹니다).
- `enabled`: `false`로 두면 해당 슬롯은 값이 있어도 렌더링하지 않고 placeholder를 유지합니다
  (배너를 코드 삭제 없이 잠깐 끄고 싶을 때 사용).
- 이미지가 실제로 로드에 실패하면(깨진 링크 등) 자동으로 `productName` 텍스트 배너로
  대체되고, 링크와 "광고" 라벨은 그대로 유지됩니다 — 광고 자체가 사라지진 않습니다.
- 배너 이미지는 `object-fit: contain`으로 렌더링되어 원본 비율이 728x90/300x250과 달라도
  잘리거나 찌그러지지 않고 여백(레터박스)으로 처리됩니다.
- 저장 후 새로고침하면 바로 반영됩니다. 수정한 JSON이 유효한지는
  `python -m json.tool ads.json`으로 확인할 수 있습니다.

> 참고: `frame-ancestors`(클릭재킹 방지)는 `<meta>` 태그로는 브라우저가 무시하므로 이 페이지의
> CSP meta에는 포함하지 않았습니다. 필요하다면 Cloudflare Pages의 `_headers` 파일 등
> HTTP 응답 헤더 레벨에서 `Content-Security-Policy: frame-ancestors 'none'`을 추가로 설정하세요.

### CSP(Content Security Policy) 갱신 필수

`index.html` 상단의 `<meta http-equiv="Content-Security-Policy">` 태그는 인라인 스크립트를
`sha256` 해시로만 허용하고 외부 스크립트를 기본적으로 차단합니다. 광고 스크립트를 추가하면
반드시 다음을 함께 수정해야 합니다.

### 구글 애드센스 삽입 시 CSP 도메인

표준 디스플레이 광고(로더 스크립트 하나를 `<script src="...">`로 넣고, 실제 광고는 구글이
서빙하는 샌드박스 iframe 안에서 렌더링되는 방식)는 아래 도메인만 CSP에 추가하면 됩니다 —
**해시 방식을 건드릴 필요가 없습니다**(이유는 아래 참고).

```
script-src ... https://pagead2.googlesyndication.com https://googleads.g.doubleclick.net https://partner.googleadservices.com;
img-src ... https://pagead2.googlesyndication.com https://googleads.g.doubleclick.net;
frame-src https://googleads.g.doubleclick.net https://tpc.googlesyndication.com https://www.google.com;
connect-src ... https://pagead2.googlesyndication.com https://googleads.g.doubleclick.net;
```

(카카오 애드핏을 쓴다면 `t1.daumcdn.net`을 같은 방식으로 `script-src`/`img-src`에 추가합니다.)
실제 삽입 후 브라우저 콘솔에 CSP 위반 로그가 뜨면 거기 찍힌 도메인을 추가로 허용하세요 —
광고 네트워크마다, 그리고 시점마다 실제로 쓰는 하위 도메인이 조금씩 달라질 수 있습니다.

### "해시 방식이 무력화된다"는 게 무슨 뜻인가

지금 CSP는 `script-src`에 **정적으로 정해진 3개 인라인 스크립트의 sha256 해시만** 허용하고
있습니다(`'unsafe-inline'` 없음). 표준 디스플레이 광고처럼 광고가 **외부 파일(`src=`)로
로드**되거나 **별도 iframe 안에서** 렌더링되는 경우, 이 페이지의 CSP는 그 iframe 내부에는
아예 적용되지 않으므로 해시 방식을 그대로 유지한 채 도메인만 추가하면 끝입니다.

문제는 **Auto ads나 일부 광고 포맷이 최상위 문서(top-level document)에 직접 인라인
`<script>`를 동적으로 주입**하는 경우입니다. 이 스크립트 내용은 광고가 노출될 때마다
달라지므로(어떤 광고가 뜨는지 미리 알 수 없음) **sha256 해시를 사전에 계산해둘 방법이
없습니다** — 해시 기반 CSP는 본질적으로 "내용을 미리 알고 있는" 스크립트에만 쓸 수 있는
방식이라, 여기엔 적용할 수 없습니다.

이때 흔히 떠올리는 우회책이 "그럼 `'unsafe-inline'`도 같이 추가하면 되지 않나?"인데, **CSP
스펙상 같은 `script-src` 안에 해시나 nonce가 하나라도 있으면 브라우저는 `'unsafe-inline'`을
완전히 무시합니다**(하위 호환을 위한 명세, CSP2부터 적용). 즉 해시를 유지한 채 옆에
`'unsafe-inline'`을 추가해봤자 아무 효과가 없고, 광고의 동적 인라인 스크립트는 여전히
차단됩니다.

**대안:**

1. **(권장) 표준/디스플레이 광고 단위만 사용**하고 Auto ads처럼 동적 인라인 주입이 필요한
   포맷은 피합니다. 대부분의 애드센스 광고 단위는 iframe 기반이라 이 문제 자체가 발생하지
   않습니다.
2. 정말 동적 인라인 스크립트가 필요하다면, **해시 소스를 전부 제거하고 `'unsafe-inline'`만
   남기되 `script-src`의 도메인 허용목록은 계속 좁게 유지**하세요(`'self'` + 광고 네트워크
   도메인만, 와일드카드 금지). 우리 자체 스크립트에 대한 무결성 보장(해시 검증)은 포기하는
   대신, 최소한 "허용되지 않은 임의 도메인에서 스크립트를 불러오는" 공격은 계속 막을 수
   있습니다. 이 경우 `tools/verify-csp.js`도 더 이상 의미가 없어지므로 참고용으로만 남겨두면
   됩니다.
3. 넌스(nonce) 기반 CSP는 요청마다 서버가 새 값을 발급해야 해서 **정적 사이트 구조와는 원천적으로
   맞지 않습니다** (미리 빌드된 HTML 파일에는 매 요청마다 바뀌는 값을 넣을 수 없음) — 이
   프로젝트에서는 선택지가 아닙니다.

### 인라인 스크립트를 직접 수정한 경우

`index.html`의 `<script>` 블록(JSON-LD 2개 + 본문 로직 1개) 내용을 바꾸면 `sha256` 해시가
달라지므로, CSP meta 태그의 해시값도 다시 계산해서 갱신해야 스크립트가 차단되지 않습니다.

가장 쉬운 방법은 `npm run verify`(`tools/verify-csp.js`)를 실행하는 것입니다 — 불일치하는
해시를 정확히 짚어주고, 새로 계산된 값도 함께 출력합니다. 그 값을 CSP meta 태그의
`script-src`에 그대로 붙여넣으면 됩니다.

수동으로 계산하고 싶다면 PowerShell 예시:

```powershell
$content = [System.IO.File]::ReadAllText("index.html", [System.Text.Encoding]::UTF8)
$sha = [System.Security.Cryptography.SHA256]::Create()
[regex]::Matches($content, '<script(?:\s[^>]*)?>([\s\S]*?)</script>') | ForEach-Object {
  $bytes = [System.Text.Encoding]::UTF8.GetBytes($_.Groups[1].Value)
  "sha256-" + [Convert]::ToBase64String($sha.ComputeHash($bytes))
}
```

출력된 3개의 해시를 CSP meta 태그의 `script-src`에 순서대로 넣어주면 됩니다.
(간단히 우회하고 싶다면 `script-src`에 `'unsafe-inline'`을 추가하는 방법도 있지만,
그러면 XSS 방어력이 약해지므로 권장하지 않습니다.)

## 배포 전 체크리스트

- [x] `index.html`(canonical, OG `og:url`, JSON-LD `url`), `sitemap.xml`(`<loc>`), `robots.txt`(`Sitemap:`)의
      도메인을 전부 `https://aggrotitle.com`으로 교체함 (2026-08-24).
- [ ] `npm run verify`로 CSP 해시 일치 확인 (아래 "최종 로컬 점검" 참고)
- [ ] GitHub 저장소 생성 및 push
- [ ] Cloudflare Pages 연결 및 배포
- [ ] Custom domain(aggrotitle.com) 연결
- [ ] Google Search Console 등록 및 사이트맵 제출

## 배포 절차 (aggrotitle.com → Cloudflare Pages)

### 1) GitHub 저장소 생성 → push

GitHub에서 새 저장소를 먼저 만든 뒤(비어 있는 상태로, README/license 자동 생성 없이), 발급된
저장소 URL로 아래를 실행합니다. `git init`/`add`/`commit`은 이미 로컬에서 완료된 상태이므로
`remote add`와 `push`만 하면 됩니다.

```bash
git remote add origin https://github.com/<사용자명>/<저장소명>.git
git branch -M main
git push -u origin main
```

### 2) Cloudflare Pages 연결

1. Cloudflare 대시보드 → **Workers & Pages** → **Create application** → **Pages** → **Connect to Git**
2. 방금 push한 저장소를 선택
3. 빌드 설정은 정적 파일이므로 다음과 같이 비워 둡니다 (아래 "빌드 스텝이 필요 없는 이유" 참고).
   - Build command: *(비워둠)*
   - Build output directory: `/` (저장소 루트에 `index.html`이 있으므로)
4. **Save and Deploy** 클릭 → 배포 완료 후 발급된 `*.pages.dev` 주소로 먼저 정상 동작을 확인

### 3) Custom domain 연결 — aggrotitle.com

1. 배포된 Pages 프로젝트 화면 → **Custom domains** 탭 → **Set up a custom domain**
2. `aggrotitle.com` 입력 → Continue
3. `aggrotitle.com`이 이미 해당 Cloudflare 계정에 등록된 도메인이면 DNS 레코드가 자동으로 붙습니다.
   그렇지 않다면(도메인을 다른 곳에서 구매) 화면에 안내되는 CNAME(또는 A) 레코드를 도메인
   등록기관의 DNS 설정에 직접 추가해야 합니다.
4. DNS 전파(보통 몇 분, 길면 몇 시간) 후 `https://aggrotitle.com` 접속 확인 — Cloudflare가 SSL
   인증서를 자동 발급합니다.
5. `www.aggrotitle.com`도 쓰고 싶다면 같은 방식으로 추가한 뒤, Cloudflare **Redirect Rules**로
   `www` → apex(또는 반대)로 통일해 중복 콘텐츠(SEO 페널티 요인)를 방지하세요.

### 4) Google Search Console 등록 + 색인 요청

1. https://search.google.com/search-console 접속 → **속성 추가**
   - **URL 접두어** 방식: `https://aggrotitle.com/` 입력 → HTML 파일 업로드 또는
     `<meta name="google-site-verification" ...>` 태그로 소유권 확인. 이 meta 태그는
     스크립트가 아니라 CSP와 무관하게 항상 허용되니 `index.html` `<head>`에 그냥 추가하면 됩니다.
   - **도메인** 방식(권장, www/apex를 한 번에 포함): DNS TXT 레코드 추가로 인증 — CSP와도
     완전히 무관해서 더 간단합니다.
2. 소유권 확인 후 왼쪽 메뉴 **Sitemaps** → `sitemap.xml` 입력 → 제출
3. 상단 **URL 검사** 도구에 `https://aggrotitle.com/` 입력 → **색인 생성 요청** 클릭 (신규
   페이지는 이렇게 수동 요청하면 크롤링이 며칠 앞당겨집니다)
4. 참고: `robots.txt`/`sitemap.xml`은 이미 `aggrotitle.com` 기준으로 반영돼 있으므로 별도 수정
   없이 그대로 제출하면 됩니다.

### GitHub Pages (대안)

Cloudflare Pages 대신 GitHub Pages를 쓰려면:

1. 저장소 설정(Settings) → **Pages** 메뉴로 이동합니다.
2. **Source**를 `Deploy from a branch`로 설정하고, 브랜치(예: `main`)와 폴더(`/root` 또는 `/docs`)를 지정합니다.
3. 저장 후 몇 분 뒤 `https://<사용자명>.github.io/<저장소명>/` 주소로 접속해 확인합니다.
4. 커스텀 도메인 `aggrotitle.com`을 연결하려면 저장소 Settings → Pages → Custom domain에 입력하고,
   도메인 등록기관 DNS에 GitHub Pages가 안내하는 A/CNAME 레코드를 추가합니다.

## Cloudflare Pages에 빌드 스텝이 필요 없는 이유

이 프로젝트는 번들러·트랜스파일러·CSS 전처리기를 전혀 쓰지 않습니다. `index.html`은 CSS/JS가
전부 인라인이고, 유일한 런타임 의존성인 `patterns.json`은 `fetch()`로 같은 폴더에서 그대로
읽어옵니다. `package.json`이 저장소에 있긴 하지만 `"scripts"`에 `build`가 정의돼 있지 않고
`dependencies`도 없어서, Cloudflare Pages가 `npm install`을 자동 실행하더라도(설치할 패키지가
없어 즉시 끝남) 결과물이 달라지지 않습니다. **Build command를 비워두면 Cloudflare Pages는
저장소 파일을 그대로 서빙**하며, 이게 정확히 우리가 원하는 동작입니다.

## 최종 로컬 점검 (배포 직전)

푸시하기 전에 로컬에서 아래 두 가지를 확인하세요.

1. **CSP 해시 검증**
   ```bash
   npm run verify
   ```
   모든 인라인 스크립트가 `[OK]`로 나오고 `[PASS]`로 끝나야 합니다. `patterns.json`은 CSP와
   무관하니(별도 파일로 `fetch()`) 여기서 체크되지 않습니다 — JSON 문법은 `python -m json.tool
   patterns.json`으로 별도 확인하세요.

2. **로컬 서버로 전 기능 수동 확인**
   ```bash
   python -m http.server 8000
   ```
   `http://localhost:8000/`에서 아래를 브라우저 개발자 도구 콘솔을 열어둔 채로 확인합니다
   (에러/경고 0건이어야 함).
   - 키워드 입력 → **제목 계산**: 결과 8개, 점수 배지, 근거 태그, `?q=`/`style=` 쿼리스트링 반영
   - **더 보기**: 8개 추가(총 16개), 직전 결과와 안 겹치는지
   - **다시 뽑기**: 전체 새 8개로 교체
   - **복사** 버튼: 클립보드 복사 후 버튼 라벨이 "복사됨"으로 1.5초간 바뀌는지
   - URL에 `?q=...&style=...` 쿼리스트링을 직접 붙여 새로고침 → 자동 계산되는지
   - 스타일 라디오(개념글/일반글/혼합), 확장자 체크박스 토글 후 재계산 정상 동작

## 주의사항

- `patterns.json`의 단어 사전은 일반적인 커뮤니티/언론 어휘로만 구성되어 있습니다. 특정 인물·집단
  비하, 성적 대상화, 미성년 관련 표현은 추가하지 마세요.
- 생성 로직(`generateTitles(keyword, options, patterns)`)은 DOM에 접근하지 않는 순수 함수로
  분리되어 있어, 나중에 서버 사이드(Node.js 등)로 그대로 옮겨 재사용할 수 있습니다.
