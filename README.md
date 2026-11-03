# ai-news-digest

Claude Code에서 쓰는 플러그인입니다. AI 코딩 도구 소식 네 곳을 가져와 **새로 나온 것 · 바뀐 것 · 고친 것 · 곧 바뀌거나 없어질 것 · 그 밖 소식** 다섯 칸으로 나누고, 항목마다 한국어 요약을 붙여 대화창이나 파일로 보여 줍니다.

가져오는 곳(이 넷만):

| 이름 | 어디서 |
|---|---|
| `claude-code` | Claude Code GitHub 릴리스 |
| `codex` | OpenAI Codex GitHub 릴리스(정식판만 — 알파 · 미리보기는 뺌) |
| `anthropic-news` | Anthropic 뉴스 페이지(고른 글은 글 페이지 본문까지) |
| `openai-news` | OpenAI 뉴스 RSS(소개문까지 — 원문 페이지는 받지 않음) |

## 설치

Claude Code 대화창에서:

```
/plugin marketplace add kuntae802/ai-news-digest
/plugin install ai-news-digest@vcu-ai-news
```

설치 뒤 Claude Code를 다시 시작하거나 `/reload-plugins`를 실행합니다.

필요한 것:

- **Linux 또는 WSL**(Windows는 WSL 안에서 Claude Code를 실행). 임시 작업 폴더를 `/tmp`에 두고, 그 경로로 권한을 좁혀 두었기 때문입니다. macOS는 규칙을 함께 적어 두었지만 시험하지 않았습니다.
- `python3`(3.9 이상) — 따로 설치할 파이썬 패키지는 없습니다.
- Claude Code **2.1.317**에서 확인했습니다.

## 부르는 법

```
/ai-news-digest:report
/ai-news-digest:report sources=claude-code,codex range=new depth=detail
/ai-news-digest:report categories=new,deprecating output=file out_dir=./news
```

인자는 모두 고를 수 있고, 안 적으면 기본값입니다.

| 인자 | 값 | 기본값 |
|---|---|---|
| `sources` | `claude-code` · `codex` · `anthropic-news` · `openai-news` 중 쉼표로 | 넷 다 |
| `range` | `recent` = 최근 몇 개 / `new` = 지난번 이후 새 것만 | `recent` |
| `limit` | 소스마다 몇 개(1~20) | `5` |
| `depth` | `brief` = 한 줄 / `detail` = 몇 문장 | `brief` |
| `categories` | `new` · `changed` · `fixed` · `deprecating` · `other` 중 쉼표로 | 다섯 다 |
| `output` | `chat` = 대화창 / `file` = 파일로 남김 | `chat` |
| `out_dir` | `output=file`일 때 남길 폴더 | `./ai-news-digest` |

- `range=new`는 지난번에 **검사를 통과한** 리포트에서 본 글을 뺍니다. 소스마다 최신 50개를 후보로 봅니다.
  - 처음 부르면 `recent`처럼 최신 몇 개를 보여 주고, 그때 후보 50개 전부를 "본 글"로 기록합니다. 그래서 다음부터는 정말 새로 올라온 글만 나옵니다.
  - 새 글이 `limit`보다 많으면 **오래된 것부터** `limit`개를 보여 주고, 나머지는 다음 실행에서 이어서 보여 줍니다(빠지는 글 없음).
- `depth` · `categories`는 대화창에 보여 줄 때만 씁니다. 파일에는 항상 가져온 항목 전부와 한 줄 · 자세한 요약이 둘 다 들어갑니다.
- `output=file`이면 `<out_dir>/<시각>.json`(결과)과 `<시각>.fetched.json`(가져온 원본)이 남습니다. 시각은 UTC입니다.

## 어떻게 도나

