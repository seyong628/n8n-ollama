# n8n + Ollama 로컬 AI 챗봇

> Docker로 구동되는 n8n 워크플로우 자동화 도구와 Ollama 로컬 LLM을 연동한 프라이버시 100% 로컬 AI 챗봇

---

## 아키텍처

```
┌─────────────────────────────────────────────────────┐
│                    Windows 11 (Host)                 │
│                                                      │
│   ┌─────────────────┐      ┌─────────────────────┐  │
│   │   WSL2 Ubuntu   │      │  Ollama (Windows앱)  │  │
│   │                 │      │                     │  │
│   │  ┌───────────┐  │      │  ┌───────────────┐  │  │
│   │  │  Docker   │  │      │  │ llama3:latest │  │  │
│   │  │           │  │      │  │  (8B, Q4_0)   │  │  │
│   │  │  ┌─────┐  │  │      │  └───────────────┘  │  │
│   │  │  │ n8n │──┼──┼──────┼──▶ localhost:11434  │  │
│   │  │  │:5679│  │  │      │                     │  │
│   │  │  └─────┘  │  │      │   RTX 5070 (12GB)   │  │
│   │  └───────────┘  │      └─────────────────────┘  │
│   └─────────────────┘                               │
└─────────────────────────────────────────────────────┘
```

## n8n 워크플로우 구조

```
┌──────────────────────────┐
│  When chat message       │  ← 채팅 입력 수신
│  received                │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│       AI Agent           │  ← LLM 추론 처리
└────────────┬─────────────┘
             │ Chat Model 슬롯
             ▼
┌──────────────────────────┐
│   Ollama Chat Model      │  ← host.docker.internal:11434
│   (llama3:latest)        │     으로 Ollama API 호출
└──────────────────────────┘
```

---

## 환경

| 항목 | 내용 |
|------|------|
| OS | Windows 11 + WSL2 (Ubuntu 24.04) |
| GPU | NVIDIA RTX 5070 (12GB VRAM) |
| Docker | Docker Desktop 29.3.1 (WSL2 백엔드) |
| n8n | v2.14.2 |
| Ollama | Windows 네이티브 앱 |
| 모델 | llama3:latest (8B, Q4_0, 4.6GB) |

---

## 사전 준비

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) 설치 (WSL2 Integration 활성화 필요)
- [Ollama](https://ollama.com/) Windows 설치 후 실행
- Ollama 모델 pull

```bash
ollama pull llama3
```

---

## 실행 방법

### 1. 저장소 클론

```bash
git clone https://github.com/seyong628/n8n-ollama.git
cd n8n-ollama
```

### 2. n8n 실행

```bash
docker compose up -d
```

### 3. n8n 접속

브라우저에서 `http://localhost:5679` 접속

### 4. 동작 확인

```bash
# Ollama 상태 확인
curl http://localhost:11434/api/tags

# n8n 컨테이너 로그 확인
docker logs n8n -f
```

---

## n8n 워크플로우 설정

### Ollama Credential 생성

1. 좌측 메뉴 **Credentials** → **+ Add Credential**
2. `Ollama API` 선택
3. **Base URL**: `http://host.docker.internal:11434`
4. **Save**

### 워크플로우 구성

1. **Workflows** → **+ New Workflow**
2. 노드 추가:
   - `When chat message received`
   - `AI Agent` (위 노드에 연결)
   - `Ollama Chat Model` (AI Agent의 Chat Model 슬롯에 연결)
     - Credential: 위에서 만든 Ollama API
     - Model: `llama3:latest`
3. **Publish** → **Open Chat** 으로 테스트

---

## 핵심 포인트

### Docker → Ollama 통신
Docker 컨테이너 내부에서 Windows 호스트의 Ollama에 접근할 때
`localhost` 대신 `host.docker.internal` 을 사용합니다.

```yaml
# docker-compose.yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

### 포트 설정
n8n이 기존 5678 포트와 충돌하지 않도록 5679로 변경하고,
CORS 오류 방지를 위해 `N8N_EDITOR_BASE_URL`을 외부 포트와 일치시킵니다.

```yaml
ports:
  - "5679:5678"
environment:
  - N8N_EDITOR_BASE_URL=http://localhost:5679
  - WEBHOOK_URL=http://localhost:5679/
```

---

## 파일 구조

```
n8n-ollama/
├── docker-compose.yaml   # n8n 컨테이너 설정
└── .gitignore
```

---

## 트러블슈팅

| 문제 | 원인 | 해결 |
|------|------|------|
| 포트 충돌 | 5678 이미 사용 중 | `5679:5678` 로 변경 |
| CORS 오류 | 포트 불일치 | `N8N_EDITOR_BASE_URL` 설정 |
| Ollama 연결 실패 | `localhost` 사용 | `host.docker.internal` 사용 |
| Credential 오류 | 노드에 미연결 | 드롭다운에서 직접 선택 |
