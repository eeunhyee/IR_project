[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/Ba_cGVYq)
# Scientific Knowledge QA (IR) — Dual-Query Hybrid Retrieval + Weighted RRF + Reranking

과학 지식 질의응답(Scientific Knowledge Question Answering) 태스크에서  
**대화 히스토리 기반 질의 정제 + 하이브리드 검색 + 가중 RRF 융합 + Cross-Encoder 리랭킹**으로
정확한 Top-k 근거 문서를 찾아 답변 생성 성능을 끌어올리는 파이프라인을 구축했습니다.  
(Upstage AI Lab / Fastcampus IR 경진대회 발표 프로젝트)

---

## 1) Competition / Task

- **Task**: 질문 + 이전 대화 히스토리를 입력으로 받아, 검색엔진에서 참고 문서를 추출하고 이를 활용해 답변 생성 :contentReference[oaicite:3]{index=3}  
- **기간**: 2025.12.18 10:00 ~ 2025.12.29 19:00 :contentReference[oaicite:4]{index=4}

---

## 2) Team

| Role | Name | Background |
|---|---|---|
| Team Lead | 황은혜 | 소프트웨어 |
| Member | 최보경 | AI/무역 |
| Member | 이동건 | 데이터사이언스/도시공학 |
| Member | 김수현 | AI/타전공 |
| Member | 오정택 | 프리랜서 |

:contentReference[oaicite:5]{index=5}

---

## 3) Data Overview & Key Insights (EDA)

- 평가 데이터: **총 220개 질의**
  - 비과학적 일상 대화(casual) 약 20개
  - 멀티턴 질의(이전 발화 의존) 약 20개 :contentReference[oaicite:6]{index=6}
- 문서 데이터: 질문이 포함되지 않은 **순수 과학 지식 문서 약 4,200개** :contentReference[oaicite:7]{index=7}
- 핵심 인사이트: **Retrieval 성능이 “질의 정제 품질”에 크게 의존** :contentReference[oaicite:8]{index=8}

문제/노이즈 패턴
- **비과학 질의**: retrieval 자체가 의미 없으므로 rule-based 필터링 필요 :contentReference[oaicite:9]{index=9}
- **말투/종결어미 노이즈**: 문서에 거의 등장하지 않아 sparse/dense 매칭 확률 저하 → normalize 필요 :contentReference[oaicite:10]{index=10}
- **멀티턴 질의**: 지시대명사/생략으로 단독 문장 의미 불명확 → Standalone Query 생성 필요 :contentReference[oaicite:11]{index=11}
- **표기 불일치(약어/외래어/한글-영문 혼재)**: 단순 토큰 매칭에서 손실 → canonicalization(치환 사전) 필요 :contentReference[oaicite:12]{index=12}

---

## 4) Method Overview (Pipeline)

### 4.1 Preprocessing / Query Engineering
1) **Chit-Chat Filter**: 과학과 무관한 일상 대화를 rule-based로 필터링 :contentReference[oaicite:13]{index=13}  
2) **Canonicalization**: 예) `네트웍→네트워크`, `파이썬→Python` 등 표준화로 검색 누락 방지 :contentReference[oaicite:14]{index=14}  
3) **Standalone Query 생성 (LLM)**: 멀티턴 맥락을 반영해 불완전 질문을 완전한 문장으로 복원 + 검색 친화 키워드 중심 추출 :contentReference[oaicite:15]{index=15}  

### 4.2 Dual-Query Hybrid Retrieval (Sparse + Dense)
- **Sparse (Elasticsearch)**  
  - LLM(gpt-4o-mini)이 정제한 *키워드 쿼리* 사용 :contentReference[oaicite:16]{index=16}  
  - 과학 용어 정확 매칭을 위해 **Title 필드 가중치 3배** :contentReference[oaicite:17]{index=17}  
- **Dense (Embedding: intfloat/multilingual-e5-large)**  
  - 사용자 **원본 질문(Raw Query)** 사용 :contentReference[oaicite:18]{index=18}  
  - 의미적 맥락 기반으로 키워드 매칭 실패 문서 보완 :contentReference[oaicite:19]{index=19}  

### 4.3 Weighted RRF (Strategic Fusion)
- **가중치 5:1 (Sparse:Dense)**  
  - 키워드 정확히 일치하는 문서를 상위권에 강하게 올리도록 설계 :contentReference[oaicite:20]{index=20}  
- **RRF_K=40 Rank Tuning**으로 상위권 변별력 강화 :contentReference[oaicite:21]{index=21}  

### 4.4 Cross-Encoder Reranking
- Model: **BAAI/bge-reranker-v2-m3** (한국어 과학 용어 처리 강점) :contentReference[oaicite:22]{index=22}  
- High-Recall: 초기 검색에서 **Top-800 후보** 확보 :contentReference[oaicite:23]{index=23}  
- Speed: 리랭킹 후보 **300개**, 입력 **512 토큰 제한**, **Batch=32**로 최적화 :contentReference[oaicite:24]{index=24}  
- 최종적으로 리랭커가 (질문, 문서) 쌍을 정밀 분석해 **Top-3**를 추출 :contentReference[oaicite:25]{index=25}  

---

## 5) Experiments & What Worked

### 5.1 Embedding Model Sweep
실험 후보:
- snunlp/KR-SBERT-V40K-klueNLI-augSTS
- intfloat/multilingual-e5-large-instruct
- BAAI/bge-m3
- paraphrase-multilingual-mpnet-base-v2
- distiluse-base-multilingual-cased
- intfloat/multilingual-e5-large
- intfloat/multilingual-e5-base :contentReference[oaicite:26]{index=26}  

### 5.2 Score Lift Drivers (Summary)
- **Dense Recall 상승**: 멀티링구얼+Retrieval 특화 구조 및 1024 차원 이점으로 문맥 유사도 확보 :contentReference[oaicite:27]{index=27}  
- **Reranker 도입**: “유사하지만 틀린 문서”를 효과적으로 걸러 Precision 상승 :contentReference[oaicite:28]{index=28}  
- **Weighted RRF**: Dense 노이즈로 키워드 매칭이 희석되는 문제를 5:1 가중으로 해결 :contentReference[oaicite:29]{index=29}  

### 5.3 Retrospective (Next)
- 문서 전체가 아니라 **Chunk 인덱스 기반**으로 검색/리랭크 단위를 가져가면, 특정 문장/단락에 정답 근거가 있을 때 더 잘 잡을 가능성 :contentReference[oaicite:30]{index=30}  
