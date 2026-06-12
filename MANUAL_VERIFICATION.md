# 수동 검증 가이드

이 문서를 위에서 아래로 따라가면 모든 검증을 직접 재현할 수 있다.

## 0. 사전 준비

- 작업 폴더: `output_b2-1/`
- 실행 환경: Python 3.10 이상
- 검증 전 기존 data 폴더 삭제 (깨끗한 상태로 시작)

```bash
cd output_b2-1
rm -rf data    # Windows: rmdir /s /q data
```

---

## 1. 모듈 로드 및 help 확인

### 1-1. 기본 실행

- **목적**: 패키지가 올바르게 로드되는지, --help가 동작하는지 확인
- **실행**:
  ```bash
  python -m budget_app --help
  ```
- **기대 출력**:
  ```
  usage: budget_app [-h] [--data-dir DATA_DIR]
                    {add,list,search,summary,budget,category,update,delete,export,import} ...
  ```
- **확인 포인트**: 10개 명령이 모두 나열되면 통과.

---

## 2. 초기화 — 기본 카테고리 자동 생성 (안 A)

### 2-1. category list (첫 실행)

- **목적**: 저장 파일 없을 때 기본 카테고리 7개가 자동 생성되는지 확인
- **실행**:
  ```bash
  python -m budget_app category list
  ```
- **기대 출력**:
  ```
  - food
  - transport
  - rent
  - salary
  - shopping
  - utility
  - entertainment
  ```
- **확인 포인트**: data/categories.jsonl 생성 여부도 확인.
  ```bash
  cat data/categories.jsonl    # Windows: type data\categories.jsonl
  ```

---

## 3. 거래 추가(add) 검증

### 3-1. 정상 추가

- **목적**: PDF §add 대화형 입력 + id 출력 확인
- **실행**:
  ```bash
  python -m budget_app add
  ```
  입력값:
  ```
  날짜(YYYY-MM-DD): 2024-01-15
  타입(income/expense): expense
  카테고리: food
  금액(양수): 15000
  메모(선택): 점심
  태그(쉼표로 구분, 없으면 엔터): meal
  ```
- **기대 출력**:
  ```
  [저장 완료] id=TX-000001
  ```
- **확인 포인트**: id가 `TX-XXXXXX` 형식이면 통과.

### 3-2. 날짜 형식 오류 처리

- **목적**: PDF §오류출력 `[오류]+[힌트]` 형식, exit code 1 확인
- **실행**:
  ```bash
  python -m budget_app add
  ```
  입력값:
  ```
  날짜(YYYY-MM-DD): 2024-13-40
  ```
- **기대 출력**:
  ```
  [오류] 날짜 형식이 올바르지 않습니다 (YYYY-MM-DD).
  [힌트] 예: 2024-01-15
  ```
- **확인 포인트**: 스택트레이스가 없어야 하고 `echo $?` (Windows: `echo %errorlevel%`)가 1이면 통과.

---

## 4. 거래 목록(list) 검증

### 4-1. 데이터 추가 후 list

- **목적**: 최신순 정렬, --limit 옵션 확인
- **먼저 추가 데이터 삽입** (income 거래 추가):
  ```bash
  python -m budget_app add
  # 2024-01-14 / income / salary / 3000000 / (빈 메모) / (빈 태그)
  python -m budget_app add
  # 2024-01-12 / expense / transport / 20000 / 버스 / (빈 태그)
  ```
- **실행**:
  ```bash
  python -m budget_app list --limit 3
  ```
- **기대 출력** (날짜 내림차순):
  ```
  TX-000001 | 2024-01-15 | expense | food | 15000 | 점심
  TX-000002 | 2024-01-14 | income | salary | 3000000 | 
  TX-000003 | 2024-01-12 | expense | transport | 20000 | 버스
  ```
- **확인 포인트**: 날짜 내림차순, 6열 파이프 형식이면 통과.

---

## 5. 거래 검색(search) 검증

### 5-1. 타입 필터

- **실행**:
  ```bash
  python -m budget_app search --type expense
  ```
- **기대 출력**: expense 거래만 최신순으로 표시.

### 5-2. 날짜 범위 + 카테고리

- **실행**:
  ```bash
  python -m budget_app search --from 2024-01-14 --to 2024-01-15 --category food
  ```
- **기대 출력**: food + 해당 날짜 범위 거래만 표시.

---

