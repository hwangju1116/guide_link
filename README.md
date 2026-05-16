# A2UI Agent & Atlassian MCP Server Deployment Guide

이 실습 가이드는 **Gemini Enterprise Agent Platform**을 기반으로 두 개의 강력한 AI 에이전트를 배포하고 연동하는 전체 흐름을 제공합니다. 
첫 번째는 사용자와 제품 정보를 인터랙티브하게 주고받는 **A2UI 기반의 삼성 제품 스펙 비교 에이전트**이며, 두 번째는 업무 관리 및 지식 베이스인 Atlassian Jira/Confluence와 실시간으로 통신하는 **MCP(Model Context Protocol) 서버**입니다.

---

## 🚀 실습 아키텍처 및 개요
* **A2UI Agent**: Vertex AI Reasoning Engine(Agent Engine) 상에서 기동하며, GCS 버킷 및 Gemini Enterprise App과 연동하여 제품군 정보 서빙 및 스펙 비교 테이블 렌더링을 처리합니다.
* **Atlassian MCP Server**: Cloud Run 상에서 무상태(Stateless) 컨테이너로 동작하며, Jira API 및 Confluence API를 AI 모델이 표준 통신 규격(MCP)으로 호출할 수 있게 브릿지 역할을 담당합니다.

---

## 🛠️ 사전 준비 (Setup and Requirements)

### Qwiklabs 콘솔 및 Cloud Shell 활성화
Google Cloud 콘솔에 로그인한 뒤, 화면 우측 상단의 **Cloud Shell 활성화(Activate Cloud Shell)** 아이콘을 클릭하여 터미널 세션을 엽니다.

---

## 📍 Task 1. Google Cloud API 활성화 및 OAuth 클라이언트 생성 (공통 설정)

두 에이전트가 구글 클라우드 리소스 및 외부 Atlassian 시스템과 통신하기 위해 필수 API를 켜고 공통 인증 신분증(OAuth Credentials)을 발급받습니다.

1. Google Cloud 콘솔의 왼쪽 상단 **탐색 메뉴(☰) > APIs & Services > Library**로 이동합니다.
2. 아래 4개의 API를 각각 검색하여 **Enable(사용 설정)**을 클릭합니다:
   * **Vertex AI API**
   * **Cloud Storage API**
   * **Cloud Run API**
   * **Artifact Registry API**
3. **탐색 메뉴(☰) > APIs & Services > OAuth consent screen**으로 이동합니다.
4. User type에서 **External**을 선택하고 **Create**를 클릭합니다.
5. **App name**에 `Gemini Agents`를 입력하고 필수 지원 이메일 정보를 기입한 뒤 **Save and Continue**를 클릭하여 단계를 완료합니다.
6. 왼쪽 사이드바 메뉴에서 **Credentials**를 클릭합니다.
7. **Create Credentials > OAuth client ID**를 클릭합니다.
8. **Application type**에서 **Web application**을 선택하고 Name은 `agent-auth`로 지정합니다.
9. **Authorized redirect URIs (승인된 리디렉션 URI)** 섹션 하단의 **ADD URI**를 클릭하고 다음 URL을 정확히 추가합니다:
   ```text
   https://vertexaisearch.cloud.google.com/oauth-redirect
   ```
10. **Create**를 클릭하여 클라이언트를 생성합니다.
11. 화면 팝업창에 나타난 **Client ID**와 **Client Secret**을 메모장 등 안전한 곳에 복사해 둡니다. (다음 태스크들에서 공통 열쇠로 사용됩니다.)

---

## 📍 Task 2. A2UI 에이전트 배포 (Samsung Device Specs)

A2UI 기반의 스펙 비교 에이전트를 빌드하고 Vertex AI Reasoning Engine에 등록합니다.

1. 구글 클라우드 콘솔의 **탐색 메뉴(☰) > Gemini Enterprise Agent Platform > Workbench**를 클릭합니다.
2. 실습용으로 할당된 인스턴스 이름 옆의 **Open JupyterLab**을 클릭합니다. 새 브라우저 탭에서 개발 환경이 활성화됩니다.
3. JupyterLab Launcher 화면의 **Other** 섹션 아래에 있는 **Terminal**을 클릭하여 새 터미널 창을 엽니다.
4. 에이전트 저장소를 복제하고 패키지 관리 도구 `uv`를 설치하기 위해 다음 명령어를 차례로 실행합니다:
   ```bash
   git clone https://github.com/hwangju1116/a2ui-agent.git
   cd a2ui-agent
   pip install uv
   touch .env
   ```
