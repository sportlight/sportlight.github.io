---
title: "[Side Project] 데이터 유출 없는 온프레미스 AI 에이전트 구축 (1) - FastAPI와 Vector DB 연동"
date: 2025-12-13 13:00:00 +0900
categories: [AI-Engineering, Backend]
tags: [python, fastapi, rag, vectordb, chromadb, langchain, on-premise, llm, llama3]
mermaid: true
---

## 1. 프로젝트 개요 (Overview)

SaaS형 AI(ChatGPT, Claude)는 강력하지만, 기업의 민감 데이터나 개인정보를 외부 서버로 전송해야 한다는 보안 이슈가 존재한다.
이번 사이드 프로젝트의 목표는 **"인터넷 연결 없이 로컬 환경(On-Premise)에서 동작하는 나만의 AI 에이전트"**를 구축하는 것이다.

흔히 데이터를 넣어 학습시키는 것을 파인튜닝(Fine-tuning)이라 부르지만, 이번 프로젝트에서는 실시간 데이터 업데이트가 용이하고 비용이 저렴한 **RAG(Retrieval-Augmented Generation, 검색 증강 생성)** 방식을 채택한다. 즉, 벡터 DB(Vector DB)를 두뇌의 장기 기억장치로 활용하는 방식이다.

### 🛠 Tech Stack
* **Language:** Python 3.10+
* **Server Framework:** FastAPI (비동기 처리 및 API 문서화 용이)
* **Orchestration:** LangChain (LLM과 DB 연결 파이프라인)
* **Vector DB:** ChromaDB (로컬 파일 기반, 가볍고 빠름)
* **LLM Engine:** Llama.cpp (CPU/GPU 겸용 경량화 모델 구동)
* **Model:** Llama-3-8B-Quantized (GGUF 포맷 사용)

---

## 2. 아키텍처 설계 (Architecture)

사용자의 질문이 들어오면 벡터 DB에서 관련 지식을 검색(Retrieval)하고, 이를 LLM에 프롬프트로 주입하여 답변을 생성(Generation)하는 흐름이다.

```mermaid
flowchart LR
    User[Client] -->|POST /chat| API[FastAPI Server]
    
    subgraph On_Premise_Server
        direction TB
        API -->|1. Query Embedding| Embed[Local Embedding Model]
        Embed -->|2. Similarity Search| VDB[(Chroma Vector DB)]
        VDB -->|3. Retrieve Context| API
        
        API -->|4. Prompt + Context| LLM[Local LLM (Llama-3)]
        LLM -->|5. Generate Answer| API
    end
    
    API -->|6. JSON Response| User
```

---

## 3. 개발 환경 설정 (Setup)

외부 API를 전혀 사용하지 않으므로, 로컬에서 구동 가능한 라이브러리들을 설치해야 한다.

### 3.1. 가상환경 및 라이브러리 설치

```bash
# 1. 가상환경 생성 및 활성화
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 2. 필수 패키지 설치
# llama-cpp-python은 하드웨어 가속(Metal/CUDA) 지원을 위해 별도 빌드가 필요할 수 있음
pip install fastapi uvicorn \
            langchain langchain-community langchain-huggingface \
            chromadb \
            llama-cpp-python \
            python-multipart \
            pydantic
```

### 3.2. 모델 파일 준비 (GGUF)
로컬에서 돌리기 위해 양자화된(Quantized) 모델이 필요하다. HuggingFace에서 다운로드하여 프로젝트 폴더 내 `models/` 디렉토리에 위치시킨다.
* **추천 모델:** `Meta-Llama-3-8B-Instruct.Q4_K_M.gguf` (약 4GB~5GB)

---

## 4. 핵심 모듈 구현

### 4.1. 벡터 DB 매니저 (RAG Core)
문서(Text)를 벡터화하여 저장하고, 질문과 유사한 내용을 검색하는 핵심 모듈이다. 임베딩 모델 또한 외부 API가 아닌 로컬 모델(`all-MiniLM-L6-v2`)을 사용한다.

```python
# app/rag_core.py
import os
from langchain_community.vectorstores import Chroma
from langchain_huggingface import HuggingFaceEmbeddings
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.docstore.document import Document

class VectorDBManager:
    def __init__(self, db_path="./chroma_db"):
        # 1. 로컬 임베딩 모델 로드 (HuggingFace)
        # CPU 환경에서도 빠르고 준수한 성능을 보여주는 경량 모델 사용
        self.embedding_func = HuggingFaceEmbeddings(
            model_name="sentence-transformers/all-MiniLM-L6-v2",
            model_kwargs={'device': 'cpu'} # GPU가 있다면 'cuda'
        )
        self.db_path = db_path
        
        # 2. ChromaDB 로드 (Persist)
        # 디스크에 저장하여 서버 재시작 후에도 데이터 유지
        self.vectordb = Chroma(
            persist_directory=self.db_path,
            embedding_function=self.embedding_func
        )

    def add_documents(self, text_list: list[str]):
        """
        텍스트 데이터를 청크(Chunk)로 나누어 벡터 DB에 저장
        """
        text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=500,     # 500자 단위로 자름
            chunk_overlap=50    # 문맥 유지를 위해 50자 중복
        )
        docs = [Document(page_content=t) for t in text_list]
        splits = text_splitter.split_documents(docs)
        
        self.vectordb.add_documents(splits)
        print(f"✅ {len(splits)} chunks added to Vector DB.")

    def get_retriever(self):
        """LangChain Chain 연결을 위한 Retriever 반환"""
        # 유사도 점수 기반으로 상위 3개 문서 참조
        return self.vectordb.as_retriever(search_kwargs={"k": 3})
```

