# 로컬 RAG 구축 참고 원문

우리 팀은 Q&A 데이터베이스를 운영 중임
질문과 답변을 trigram 인덱스와 임베딩으로 모두 색인해 Postgres에 저장함
검색 시 pgvector와 trigram 검색을 함께 사용하고, 관련도 점수로 결과를 결합함

검색 단계에서는 CPU 친화적인 고효율 텍스트 임베딩 모델을 개발했음
MongoDB/mdbr-leaf-ir 모델로, 같은 크기 모델 중 리더보드 1위를 차지함
Snowflake/snowflake-arctic-embed-m-v1.5 모델과 호환 가능함
search-sensei 데모를 통해 semantic 검색 vs BM25 vs hybrid 비교 가능함
예를 들어, 임베딩 모델은 “j lo”가 “Jennifer Lopez”를 의미한다는 걸 인식함
또한 훈련 레시피를 공개했으며, 보통 수준의 하드웨어로도 쉽게 학습 가능함

이 모델의 임베딩 속도와 recall이 minish나 model2vec 같은 정적 워드 임베딩과 비교해 어떤지 궁금함
나는 2024년 4월부터 Meta-Llama-3-8B로 벡터를 생성했음
Python과 Transformers를 RTX-A6000에서 사용했는데, 빠르지만 소음과 발열이 심했음
이후 M1 Ultra로 옮기고 Apple의 MLX 라이브러리를 사용했는데, 속도는 비슷하면서 훨씬 조용함
Llama 모델은 4k 차원이라 fp16 기준 8KB/청크이며, 이를 numpy.save()로 SQLite의 BLOB 컬럼에 저장함
검색 시 SQLite에서 모든 벡터를 불러와 numpy.array로 만든 뒤 FAISS로 검색함
RTX6000의 faiss-gpu는 매우 빠르고, M1 Ultra의 faiss-cpu도 내 용도(하루 몇 쿼리)에는 충분히 빠름
500만 청크 기준 메모리 사용량은 약 40GB로, 두 장비 모두 여유롭게 처리 가능함

대부분의 복잡한 문서는 Markdown 파일임
tobi/qmd라는 간단한 CLI 도구를 추천함
예전에는 fzf 기반 워크플로를 썼지만, 이 도구는 더 나은 퍼지 검색을 제공함
코드 검색에는 사용하지 않음

소개를 보고 golang으로 작성된 grep 대체 도구일 줄 알았는데, Markdown 헤딩 가중치 같은 기능이 있을 거라 예상했음
코드 검색에는 벡터 데이터베이스를 쓰지 말라고 권함
임베딩은 느리고 코드에는 맞지 않음
BM25 + trigram 조합이 더 좋은 결과를 내며, 응답 속도도 빠름

Postgres에서도 하이브리드 검색이 가능함
plpgsql_bm25 프로젝트를 참고할 수 있음
BM25와 pgvector를 Reciprocal Rank Fusion으로 결합한 예시와 Jupyter 노트북도 포함됨
나도 동의함. 예전에 grep 대체 도구로 하이브리드 검색을 써봤는데, 지속적인 재색인이 번거로웠음
코드용이 아닌 모델을 쓰면 벡터 검색이 노이즈를 많이 유발함
지금은 gpt-oss 20B를 ripgrep과 함께 루프 돌리는 방식이 훨씬 빠르고 정확함
BM25와 벡터 검색을 함께 제공하는 간단한 서비스나 Docker 이미지를 아는 사람 있는지 궁금함
파일 경로나 시그니처에 적용했을 때 좋은 결과를 얻었음
BM25와 결합(fusion) 하면 더 향상됨
문서 검색용으로 RAG를 사용하는 것에 대한 의견이 궁금함
로컬 RAG 실험용으로 local-LLM-with-RAG를 만들었음
Ollama의 “nomic-embed-text”로 임베딩을 생성하고, LanceDB를 벡터 DB로 사용함
최근에는 “agentic RAG”로 업데이트했지만, 작은 프로젝트에는 과할 수도 있음

나도 비슷한 걸 하고 있음. lance-context를 사용 중이며, 훨씬 단순한 버전임
“RAG”가 무슨 뜻인지 설명해줘서 고마움. 이 스레드를 읽으며 헷갈렸음
fp16 벡터를 SQLite에 BLOB으로 저장하고, 필터링 후 메모리에 로드해 행렬-벡터 곱(matvec) 으로 유사도 계산함
numpy나 torch가 멀티스레딩/BLAS/GPU를 활용하면 매우 빠름
병목이 생기면 sqlite-vector로 마이그레이션할 예정임
날짜나 위치 같은 필터로 데이터가 많이 줄어들기 때문에 효율적임
백엔드는 교체 가능한 인터페이스 뒤에 감춰져 있음

내 문서의 95%가 작은 Markdown 파일이라 SQLite FTS5로 일반 텍스트 검색 인덱스를 사용함
이미 인덱스가 있어서 mastra agent에 바로 연결했음
각 파일에는 짧은 설명 필드가 있어, 키워드 검색 후 설명이 일치하면 전체 문서를 로드함
설정에 한 시간 정도 걸렸고, 매우 잘 작동함

사실 그게 바로 RAG(Retrieval-Augmented Generation) 임
임베딩 기반 검색이 더 흔하긴 하지만, 본질은 동일함
우리는 Postgres에 익숙해서 PGVector로 시작했음
나중에 프롬프트의 반정형 필드가 필요한 콘텐츠와 100% 일치한다는 걸 발견함
운영자가 입력과 문서 양쪽에 태그를 넣기 시작했기 때문임 (약 50개 문서 정도)
그래서 먼저 필드를 검색해 해당 파일을 프롬프트에 넣고, 그다음에 임베딩 검색을 수행함
결과적으로 85%의 경우 vectordb가 필요 없음

대부분의 vectordb는 해결책을 찾는 망치 같음
나는 llmemory를 만들어 로컬과 회사 앱 양쪽에서 사용 중임
PostgreSQL + pgvector 기반이며, 하이브리드 BM25, 멀티 쿼리 확장, reranking 기능을 포함함
처음 공개하는 거라 약간의 버그는 있을 수 있음
성능에는 꽤 만족함