1. `ai-news-fetch`(파이썬)가 네 곳에서 글을 가져와 기록용 `fetched.json`을 쓰고, Claude가 읽을 보기 파일(`fetched_view.md`)을 임시 작업 폴더(`/tmp/ai-news-digest-…`)에 둡니다.
2. Claude가 보기 파일을 읽고 칸을 나누고 한국어로 요약해 같은 작업 폴더에 `result.json`을 씁니다.
3. 대화가 끝날 때 플러그인의 Stop 훅이 검사(`ai-news-check`)를 돌립니다. 가져온 항목이 빠지거나 덧붙거나, 링크 · 제목 · 날짜 · 오류 목록이 가져온 것과 다르거나, 칸 이름이 틀리거나, 요약이 비어 있으면 Claude에게 고치라고 돌려보냅니다. 두 번째에도 실패하면 멈춤을 막지 않고 그 리포트를 포기 표시로 남깁니다(기록은 바뀌지 않음).
4. 검사를 통과해야 결과가 기록 폴더로 옮겨지고 "본 글" 기록이 갱신됩니다.

Stop 훅은 이 플러그인을 켠 동안 모든 대화의 끝에서 불리지만, **이 스킬을 부른 그 대화**에 만든 지 2시간이 안 된 대기 표시가 있을 때만 검사하고 나머지는 바로 지나갑니다.

### 권한 범위

스킬이 묻지 않고 쓰는 권한은 이것뿐입니다.

| 규칙 | 뜻 |
|---|---|
| `Bash(ai-news-fetch *)` · `Bash(ai-news-check *)` | 이 플러그인의 두 프로그램만 실행 |
| `Read(//tmp/ai-news-digest-*/**)` | 임시 작업 폴더 안 파일만 읽기 |
| `Edit(//tmp/ai-news-digest-*/**)` | 임시 작업 폴더 안 파일만 쓰기 · 고치기(`Edit` 규칙이 Write 도구에도 걸립니다) |
| `//private/tmp/…` 두 줄 | macOS에서 `/tmp`가 `/private/tmp`로 이어지는 경우를 위한 같은 규칙 |

결과를 플러그인 데이터 폴더에 바로 쓰지 않는 까닭: Claude Code는 경로에 `.claude`가 든 파일을 민감 파일로 보아, 허용 규칙이 있어도 쓰기 · 고치기를 막습니다. 그래서 작업은 `/tmp`에서 하고, 기록 폴더로 옮기는 일은 검사 프로그램이 합니다.

가져온 글은 요약할 자료로만 다루고, 글 안의 문장을 지시로 따르지 않게 해 두었습니다.

## 기록이 남는 곳

플러그인 데이터 폴더(`~/.claude/plugins/data/` 아래)에 남습니다. 마켓플레이스로 설치했다면 보통 `ai-news-digest-vcu-ai-news`, `--plugin-dir`로 띄웠다면 `ai-news-digest-inline` 폴더입니다(둘은 서로 다른 기록입니다).

| 파일 | 뜻 |
|---|---|
| `state.json` | 소스마다 본 글 ID(최근 200개) — `range=new`의 기준 |
| `runs/<시각>/fetched.json` · `result.json` | 실행마다 가져온 것과 결과(결과는 검사를 통과한 것만) |
| `pending/<세션 ID>.json` | 그 대화에서 검사를 기다리는 리포트 표시(통과하면 지워짐) |
| `pending/<세션 ID>.abandoned.json` | 두 번 고쳐도 검사를 통과하지 못해 포기한 리포트 표시 |

기록을 처음부터 다시 하려면 이 폴더의 `state.json`을 지우면 됩니다.

## 알아 둘 것

- **구독 사용량을 씁니다.** 요약 · 분류는 지금 쓰는 Claude Code 계정(구독이나 API 키)으로 돌아갑니다. 소스와 `limit`을 늘릴수록 많이 씁니다.
- 요청 예절: 요청마다 `User-Agent: ai-news-digest/<버전>`(`plugin.json`의 버전)을 밝히고, 요청 사이 1초를 쉬며, 일시적 오류(연결 · 5xx · 429 · 응답이 중간에 잘림)만 두 번까지 다시 시도합니다.
- 본문은 글마다 6,000자까지만 읽습니다.
- Codex 릴리스는 최근 30개 중 정식판만 고르므로, 알파판이 많은 시기에는 `limit`보다 적게 나올 수 있습니다.