### 4.2. LLM 엔진 (Local Model Loader)
`llama-cpp-python`을 사용하여 GGUF 모델을 로드한다. 시스템 리소스에 따라 `n_gpu_layers` 등의 파라미터 튜닝이 필요하다.

```python
# app/llm_engine.py
import os
from langchain_community.llms import LlamaCpp
from langchain.callbacks.manager import CallbackManager
from langchain.callbacks.streaming_stdout import StreamingStdOutCallbackHandler

def load_llm(model_path="./models/llama-3-8b.gguf"):
    """
    로컬 Llama-3 모델 로드
    """
    if not os.path.exists(model_path):
        raise FileNotFoundError(f"Model not found at {model_path}")

    callback_manager = CallbackManager([StreamingStdOutCallbackHandler()])
    
    llm = LlamaCpp(
        model_path=model_path,
        temperature=0.1,    # RAG에서는 환각(Hallucination) 방지를 위해 낮게 설정
        n_ctx=4096,         # 컨텍스트 윈도우 (입력 + 출력 토큰 한계)
        max_tokens=1024,    # 답변 최대 길이
        n_gpu_layers=0,     # GPU 사용 시 레이어 수 조절 (Mac Metal: -1)
        callback_manager=callback_manager,
        verbose=True
    )
    return llm
```

### 4.3. FastAPI 메인 서버 (Main)
LangChain의 `RetrievalQA` 체인을 사용하여 DB 검색 결과와 LLM 추론을 연결한다.

```python
# main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from app.rag_core import VectorDBManager
from app.llm_engine import load_llm
from langchain.chains import RetrievalQA

app = FastAPI(title="On-Premise AI Agent Server")

# --- 전역 리소스 초기화 ---
print("Initializing Vector DB...")
vdb_manager = VectorDBManager()

print("Loading Local LLM...")
llm = load_llm()

# --- RAG 체인 생성 ---
# retriever가 관련 문서를 찾아오면, llm이 그것을 보고 답변을 생성함
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    retriever=vdb_manager.get_retriever(),
    return_source_documents=True # 답변의 출처 확인용
)

# --- DTO 정의 ---
class QueryRequest(BaseModel):
    query: str

class DocRequest(BaseModel):
    texts: list[str]

# --- API Endpoints ---

@app.get("/")
def health_check():
    return {"status": "ok", "mode": "on-premise"}

@app.post("/ingest")
async def ingest_knowledge(request: DocRequest):
    """
    [학습 단계] 텍스트 데이터를 벡터 DB에 주입 (Embedding & Indexing)
    """
    try:
        vdb_manager.add_documents(request.texts)
        return {"status": "success", "message": f"{len(request.texts)} documents ingested."}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/chat")
async def chat(request: QueryRequest):
    """
    [추론 단계] RAG 기반 질문 답변
    """
    try:
        # LangChain invoke
        result = qa_chain.invoke({"query": request.query})
        
        # 출처 문서 정리
        sources = [doc.page_content for doc in result['source_documents']]
        
        return {
            "query": request.query,
            "answer": result['result'],
            "context_used": sources
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## 5. 실행 및 테스트 (Testing)

서버를 실행하고, 모델이 내가 주입한 지식을 바탕으로 답변하는지 확인한다.

```bash
# 서버 실행
python main.py
```

### 5.1. 지식 주입 (Ingest)
일반적인 LLM은 알 수 없는, 나만의 '사이드 프로젝트' 정보를 주입해 본다.

```bash
curl -X POST "http://localhost:8000/ingest" \
     -H "Content-Type: application/json" \
     -d '{
           "texts": [
             "2025년 진행하는 사이드 프로젝트의 코드명은 Project Zero입니다.",
             "Project Zero의 핵심 목표는 온프레미스 환경에서 완벽한 데이터 보안을 구축하는 것입니다."
           ]
         }'
```

### 5.2. 질의응답 (Chat)
주입된 지식을 바탕으로 질문을 던진다.

```bash
curl -X POST "http://localhost:8000/chat" \
     -H "Content-Type: application/json" \
     -d '{"query": "Project Zero의 목표가 뭐야?"}'
```

**[예상 응답]**
```json
{
  "query": "Project Zero의 목표가 뭐야?",
  "answer": "Project Zero의 핵심 목표는 온프레미스 환경에서 완벽한 데이터 보안을 구축하는 것입니다.",
  "context_used": ["Project Zero의 핵심 목표는..."]
}
```

---

## 6. 결론 (Conclusion)

이로써 외부 인터넷 연결 없이 동작하는 **Private AI 서버**의 기초를 다졌다.
일반적인 파인튜닝(Fine-tuning)은 GPU 자원이 많이 들고 모델 재학습 시간이 오래 걸리지만, **Vector DB를 활용한 RAG 방식**은 문서를 넣는 즉시 AI가 지식을 습득한다는 장점이 있다.

### Next Steps
1.  **PDF/Word 파서 연동:** 단순 텍스트가 아닌 파일 업로드 기능 구현
2.  **GPU 가속 최적화:** `n_gpu_layers` 튜닝을 통한 응답 속도 개선
3.  **Dockerizing:** 배포 편의성을 위한 컨테이너 이미지 빌드
