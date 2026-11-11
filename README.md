# 웹서비스설계 9주차 실습 - 검색·정렬·페이지네이션 게시판 개선하기

곽우현 (202446609)

1,000건짜리 게시글 데이터로 검색, 정렬, 오프셋/커서 페이지네이션, 디바운스, 요청 취소, URL 상태 동기화를 테스트해 보고 시작 코드에서 빠져 있거나 잘못된 부분을 고쳤습니다.
고친 곳은 코드에 `[수정 N]` 주석을 달아 놨고, 아래 3번에 전부 정리했습니다.

- 제출물: 이 저장소(GitHub 링크), 실습 보고서와 테스트 캡처(LMS 파일)
- 시작 코드: 수업에서 받은 `week09-practice` (`server.js`, `index.html`)

---

## 1. 실행 방법 (Windows PowerShell)

```powershell
git clone https://github.com/zoe6rain/week09-practice.git
cd week09-practice
npm install express        # node_modules 는 저장소에 안 올렸기 때문에 설치가 필요합니다
node server.js             # "서버 구동 완료" 가 뜨면 성공
```

브라우저에서 http://localhost:3000 을 엽니다.
`server.js` 를 고쳤을 때는 `Ctrl + C` 로 서버를 끄고 `node server.js` 를 다시 실행해야 반영됩니다.
조회수는 서버를 켤 때마다 랜덤으로 새로 만들어져서 캡처랑 숫자가 다를 수 있습니다.

| 파일 | 내용 |
|---|---|
| `server.js` | Express 서버. 오프셋 API `/api/posts`, 커서 API `/api/posts/cursor` |
| `index.html` | 게시판 화면(검색, 정렬, 페이지, 커서 더 보기)과 오른쪽 요청/SQL/응답 검사기 |
| `docs/` | 실습 보고서(`week09_report.pdf`)와 테스트 캡처(`docs/captures/`, 원본은 `docs/captures/original/`) |

---

## 2. 요구사항별 확인 결과 (요약)

입력, 기대 결과, 실제 결과는 보고서에 자세히 적었고, 캡처 번호(W..)는 `docs/captures/` 파일 이름과 같습니다.

| 영역 | 확인한 내용 | 결과 | 캡처 |
|---|---|---|---|
| 검색·정렬 | 제목/본문 검색, 결과 없음 안내, 최신/오래된/조회수 정렬, 이상한 정렬 값은 최신순으로, `__proto__`·`constructor` 막기 | 통과 (수정 1) | W03~W06, W14-6, W16-1, W16-2, W16-5 |
| 오프셋 페이지네이션 | `page=0/-1/abc`, `limit=1000`, 범위를 넘은 page, 정렬·검색 바꾸면 1페이지로 | 통과 | W15-1~5, W16-3, W16-4, W16-6 |
| 커서 페이지네이션 | 검색 포함, `limit+1`로 hasNext 계산, id DESC 고정(정렬 비활성), 새 검색하면 목록 초기화, 끝에서 버튼 비활성 | 통과 (수정 2~5) | W07~W11 |
| 비동기 | 300ms 디바운스(요청 1번), AbortController 취소, 오래된 응답이 덮어쓰지 않음, AbortError 처리 | 통과 | W12, W13 |
| URL 상태 | 검색·정렬·page가 주소창과 같이 바뀜, 새로고침·뒤로 가기 후 복원, 잘못된 값 보정 | 통과 (수정 7~9) | W14-1~6, W17 |

---

## 3. 수정한 코드

### [수정 1] server.js: 정렬 키 `__proto__`, `constructor` 막기

원래 코드는 `SORT_WHITELIST[sortKey]` 로 정렬 함수를 찾았습니다. 그런데 자바스크립트 객체에는 `__proto__`, `constructor`, `hasOwnProperty` 처럼 상속된 속성이 있어서, 이런 값이 sort로 들어오면 서버가 500 에러가 났습니다(W03). 그래서 화이트리스트에 직접 들어 있는 키인지 먼저 확인하도록 바꿨습니다.