## 6. 예산 설정(budget) + 월별 요약(summary) 검증

### 6-1. budget set

- **실행**:
  ```bash
  python -m budget_app budget set --month 2024-01 --amount 500000
  ```
- **기대 출력**:
  ```
  [저장 완료] 2024-01 예산 500000원
  ```

### 6-2. summary

- **실행**:
  ```bash
  python -m budget_app summary --month 2024-01 --top 3
  ```
- **기대 출력** (값은 추가한 데이터에 따라 다름):
  ```
  총 수입: 3000000원
  총 지출: 35000원
  잔액: 2965000원
  예산: 500000원 (사용률 7.0%)
  
  지출 TOP 3
  1) food 15000원
  2) transport 20000원
  ```
- **확인 포인트**: 예산 사용률 출력 여부, 초과 시 `[경고]` 출력 여부 확인.

### 6-3. budget amount 음수 오류

- **실행**:
  ```bash
  python -m budget_app budget set --month 2024-01 --amount -1
  ```
- **기대 출력**:
  ```
  [오류] 금액은 양수 정수여야 합니다.
  [힌트] 예: 15000
  ```

---

## 7. 거래 수정(update) / 삭제(delete) 검증

### 7-1. update (옵션 방식 안 A)

- **실행**:
  ```bash
  python -m budget_app update --id TX-000001 --memo "수정된 점심"
  ```
- **기대 출력**: `[수정 완료] id=TX-000001`
- **확인**: `python -m budget_app list --limit 1` 으로 memo가 변경됐는지 확인.

### 7-2. delete — 없는 id

- **실행**:
  ```bash
  python -m budget_app delete --id TX-999999
  ```
- **기대 출력**:
  ```
  [오류] id=TX-999999를 찾을 수 없습니다.
  ```

---

## 8. 카테고리 관리(category) 검증

### 8-1. 사용 중 카테고리 삭제 차단

- **실행**:
  ```bash
  python -m budget_app category remove
  # 삭제할 카테고리명: food
  ```
- **기대 출력**:
  ```
  [오류] 카테고리 'food'은(는) 사용 중입니다.
  [힌트] 해당 카테고리의 거래를 먼저 삭제하거나 수정하세요.
  ```

### 8-2. 미사용 카테고리 삭제

- **실행**:
  ```bash
  python -m budget_app category add
  # 카테고리명: test_cat
  python -m budget_app category remove
  # 삭제할 카테고리명: test_cat
  ```
- **기대 출력**: `[완료] category=test_cat 삭제됨`

---

## 9. CSV export/import 검증

### 9-1. export

- **실행**:
  ```bash
  python -m budget_app export --out myexport.csv --month 2024-01
  ```
- **기대 출력**: `[완료] myexport.csv (N records)`
- **확인**: `cat myexport.csv` — 첫 줄이 `date,type,category,amount,memo,tags` 헤더, UTF-8 인코딩.

### 9-2. import

- **실행**:
  ```bash
  python -m budget_app import --from myexport.csv
  ```
- **기대 출력**: `[완료] imported=N, skipped=0`

### 9-3. export 조건 없음 오류

- **실행**:
  ```bash
  python -m budget_app export --out out.csv
  ```
- **기대 출력**:
  ```
  [오류] --month 또는 --from/--to 중 하나 이상 필요합니다.
  ```

---

## 10. 제약 충족 수동 확인

| PDF 제약 | 확인 방법 | 위치 |
|----------|-----------|------|
| 외부 라이브러리 미사용 | `grep -r "^import\|^from" budget_app/` — json/csv/sys/functools/datetime/pathlib만 | 전체 |
| JSONL 저장 | `cat data/transactions.jsonl` — 줄당 JSON 1건 | - |
| 3개 파일 분리 | `ls data/` — transactions.jsonl, categories.jsonl, budgets.jsonl | - |
| 기본 저장 폴더 ./data | `__main__.py:DEFAULT_DATA_DIR = "./data"` | `__main__.py` 상단 |
| 오류 exit code 1 | 오류 명령 후 `echo $?` → 1 | - |
| 스택트레이스 없음 | 오류 발생 시 `Traceback` 문자열 미출력 | - |
| 옵션 `--` 형식 | `python -m budget_app list --help` — 모든 옵션 `--` 시작 | - |
| update 안 A 고정 | `__main__.py` update_p 참조 | `__main__.py:update_p` |

---
