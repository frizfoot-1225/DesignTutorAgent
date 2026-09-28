# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# 메모리 반도체 설계 온보딩 튜터 Agent

## 프로젝트 개요
메모리(DRAM 중심, NAND는 기초·비교 수준) 설계 부서의 신입·전배·비전공 엔지니어를 위한 AI 교육 에이전트.
사내 과제 시연용 프로토타입이지만, 나중에 사내 LLM과 사내 문서로 바꿀 수 있는 구조로 만든다.

목표: 설계 Flow 전체(스펙 → 아키텍처 → 회로 → 시뮬레이션 → 레이아웃 → 검증 → 테이프아웃 이후)를 쉽게 배우고,
실무 문서를 읽고 설계 리뷰에 참여할 수 있는 수준까지 끌어올린다.

원본 제안서는 `docs/PROPOSAL.md`에 있다. 기능 상세(F1~F8), 커리큘럼(Level 1~4), 코퍼스 작성 지침, 시연 시나리오는 그 문서가 기준이다.

## 진행 방식 (반드시 지킬 것)
- Phase 단위로 진행한다. **Phase 하나가 끝나면 멈추고 아래 형식으로 보고한 뒤, 승인 없이 다음 Phase로 넘어가지 않는다.**
  1. 구현한 내용 요약 2. 실행 방법(명령어) 3. 테스트 결과 4. 미해결 이슈와 판단이 필요한 사항 5. 다음 Phase에서 할 일
- 불명확한 부분은 추측하지 말고 질문한다.
- 기능 하나가 끝날 때마다 git commit. 파일 하나에 너무 많은 기능을 넣지 않는다.
- Phase가 끝나면 이 파일의 "진행 현황"을 갱신한다.

## 진행 현황
| Phase | 내용 | 상태 |
|---|---|---|
| 0 | CLAUDE.md, git init | **완료 (2026-09-28)** |
| 1 | 뼈대·환경: requirements, config.yaml, .env.example, NOTE_API_KEY.md, `llm.check`, LLM 추상화, 기본 채팅 | 대기 |
| 2 | 코퍼스 작성, RAG 인덱싱, F1 개념 Q&A, F2 Flow 내비게이터 | 대기 |
| 3 | SQLite, F3 퀴즈·간격 반복, F6 로드맵(진단 고정 문항) | 대기 |
| 4 | F4 실습 튜터: iverilog 트랙 먼저, 그다음 ngspice 트랙. 샌드박스 | 대기 |
| 5 | F5 실무 적응 트랙 4모드 | 대기 |
| 6 | F7 로그 해석, F8 대시보드·멘토 큐, 시연 시나리오 점검, README | 대기 |

## 환경 (2026-09-28 확인)
- Windows 10, Python 3.12.10. 설치됨: streamlit, chromadb, pydantic v2, pyyaml, openai SDK, python-dotenv, pypdf, pandas.
- 미설치(pip): pytest, plotly, python-docx, sentence-transformers. Phase 1의 `requirements.txt`에서 해결한다. sentence-transformers는 무거우므로 `requirements-local-embed.txt`로 분리.
- **미설치(외부 프로그램): ngspice, iverilog/vvp.** 사용자가 직접 설치한다. `sim/`은 미설치 시 앱을 죽이지 않고 설치 안내를 표시하고, 시뮬레이터 의존 테스트는 `skipif`로 건너뛴다.
- 상위 폴더 `C:\ds260928\CLAUDE.md`도 함께 로드된다. 그 규칙(stdlib 전용, SDK 대신 `urllib`, 테스트 없음 등)은 형제 앱 `rag-lab/`·`agents-lab/`용이며 **이 프로젝트에는 적용하지 않는다.** 단, `rag-lab/lab/`의 하이브리드 검색(벡터+BM25)과 인용문을 청크 원문에서 그대로 찾아 검증하는 `answer.py:locate()`는 Phase 2에서 참고할 만하다. 코드는 공유하지 않고 필요한 부분만 옮겨 온다.

## 실행 (Phase 1 이후 실제 명령으로 갱신)
```
pip install -r requirements.txt
copy .env.example .env               # 키 입력. 자세한 절차는 NOTE_API_KEY.md
python -m llm.check                  # 키·엔드포인트·선택된 모델·임베딩 1회 점검
python -m rag.index --rebuild        # docs_corpus/ 전체 재인덱싱 (--add <path>: 파일 하나 증분 추가)
streamlit run app.py
pytest                               # 전체
pytest tests/test_quiz.py::test_name # 단일 테스트
```

## 시스템 구성과 설계 결정