```js
// 수정 전
const sortComparator = SORT_WHITELIST[sortKey] || SORT_WHITELIST.latest;

// 수정 후
const sortComparator = Object.keys(SORT_WHITELIST).includes(sortKey)
  ? SORT_WHITELIST[sortKey]
  : SORT_WHITELIST.latest;
```

### [수정 2] server.js: 커서 API가 검색어를 무시하던 문제

커서 API는 `search` 를 안 받아서 `?search=Redis` 를 보내도 전체 글이 나왔습니다(W04). 오프셋 API처럼 검색 조건을 먼저 적용하고, 그 결과에서 `id < cursor` 를 적용하도록 했습니다. 아래 `limit + 1` 부분은 그대로입니다.

```js
const search = (req.query.search || '').trim().toLowerCase();

let candidate = POSTS.filter(p => {
  if (!search) return true;
  return p.title.toLowerCase().includes(search) || p.content.toLowerCase().includes(search);
});
if (!isNaN(rawCursor) && rawCursor > 0) {
  candidate = candidate.filter(p => p.id < rawCursor);
}
```

### [수정 3] index.html: 커서 요청에도 검색어 보내기

```js
if (state.cursor) q.set('cursor', state.cursor);
if (state.search) q.set('search', state.search); // [수정 3]
```

### [수정 4] index.html: 커서 모드에서는 정렬 선택 막고 안내 문구 넣기

커서 방식은 `id DESC` 로 고정이라 다른 정렬을 고를 수 없습니다. 그래서 정렬 선택을 `disabled` 로 막고 이유를 화면에 적었습니다(`cursorNote`). 모드를 바꾸는 코드는 `updateModeUI()` 로 따로 빼서 `switchMode` 와 뒤로 가기에서 같이 쓰게 했습니다.

```js
function updateModeUI() {
  document.getElementById('tabOffset').classList.toggle('active', mode === 'offset');
  document.getElementById('tabCursor').classList.toggle('active', mode === 'cursor');
  document.getElementById('paginationBar').style.display = (mode === 'offset') ? 'flex' : 'none';
  document.getElementById('cursorMoreWrap').style.display = (mode === 'cursor') ? 'flex' : 'none';
  document.getElementById('sortSelect').disabled = (mode === 'cursor');
  document.getElementById('cursorNote').style.display = (mode === 'cursor') ? 'block' : 'none';
}
```
```html
<p id="cursorNote" style="display:none; ...">커서 모드는 id 내림차순(최신순)으로 고정되어 정렬을 선택할 수 없습니다.</p>
```

### [수정 5] index.html: 커서 모드 SQL 미리보기에 검색 조건 표시

```js
let conds = [];
if (state.search) conds.push(`(title LIKE '%${state.search}%' OR content LIKE '%${state.search}%')`);
if (state.cursor) conds.push(`id < ${state.cursor}`);
let whereClause = conds.length ? `WHERE ${conds.join(' AND ')} ` : '';
```

### [수정 6] index.html: 조회 버튼과 디바운스 타이머가 겹쳐서 요청이 두 번 나가는 문제

```js
function applySearch() {
  clearTimeout(debounceTimer); // [수정 6]
  ...
}
```

### [수정 7] index.html: 서버가 고쳐 준 page를 화면 상태와 주소창에도 맞추기

서버는 `page=-1` 이나 `page=999` 가 오면 1페이지나 마지막 페이지로 보정해서 돌려줍니다. 그 값에 맞춰서 화면의 `state.page` 와 주소창도 같이 바꿉니다.

```js
state.page = json.meta.page;
const urlPage = new URLSearchParams(window.location.search).get('page');
if (urlPage !== null && urlPage !== String(state.page)) {
  const fixed = new URLSearchParams(window.location.search);
  fixed.set('page', state.page);
  window.history.replaceState({}, '', `?${fixed.toString()}`);
}
```

### [수정 8] index.html: 새로고침과 뒤로 가기에서 같은 복원 함수 쓰기

원래는 `popstate` 와 `DOMContentLoaded` 에 거의 똑같은 코드가 두 번 있었고, 정렬 값도 검사하지 않았고, 커서 모드에서 뒤로 가면 예전 커서가 남을 수 있었습니다. `restoreStateFromURL()` 하나로 합쳐서 허용된 정렬 값만 쓰고, 커서 상태를 비우고, 오프셋 모드로 돌려놓게 했습니다.