5. 에이전트가 외부 인증 환경과 안전하게 매핑되도록 Vertex AI 내부에 **Authorization(권한) 리소스**를 생성해야 합니다. 
   아래 명령어 중 **CLIENT_ID**와 **CLIENT_SECRET**에 **Task 1**에서 발급받은 값을 입력한 후 터미널에 복사하여 통째로 실행하세요. (ID와 Secret의 대괄호 `[...]` 기호는 제거해야 합니다.)
   ```bash
   export PROJECT_ID=$(gcloud config get-value project)
   export AUTH_ID=a2ui-agent-auth
   export CLIENT_ID=[Task 1에서 복사한 Client ID]
   export CLIENT_SECRET=[Task 1에서 복사한 Client Secret]

   curl -X POST \
     -H "Authorization: Bearer $(gcloud auth print-access-token)" \
     -H "Content-Type: application/json" \
     -H "X-Goog-User-Project: ${PROJECT_ID}" \
     "https://global-discoveryengine.googleapis.com/v1alpha/projects/${PROJECT_ID}/locations/global/authorizations?authorizationId=${AUTH_ID}" \
     -d '{
       "name": "projects/'"${PROJECT_ID}"'/locations/global/authorizations/'"${AUTH_ID}"'",
       "serverSideOauth2": {
         "clientId": "'"${CLIENT_ID}"'",
         "clientSecret": "'"${CLIENT_SECRET}"'",
         "authorizationUri": "https://accounts.google.com/o/oauth2/v2/auth?client_id='"${CLIENT_ID}"'&redirect_uri=https%3A%2F%2Fvertexaisearch.cloud.google.com%2Foauth-redirect&scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloud-platform&response_type=code&access_type=offline&prompt=consent",
         "tokenUri": "https://oauth2.googleapis.com/token"
       }
     }'
   ```
6. JupyterLab 좌측의 파일 브라우저 패널에서 `a2ui-agent` 폴더 내에 생성된 **`.env`** 파일을 더블 클릭하여 편집기로 엽니다.
7. 파일 안에 다음 템플릿 내용을 붙여넣고 본인의 구글 클라우드 환경값에 맞춰 변경합니다:
   ```text
   PROJECT_ID=[본인의-프로젝트-ID]
   LOCATION=us-central1
   STORAGE_BUCKET=[본인의-프로젝트-ID]-a2ui-bucket
   GEMINI_ENTERPRISE_APP_ID=[본인의-Gemini-App-ID]
   AGENT_AUTHORIZATION=projects/[본인의-프로젝트-ID]/locations/global/authorizations/a2ui-agent-auth
   ```
   > **Note:** `GEMINI_ENTERPRISE_APP_ID` 자리에는 생성해 두신 Gemini Enterprise 앱 상세 화면 등에서 제공되는 고유 식별자 ID를 기입합니다.
8. 내용을 실제 값으로 다 변경했다면 파일을 반드시 **저장(Ctrl + S 또는 File > Save)**하고 편집 창을 닫습니다.
9. 터미널로 돌아와 다음 배포 자동화 명령어를 실행합니다:
   ```bash
   uv run deploy.py
   ```
10. 배포 프로세스가 모두 완료된 후 터미널 콘솔 로그의 마지막 부분에 표시되는 **`REASONING_ENGINE_ID`** 값을 복사하여 보관합니다.

---

## 📍 Task 3. Atlassian MCP 서버 빌드 및 Cloud Run 배포

Jira 및 Confluence API 인터페이스 역할을 할 MCP 서버를 도커 컨테이너화하여 Cloud Run 서비스로 배포합니다.

1. JupyterLab 터미널에서 상위(홈) 디렉토리로 복귀하고, 두 번째 MCP 리포지토리를 복제합니다:
   ```bash
   cd ~
   git clone https://github.com/hwangju1116/jira_confluence_mcp.git
   cd jira_confluence_mcp
   touch .env
   ```
