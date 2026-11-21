[README.md](https://github.com/user-attachments/files/35200107/README.md)
# term_project-2-
# AI 브랜드 아이덴티티 생성기

브랜드 브리프(JSON)를 입력하면 LLM API로 네이밍·슬로건·스토리·컬러 팔레트를, 이미지 생성 API로 로고 시안을 만들어 폴더에 저장하는 CLI 프로그램입니다.

## 실행 방법 (Python 3.10 이상)

```bash
pip install -r requirements.txt
export OPENAI_API_KEY=sk-...        # Windows: set OPENAI_API_KEY=sk-...
python brand_generator.py
```

```
🎨 AI 브랜드 아이덴티티 생성기

브리프 파일 경로를 입력하세요: brief.json
출력 폴더 경로를 입력하세요 (엔터 시 ./output):
[1/5] 브랜드 네이밍 생성 중...
  - 블루밍 (Blooming): 자연에서 피어나는 아름다움
...
✅ 완료! ./output/ 폴더를 확인하세요.
```

## 결과물

```
output/
├── brand_result.json   # 모든 텍스트 결과 (네이밍·슬로건·스토리·컬러·경쟁사 분석·오류 기록)
├── color_palette.png   # 컬러 팔레트 시각화 (matplotlib)
├── logo_01.png         # 로고 시안 1
└── logo_02.png         # 로고 시안 2
```

## 요구사항 대응표

| 요구사항 | 구현 |
|---|---|
| print/input 대화형 입력 | 브리프 경로(필수, 잘못되면 다시 입력), 출력 폴더(선택, 기본 `./output`) |
| 브리프 JSON | 필수 `industry`·`target`·`keywords` 검증, 선택 `tone`·`competitors`·`notes` 프롬프트에 반영 |
| 네이밍 3~5개 + 의미/유래 | 4개 요청, 3~5개가 아니면 실패 처리 |
| 슬로건 3개, 톤앤매너 반영 | `tone`을 프롬프트에 명시, 3개 미만이면 실패 처리 |
| 스토리 300자 내외, 배경·철학·비전 | origin/philosophy/vision 따로 받고 story로 합침. 250~350자를 벗어나면 1회 재생성 |
| 메인 1 + 서브 2~3 HEX | HEX 형식 검증·정규화 후 matplotlib으로 `color_palette.png` 저장 |
| 로고 시안 2~3개 PNG | 이미지 API로 2개 생성, 받은 이미지는 PNG로 변환해 `logo_01.png`, `logo_02.png` 저장 |
| 결과 저장 | 텍스트 → `brand_result.json`, 이미지 → 개별 PNG |
| API 실패 시 계속 진행 | 단계마다 예외를 잡아 메시지 출력 후 다음 단계 진행, `errors`에 기록 |
| API 키 없음/잘못됨 안내 | 없음·형식 오류 → 설정 방법 안내 후 종료. 401 → 안내 출력, 남은 API 단계는 건너뛰고 결과 저장 |
| API 키 관리 | 코드에 키 없음. 환경 변수 → `.env` → `config.json` 순서로 읽음 |
| 보너스: 다국어 네이밍 | 각 후보에 한글명 + 영문명 동시 생성 |
| 보너스: 경쟁사 분석 | `competitors`가 있으면 경쟁사별 포지셔닝·강약점 + 차별화 포인트 생성 |

## 파이프라인

```
브리프 로드·검증
 → [1/5] 네이밍 ─ 첫 후보를 대표 브랜드명으로 선정
 → [2/5] 슬로건 ─┐
 → [3/5] 스토리 ─┼─ 대표 브랜드명 + 브리프를 프롬프트에 사용
 → [4/5] 컬러   ─┘ → HEX 검증 → matplotlib PNG
 → [5/5] 로고 ─ 브랜드명 + 키워드 + 톤 + 컬러 팔레트로 이미지 프롬프트 작성
 → [보너스] 경쟁사 분석
 → brand_result.json 저장
```

LLM 호출에는 `response_format: {"type": "json_object"}`를 지정해 응답을 바로 dict로 파싱하고, 개수·형식을 코드로 다시 검증합니다.

## API 오류와 대응

| 상황 | 대응 |
|---|---|
| 키 없음 / `sk-` 형식 아님 | 설정 방법 안내 후 종료 |
| 401 잘못된 키 | 안내 출력, 남은 API 단계는 즉시 건너뛰고 결과 저장 |
| 429 속도 제한, 5xx 서버 오류, 타임아웃, 네트워크 오류 | 2초 → 4초 간격으로 최대 3회 시도, 그래도 실패하면 다음 단계 |
| 429 크레딧 소진, 400·403·404 | 재시도하지 않고 다음 단계 (이미지는 `gpt-image-1` 모델로 한 번 더 시도) |
| 응답 JSON 오류, HEX 오류, 개수 부족 | 해당 단계 실패 처리 후 다음 단계 |
| 네이밍 실패 | 업종명을 임시 이름으로 사용해 이후 단계 진행 |

## 설정

- `OPENAI_MODEL`: 텍스트 모델 (기본 `gpt-4o-mini`)
- `OPENAI_IMAGE_MODEL`: 이미지 모델 (기본 `dall-e-3`)
- `.env` / `config.json`을 쓸 경우 `.gitignore`에 추가하세요.

## 테스트

```bash
python test_mock.py
```

API 응답만 가짜로 바꿔 정상 흐름, 단계 실패 후 계속 진행, 스토리 길이 재생성, 401, 키 없음, 필수 필드 누락을 검증합니다.