```js
function restoreStateFromURL() {
  clearTimeout(debounceTimer);
  const params = new URLSearchParams(window.location.search);
  const sortList = ['latest', 'views', 'oldest'];
  ...
  state.sort = sortList.includes(params.get('sort')) ? params.get('sort') : 'latest';
  state.search = params.get('search') || '';
  state.cursor = null;
  cursorPosts = [];
  mode = 'offset';
  updateModeUI();
  document.getElementById('searchInput').value = state.search;
  document.getElementById('sortSelect').value = state.sort;
  ...
}
window.addEventListener('popstate', () => { restoreStateFromURL(); loadData(false); });
window.addEventListener('DOMContentLoaded', () => { restoreStateFromURL(); loadData(false); });
```

### [수정 9] index.html: 잘못된 `page` 값을 주소창에서도 바로 고치기

처음에는 [수정 7]만 하면 `?page=-1` 이 주소창에서도 `page=1` 로 바뀔 줄 알았습니다. 그런데 실제로는 주소창에 `page=-1`, `page=abc` 가 그대로 남았고(W14-4, W14-5 1차 캡처), SQL 미리보기에도 `OFFSET -20` 이 나왔습니다. 나중에 보니 [수정 7] 코드를 오프셋 모드 쪽이 아니라 커서 모드 쪽 `else` 안에 잘못 넣어 놨던 거였고, 위치를 옮긴 뒤 W17 에서 제대로 되는 걸 확인했습니다. 그때는 이걸 몰라서 `restoreStateFromURL()` 에서 page를 먼저 고치고 주소창도 바로 바꾸게 했고, 재캡처에서 통과했습니다.

```js
const rawPage = parseInt(params.get('page'), 10);
state.page = rawPage > 0 ? rawPage : 1;      // 0, 음수, 숫자 아닌 값은 1페이지로
...
const fixed = new URLSearchParams({ page: state.page, limit: state.limit, sort: state.sort });
if (state.search) fixed.set('search', state.search);
window.history.replaceState({}, '', `?${fixed.toString()}`);
```

---

## 4. 개념 설명

### ① 오프셋과 커서는 어떤 화면에 맞는가

오프셋은 "몇 번째부터 몇 개"로 가져오는 방식입니다(`LIMIT 10 OFFSET 20`). 커서는 "마지막으로 본 글의 id 다음부터 몇 개"로 가져옵니다(`WHERE id < ? ORDER BY id DESC LIMIT 11`).

- 오프셋이 맞는 화면: 게시판 목록, 관리자 목록, 검색 결과처럼 3페이지나 마지막 페이지로 바로 가야 하고, 그 위치를 주소(URL)로 공유하거나 복원해야 하는 화면입니다. 총 페이지 수도 보여 줄 수 있습니다.
- 커서가 맞는 화면: 더 보기 버튼, 무한 스크롤, 피드나 타임라인처럼 다음 묶음만 계속 이어서 보는 화면입니다. 전체 개수를 세지 않아도 되고, 새 글이 올라와도 중복이 적습니다.
- 한계: 오프셋은 뒤쪽 페이지일수록 앞의 행을 건너뛰어야 해서 느려질 수 있고, 목록이 바뀌면 중복이나 누락이 생길 수 있습니다. 커서는 임의의 페이지로 못 가고, 정렬이 id 기준으로 고정되고, 총 페이지 수를 알 수 없습니다.

이번 게시판은 `page` 를 주소창에 저장하고 뒤로 가기와 새로고침으로 복원해야 해서 오프셋 모드가 기본입니다. 커서 모드는 더 보기 화면이고 id 내림차순으로 고정돼 있어서 정렬 선택을 막았습니다([수정 4]).
다만 이 실습은 1,000건짜리 배열이라 두 방식의 속도 차이는 거의 없습니다(응답 시간이 0.2~0.9ms 정도였습니다). 위 한계는 DB에 데이터가 아주 많을 때의 일반적인 이야기입니다.

