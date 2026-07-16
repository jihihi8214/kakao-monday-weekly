# 설정 가이드 (주요 부처 주간 보도계획 · 일정 크롤러)

이 크롤러는 연합인포맥스(news.einfomax.co.kr)와 행정안전부 홈페이지에서 이번주 부처별
보도계획/일정을 긁어옵니다. 연합인포맥스는 봇 차단이 있어서 API 키가 아니라 실제 브라우저
쿠키가 필요하고, 이 쿠키는 시간이 지나면 만료될 수 있습니다 (만료되면 워크플로가
"수집된 데이터가 없습니다"라며 실패하듯 끝나는데, 그때 4번 과정을 다시 하시면 됩니다).

## 1. GitHub 저장소 생성

1. https://github.com 에서 새 저장소 생성 (예: weekly-schedule-crawler), Public으로 설정

## 2. 파일 업로드

1. "Add file" → "Upload files"로 이 폴더의 `crawler-3.py`, `index.html`, `.gitignore`를 올립니다.
2. `.github/workflows/weekly-schedule.yml`은 점(.)으로 시작하는 폴더라 드래그로 누락되기 쉬우니,
   "Add file" → "Create new file"에서 파일명 칸에 `.github/workflows/weekly-schedule.yml`을
   슬래시 포함해서 한 번에 입력하고, 아래 내용을 붙여넣어 만드세요.

```yaml
name: Weekly report schedule crawler

on:
  schedule:
    - cron: "45 0 * * 1"
  workflow_dispatch:

permissions:
  contents: write

concurrency:
  group: weekly-schedule-crawler
  cancel-in-progress: false

jobs:
  run-weekly-schedule-crawler:
    runs-on: ubuntu-latest
    env:
      TZ: Asia/Seoul
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install requests beautifulsoup4 urllib3

      - name: Configure git
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

      - name: Run weekly schedule crawler
        run: |
          python crawler-3.py
        env:
          EINFOMAX_COOKIE: ${{ secrets.EINFOMAX_COOKIE }}
          EINFOMAX_USER_AGENT: ${{ secrets.EINFOMAX_USER_AGENT }}
```

## 3. GitHub Pages 활성화

1. Settings → Pages → Source "Deploy from a branch" → Branch "main" / "(root)" → Save
2. 표시되는 URL이 고정 주소입니다.

## 4. 연합인포맥스 쿠키 발급 (가장 중요, 주기적으로 반복 필요)

1. 크롬에서 https://news.einfomax.co.kr 접속
2. 개발자 도구 열기 (Mac: `Cmd + Option + I`)
3. 아무 기사나 클릭해서 들어가기
4. 개발자 도구 상단 "Network(네트워크)" 탭 클릭
5. 목록에서 `articleView` 로 시작하는 요청을 클릭
6. 우측 "Headers(헤더)" 안의 "Request Headers"에서 `Cookie:` 값과 `User-Agent:` 값을 찾아
   각각 전체를 복사

## 5. Secret 등록

저장소 Settings → Secrets and variables → Actions → New repository secret

| Name | Value |
|---|---|
| EINFOMAX_COOKIE | (4번에서 복사한 Cookie 값 전체) |
| EINFOMAX_USER_AGENT | (4번에서 복사한 User-Agent 값 전체) |

## 6. 첫 실행 테스트

1. Actions 탭 → "Weekly report schedule crawler" → "Run workflow"
2. 로그에서 "✅ 데이터 저장 완료" 가 뜨는지, 실패하면 "쿠키가 만료됐을 수 있으니" 메시지가
   뜨는지 확인 (뜨면 4~5번을 다시 반복)
3. 성공하면 Pages URL에서 이번주 일정이 보이는지 확인

이후로는 매주 월요일 오전 9시45분(KST)에 자동 실행됩니다. 다만 쿠키 만료로 실패하기 시작하면
그때마다 4~5번 과정을 반복해서 Secret을 갱신해주셔야 합니다 — 이 부분만큼은 완전 자동화가
어렵다고 원래 인수인계 문서에도 나와 있던 내용이에요.