2. 파일 브라우저에서 `jira_confluence_mcp` 폴더 안의 **`.env`** 파일을 열고 실제 Atlassian 계정 정보와 API 토큰을 기입합니다:
   ```text
   JIRA_URL=[본인의-지라-URL]
   JIRA_EMAIL=[본인의-지라-이메일]
   JIRA_API_TOKEN=[본인의-지라-API-토큰]
   ```
   > **Tip:** Atlassian API 토큰은 [Atlassian Account API Tokens](https://id.atlassian.com/manage-profile/security/api-tokens) 허브에서 발급받으실 수 있습니다. 작성을 마쳤다면 반드시 파일을 저장하고 닫아주세요.
3. 컨테이너 이미지 관리를 위해 **Artifact Registry** 저장소를 생성합니다:
   ```bash
   gcloud artifacts repositories create mcp-server-repo \
     --repository-format=docker \
     --location=us-central1
   ```
4. Cloud Build로 소스코드를 압축 전송하여 가상 이미지를 빌드한 다음, Cloud Run으로 무중단 서버 배포를 한 번에 지시합니다:
   ```bash
   gcloud builds submit --tag us-central1-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/mcp-server-repo/atlassian-mcp-server:latest .

   gcloud run deploy atlassian-mcp-server \
     --image us-central1-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/mcp-server-repo/atlassian-mcp-server:latest \
     --set-env-vars=JIRA_URL="$(grep JIRA_URL .env | cut -d '=' -f2)",JIRA_EMAIL="$(grep JIRA_EMAIL .env | cut -d '=' -f2)",JIRA_API_TOKEN="$(grep JIRA_API_TOKEN .env | cut -d '=' -f2)" \
     --region us-central1 \
     --allow-unauthenticated
   ```
5. 배포 완료 메세지와 함께 출력되는 서비스 엔드포인트 경로인 **Service URL** 값을 안전한 곳에 보관해 둡니다.

---

## 📍 Task 4. Gemini Enterprise에 커스텀 MCP 데이터 스토어 등록

배포를 성공적으로 완료한 MCP 서버를 Gemini Enterprise 포털에 커스텀 연동 액션(Data Store)으로 정식 등록합니다.

1. 새 브라우저 탭을 열고 **Gemini Enterprise 콘솔**로 진입합니다.
2. 설정 대시보드 메뉴에서 **Custom MCP Server** 추가 혹은 만들기 버튼을 클릭합니다.
3. 구글 인증 자격 증명 스펙에 맞춰 연동 정보를 아래와 같이 필드로 기입합니다:
   * **MCP 서버 URL**: `https://<본인의_Cloud_Run_도메인>/mcp` 
     *(반드시 Cloud Run Service URL 끝에 `/mcp`가 포함되어야 동작 인터페이스가 정상 바인딩됩니다.)*
   * **승인 URL (Authorization URL)**: `https://accounts.google.com/o/oauth2/v2/auth`
   * **승인 URL 매개변수 (Params)**: `&access_type=offline`
   * **토큰 URL (Token URL)**: `https://oauth2.googleapis.com/token`
   * **클라이언트 ID / 비밀번호**: **Task 1**에서 메모해 둔 GCP OAuth 자격증명 정보
   * **범위 (Scopes)**: `https://www.googleapis.com/auth/cloud-platform`
4. 연동 에이전트의 AI 호출 최적화를 위해 **Description** 영역에 다음 영문 설정을 그대로 복사하여 입력합니다:
   ```text
   This MCP server connects to Atlassian services (Jira and Confluence). It provides tools to read and search Confluence pages, and create new pages. It also includes tools to search, retrieve, and create Jira issues. Use this server to analyze planning documents in Confluence and manage tasks in Jira.
   ```
5. 우측 하단의 **로그인(Authorize)** 버튼을 통해 Google 계정 로그인을 승인한 다음 데이터 스토어를 등록 저장합니다.
6. 연동된 에이전트의 행위 규범을 통제하기 위해 **Agent Instructions** 공란에 다음 규칙을 정확히 붙여넣습니다:
   ```text
   1. For read-only actions (searching or reading Confluence pages and Jira issues), proceed immediately without asking for user confirmation.
   2. When asked to CREATE a Jira ticket or a Confluence page, you MUST generate a draft of the content first and ask the user for confirmation before executing the creation tool.
   3. If a space key or project key is not provided by the user, use list_confluence_spaces or list_jira_projects to find the appropriate key, and confirm with the user before proceeding with that key.
   ```
7. 등록 프로세스를 완료한 후 데이터 스토어 상세에서 **Actions > Refresh Custom Actions (커스텀 작업 새로고침)**를 클릭하여 도구를 최종 활성화합니다.

---

## 📍 Task 5. Gemini Enterprise에서 엔터프라이즈 워크플로우 자동화 테스트

모든 파이프라인 연동이 끝났습니다. Gemini Enterprise 대화 인터페이스에서 실질적인 자동화 업무를 처리해 봅니다.

1. Gemini Enterprise 앱 대화창 인터페이스로 이동합니다.
2. 아래와 같이 Atlassian 데이터에 액세스하고 연쇄적인 도구 연동을 처리해야 하는 엔터프라이즈 프롬프트를 입력하고 테스트합니다:

   > **프롬프트 예시:**
   > "컨플루언스에서 '카메라 개발 계획' 문서를 검색해서 읽은 뒤, 그 내용을 바탕으로 Jira에 티켓들을 생성해줘."