### ② 디바운스와 요청 취소(AbortController)의 역할

둘은 막는 문제가 서로 다릅니다.

- 디바운스(300ms): 글자를 칠 때마다 요청을 보내지 않고, 입력이 300ms 동안 멈추면 그때 한 번만 보냅니다. 입력할 때마다 이전 타이머를 `clearTimeout` 으로 지우고 `setTimeout` 으로 300ms를 다시 셉니다. 보내는 요청의 수를 줄여 줍니다. (W12: `redis` 를 빠르게 쳐도 요청이 1개)
- 요청 취소(AbortController): 이미 서버로 나간 이전 요청을 새 요청을 보낼 때 `abort()` 로 취소합니다. 네트워크가 느릴 때 오래된 요청의 응답이 늦게 도착해서 최신 검색 결과를 덮어쓰는 걸 막아 줍니다. 취소된 요청은 `AbortError` 를 던지는데, 오류 화면을 띄우지 않고 콘솔에 로그만 남기게 했습니다. (W13: `redis` 에서 `abort` 로 바꾸면 앞 요청이 `(canceled)` 가 되고 화면에는 `abort` 결과만 남음)

디바운스만 쓰면 이미 나간 요청의 응답이 뒤바뀌는 걸 못 막고, 취소만 쓰면 요청 횟수는 그대로라서 둘을 같이 씁니다.

### ③ `limit + 1` 의 원리

다음 글이 더 있는지(`hasNext`)를 알려면 보통 전체 개수를 세는 `COUNT(*)` 가 필요합니다. 대신 필요한 개수보다 1개 더(`limit + 1`) 가져와 보면 됩니다.

- `limit + 1` 개가 오면 뒤에 더 있다는 뜻이라 `hasNext = true` 이고, 응답에는 앞의 `limit` 개만 담습니다.
- `limit` 개 이하로 오면 더 없다는 뜻이라 `hasNext = false` 입니다.

```js
const sliced = candidate.slice(0, limit + 1);
const hasNext = sliced.length > limit;
const items = hasNext ? sliced.slice(0, limit) : sliced;
const nextCursor = items.length > 0 ? items[items.length - 1].id : null;  // 다음 요청의 cursor
```

예를 들어 `Redis` 검색 결과는 83건입니다. 10개씩 8번 받고 마지막에 11개를 요청하면 3개만 와서 `hasNext=false` 가 되고, 버튼이 "더 이상 글이 없습니다"로 바뀌면서 비활성화됩니다(W11, 마지막 글 id 7).

### ④ 오프셋과 `totalPages` 계산식

```js
const totalPages = Math.max(1, Math.ceil(totalCount / limit)); // 올림, 결과가 0건이어도 최소 1페이지
const safePage   = Math.min(page, totalPages);                 // 범위를 넘은 page 는 마지막 페이지로
const offset     = (safePage - 1) * limit;                     // 건너뛸 글 수
const items      = filtered.slice(offset, offset + limit);     // slice 의 끝 인덱스는 포함되지 않음
// hasNext = safePage < totalPages,  hasPrev = safePage > 1
```

순서는 검색(filter), 정렬(sort), 자르기(slice) 입니다. 검색한 결과 안에서 정렬하고 마지막에 잘라야 합니다.

| 상황 | 계산 | 확인 |
|---|---|---|
| 1,000건, `limit=10`, `page=3` | `offset = 20` | W16-3a (SQL `OFFSET 20`) |
| `redis` 83건, `limit=10` | `totalPages = ceil(83/10) = 9`, 9페이지는 `offset = 80` 이라 3개 | W16-6 |
| `limit=1000` 요청 | 서버가 100으로 제한해서 `totalPages = 10` | W15-4 |
| `page=999` 요청 | `totalPages = 100` 이라 `safePage = 100` | W15-5 |
| 검색 결과 0건 | `totalPages = max(1, 0) = 1` | W16-5 |

---

## 5. AI 사용 내역과 직접 확인한 내용

AI는 Claude(Anthropic)를 썼습니다. 도움받은 부분은 아래와 같습니다.