### LLM·모델 선택 (`llm/`)
- `llm/base.py`의 Provider 인터페이스: `chat()`, `chat_json(schema: type[BaseModel])`, `embed()`. 구현은 `openai_provider.py`(OpenAI SDK, `base_url` 교체로 사내 엔드포인트 대응)와 테스트용 `fake.py`(결정적 응답).
- **모델은 `.env` 한 파일에서 결정한다.** `OPENAI_API_KEY`, `OPENAI_BASE_URL`(비우면 공식), `OPENAI_MODEL`(비우면 자동 선택).
  자동 선택: 시작 시 `/v1/models`를 한 번 조회해 `config.yaml`의 선호 순서에서 실제 사용 가능한 첫 모델을 고른다. 키·엔드포인트만 바꿔도 그 계정에서 쓸 수 있는 모델로 알아서 바뀐다. 선택된 모델은 사이드바와 `llm.check` 출력에 표시한다.
- 키가 없거나 잘못되면 앱이 죽지 않고 화면에 설정 안내를 표시한다.
- 임베딩 기본은 OpenAI, 옵션으로 로컬 임베딩(sentence-transformers). `config.yaml`의 `embedding.provider`로 교체.

### 공용 자원 (`core/`)
- `core/services.py`: DB 연결, LLM provider, ChromaDB 클라이언트를 `@st.cache_resource`로 한 번만 생성. 페이지는 `get_db()`, `get_llm()`, `get_chroma()`만 쓰고 직접 생성하지 않는다. 실제 생성 로직은 각 모듈에 두고 `services.py`는 캐시 래퍼만 담당해 테스트에서 Streamlit 없이도 쓸 수 있게 한다.
- `core/session.py`: 사이드바 이름 입력 → `repo.get_or_create_learner(name)` → `st.session_state["learner_id"]`. 이름이 비어 있으면 기록이 필요한 페이지는 입력 안내만 표시.

### 프롬프트와 스키마
- 프롬프트는 코드에 쓰지 않고 `prompts/*.txt`에 둔다. `prompts/loader.py`가 `{변수}`를 치환하며 누락 변수는 즉시 오류로 낸다(빈 문자열이 조용히 들어가 답변이 틀어지는 것을 막기 위해).
- `chat_json`에 넘기는 pydantic 모델은 모두 `tutor/schemas.py`에 모은다(QAAnswer, QuizSet, CaseEvaluation, ReviewQuestions, SpecEvaluation, ReportFeedback, LogExplanation, LabHint 등). 검증 실패 시 config에 정한 횟수만큼 재생성.

### RAG (`rag/`, `docs_corpus/`)
- 로더: Markdown(frontmatter 필수: `title, level, concepts, flow_stage`), PDF, DOCX. ChromaDB 로컬, frontmatter로 메타데이터 필터.
- 코퍼스는 공개 지식 수준으로만 직접 작성한다. 확신 없는 수치는 넣지 않는다. 타이밍 파라미터는 정의·결정 요인·회로 연결 중심으로 쓰고 구체 ns 값은 "규격 참조"로 둔다. JEDEC 표는 원문을 복사하지 않고 형식만 흉내 낸 자체 제작 표를 쓴다.
- Q&A 답변 형식은 고정(한 줄 정의 → 비유/그림 → 회로·동작 예시 → 관련 Flow 단계 → 출처). LLM에게 JSON으로 받아 UI가 조립하고, **출처는 검색 결과 frontmatter에서 코드로 채운다**(LLM이 지어내지 못하게).
- 검색 유사도가 `config.yaml` 임계값에 못 미치면 답하지 않고 "문서 근거 없음 / 멘토 확인 필요"로 표시하고 멘토 큐에 저장한다. 임계값은 Phase 2에서 실제 코퍼스로 측정해 정하고 근거를 주석에 남긴다.
- 멘토 답변은 `docs_corpus/mentor_answers/`에 저장되고 `rag.index`의 증분 추가 함수로 즉시 인덱싱된다.

### 결정적 채점 원칙
LLM은 설명·힌트·피드백만 맡고, 채점과 구조는 코드/데이터가 맡는다.
- 퀴즈: LLM이 생성 → pydantic 검증 → 코드가 채점. 오답 개념은 SQLite에 저장하고 1일·3일·7일 뒤 재출제.
- 로드맵: 진단 10문항은 `curriculum/diagnostic.yaml` **고정 문항**(LLM 미사용). 직무별 6주 템플릿(`curriculum/template_*.yaml`)에서 약한 개념을 앞 주차로 끌어올린다. LLM은 설명 문장만 만든다.
- 실습: iverilog 채점은 테스트벤치 결과로만 한다. ngspice 마진 수치는 코드로 계산하고 LLM은 해석만.
- Flow 7단계, 파라미터-회로 매핑 표, 커리큘럼은 코드가 아니라 YAML 데이터로 둔다.