| 부분 | AI(Claude)가 도와준 것 |
|---|---|
| 시작 코드 분석 | 빈틈 찾기: 정렬 키 상속 속성 문제, 커서 검색 누락, 주소창과 화면 상태 불일치 등 |
| 코드 수정 `[수정 1~9]` | 고칠 부분, 원리, 고칠 위치를 설명받고 예시 코드를 받음. 이를 이해한 뒤 제가 직접 파일에 반영함 |
| JavaScript 개념·문법 | 디바운스(`setTimeout`/`clearTimeout`), `AbortController`, `URLSearchParams`, `history.pushState`/`replaceState`/`popstate` 등 개념과 문법 설명 |
| 테스트 방법 | W03~W06은 AI가 알려 준 PowerShell 반복 명령(`curl.exe`, `Invoke-RestMethod`)으로 한꺼번에 확인했음(수업 자료에 없는 방법). W07 이후는 크롬 주소창, 화면 조작, F12(Network/Console)로 확인 |
| 테스트 항목 | 확인할 항목, 순서, 기대 결과 안내 |
| 캡처 강조 표시 | 캡처 위에 빨간·파란 박스와 설명 글자를 AI가 넣어 줌(`docs/captures/`). 원본은 `docs/captures/original/` 에 그대로 있고, 화면 내용은 고치지 않고 박스와 글자만 추가함 |
| README·보고서 | 초안 작성 도움 |

### 직접 검증한 내용

- 코드 수정: 수정 방법과 개념은 AI에게 배우고, 코드는 제가 직접 수정했습니다. `[수정 1~9]` 를 `server.js`, `index.html` 에 직접 넣고 저장한 뒤 서버를 다시 실행했습니다.
- 테스트: W01~W17 테스트를 전부 직접 실행하고(W03~W06은 PowerShell, 나머지는 크롬 주소창과 F12 Network·Console) 결과를 캡처로 확인했습니다. 실패한 시도와 확인하지 못한 부분도 그대로 적었습니다.

### 시행착오 (실패한 것)

- W06 1차: `curl.exe ... | ConvertFrom-Json` 이 한글 때문에 JSON 파싱 오류가 났습니다. 코드 문제가 아니라 AI가 알려 준 명령 문제였고, `Invoke-RestMethod` 로 바꿔서 통과했습니다. (`W06-try1`)
- W13 1차: `Slow 4G` 에서 `redis` 를 지우고 `인덱스` 로 바꿨는데, 앞 요청이 취소되지 않고 끝까지 받아졌고, `인덱스` 는 모든 글에 들어 있어서 결과도 구분이 안 됐습니다. 타이밍을 맞추고 검색어를 `redis` 에서 `abort` 로 바꿔 다시 해서 통과했습니다. (`W13-try1`)
- W14-4, W14-5 1차: `?page=-1`, `?page=abc` 에서 화면은 1페이지로 나왔지만 주소창이 그대로였습니다. [수정 9]를 추가하고 다시 캡처해서 통과했습니다. (`W14-4-try1`, `W14-5-try1`)
- [수정 7] 위치: 처음에 오프셋 모드가 아니라 커서 모드 쪽 `else` 안에 잘못 넣어서 오프셋 모드에서는 동작하지 않았습니다(W14-4, W14-5는 [수정 9]가 대신 처리했습니다). 오프셋 모드 쪽으로 옮긴 뒤 `?page=999` 를 직접 입력해서 주소창이 `page=100` 으로 바뀌는 걸 확인했습니다(W17). 이 캡처의 SQL 미리보기에 나오는 `OFFSET 9980` 은 요청한 999페이지 기준으로 계산한 값이고, 실제 화면은 100페이지입니다.

### 확인하지 못한 부분

- [수정 6](조회 버튼과 디바운스 중복 방지)은 코드에는 넣었지만 따로 캡처로 테스트하지는 못했습니다.
- 조회수순 정렬에서 조회수가 같을 때(id 내림차순) 순서는 조회수가 랜덤이라 화면에서 같은 값을 찾지 못했습니다. 코드(`b.views !== a.views ? b.views - a.views : b.id - a.id`)로만 확인했습니다.
- 선택 심화(DB 연동 등)는 하지 않았습니다.