### 실습 샌드박스 (`sim/`)
- 공통(`sandbox.py`): 임시 디렉터리, 실행 10초 제한, 출력 크기 제한, `$system`·파일 시스템 접근 구문 실행 전 차단, 설치 여부 자동 확인.
- 과제는 `sim/assignments/{verilog,spice}/<id>/`에 폴더 단위로 두고, 각 폴더의 `meta.yaml`(`id, title, track, difficulty, concepts, flow_stage, hints[3단계], entry, params`)을 `sim/catalog.py`가 스캔해 Lab 페이지 목록을 자동 생성한다. 과제 추가 = 폴더 추가.
- 힌트 3단계: 1단계 문제 위치만 → 2단계 원인 설명 → 3단계 수정 예시. 정답을 바로 주지 않는다.
- Verilog 과제: 리프레시 카운터, 커맨드 디코더(ACT/RD/WR/PRE), 해밍 ECC. SPICE 과제: 비트라인 센싱(차지 셰어링), 센스앰프, 프리차지.

### 학습자 DB (`db/`)
- SQLite. `schema.sql` + `repo.py`. 테이블: learners(name UNIQUE), 진도, 퀴즈 이력, 취약 개념(복습 예정일 포함), 실습 이력, 멘토 질문. 모든 이력은 `learner_id` FK.

## 디렉터리 구조
```
aiagent/
├── app.py                 # 진입점: 설정/키 확인, 사이드바 이름 입력
├── pages/                 # 1_QA … 8_Dashboard
├── config.yaml            # 모델 선호 순서, 임베딩, 임계값, 시뮬레이터 제한, 경로
├── core/                  # services.py(cache_resource), session.py(학습자)
├── prompts/               # *.txt 프롬프트 + loader.py
├── llm/                   # base.py, openai_provider.py, fake.py, check.py
├── rag/                   # loaders.py, index.py(CLI 포함), search.py
├── tutor/                 # schemas.py, qa.py, flow.py, quiz.py, roadmap.py, practice.py, logs.py
├── sim/                   # sandbox.py, verilog.py, spice.py, catalog.py, assignments/{verilog,spice}/
├── db/                    # schema.sql, repo.py
├── docs_corpus/           # glossary/ flow/ timing/ cases/ faq/ errors/ mentor_answers/
├── curriculum/            # diagnostic.yaml, template_*.yaml
├── docs/                  # PROPOSAL.md (원본 제안서)
├── tests/
└── requirements.txt, .env.example, .gitignore, NOTE_API_KEY.md, README.md
```

## 원칙
- **정확성 우선**: 교육용이므로 틀린 설명이 가장 치명적. 코퍼스 근거가 없으면 답하지 않는다.
- **보안**: 실제 설계 데이터·공정 파라미터·사내 대외비는 코퍼스에 넣지 않는다. 비밀정보를 코드에 하드코딩하지 않는다. `.env`, `chroma_db/`, SQLite 파일은 `.gitignore`. LLM 호출 외에 사용자 입력을 외부로 보내는 코드를 넣지 않는다.
- **교육적 설계**: 정답보다 힌트로 사고를 유도한다.
- **언어**: UI·설명·코드 주석은 한국어, 기술 용어는 영문 병기. 예: 센스앰프(sense amplifier), 셋업 타임(setup time).
- **교체 가능성**: LLM, 임베딩, 코퍼스는 설정만 바꿔 교체할 수 있어야 한다.

## 시연 시나리오 (최종 목표)
1. Flow 내비게이터에서 "회로 설계" 클릭 → 코어/주변/디지털 블록 설명과 센스앰프 문서 링크
2. Q&A "tRCD가 뭐고 어디서 결정돼요?" → 정의, 비유, 결정 요인(워드라인 RC, 차지 셰어링, 센스앰프 속도), 출처
3. 퀴즈 3문제 → 틀린 개념이 복습 목록에 등록
4. ngspice 실습: 셀 캡을 줄이면 센싱 마진 변화를 파형 차트로 확인, Agent 해석
5. 케이스 스터디 "특정 뱅크에서만 읽기 페일" → 원인 후보 작성 → 모범 답안 비교
6. 대시보드에서 진도·취약 개념, 멘토 큐 확인
