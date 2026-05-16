# A2UI Agent and Atlassian MCP Server Deployment

## 📋 Overview
이 실습에서는 Gemini Enterprise Agent Platform을 기반으로 두 가지 에이전트를 배포하고 연동합니다.
1. **A2UI 기반 삼성 제품 스펙 비교 에이전트**
2. **Atlassian Jira 및 Confluence와 연동하여 업무를 자동화하는 MCP(Model Context Protocol) 서버**

---

## 🛠️ Setup and Requirements

### Qwiklabs Setup & Activate Cloud Shell
실습 시작 전 Qwiklabs 가이드에 따라 Google Cloud 환경을 준비하고 Cloud Shell을 활성화하세요.

---

## 🚀 Step-by-Step Deployment Guide

### Task 1. Enable APIs and Create OAuth Credentials
인증 및 연동을 위한 필수 API를 활성화하고 OAuth 2.0 클라이언트를 생성합니다.

1. **Google Cloud Console**의 내비게이션 메뉴에서 **APIs & Services > Library**를 클릭합니다.
2. 다음 API를 검색하여 **Enable(사용)**을 클릭합니다.
   - `Discovery Engine API`
3. **APIs & Services > Credentials**로 이동합니다.
4. **Create Credentials > OAuth client ID**를 클릭합니다.
5. **Application type**에서 `Web application`을 선택하고, Name은 `agent-auth`로 지정합니다.
6. **Authorized redirect URIs (승인된 리디렉션 URI)** 섹션에서 **Add URI**를 클릭하고 다음 URL을 추가합니다.
   ```text
   https://vertexaisearch.cloud.google.com/oauth-redirect
   ```
7. **Create**를 클릭합니다.
8. 화면에 나타난 **Client ID**와 **Client Secret**을 복사하여 안전한 곳에 보관합니다. *(이 정보는 이후 작업에서 공통으로 사용됩니다.)*

---

### Task 2. Create and Set Gemini Enterprise App
에이전트와 MCP 서버를 연동하여 사용할 Gemini Enterprise App을 생성하고 identity를 설정합니다.

1. Google Cloud Console 검색창에서 **Gemini Enterprise**를 검색합니다.
2. 검색 결과에서 **"30-day free trial"**을 선택한 후 **Create App**을 진행합니다. *(이름은 기본값을 사용하거나 원하는 대로 지정 가능합니다.)*
3. App 생성이 완료되면, 화면 하단의 **"Set up identity"**를 클릭합니다.
4. **"Use Google Identity"**를 선택한 후 **"Confirm"**을 눌러 설정을 완료합니다.
   > **💡 참고:** 이 과정은 Gemini Enterprise를 안전하게 사용하기 위해 로그인과 데이터 접근 권한을 관리할 'Google Cloud Identity'를 연동하는 작업입니다.
5. 생성된 앱의 **ID(App ID)**를 별도로 기록해 둡니다. *(이후 환경 설정 파일에 사용됩니다.)*

---

### Task 3. Deploy the A2UI Agent
JupyterLab 터미널을 통해 삼성 제품 스펙 비교 에이전트(A2UI Agent)를 배포합니다.

1. 내비게이션 메뉴에서 **Gemini Enterprise Agent Platform > Workbench**를 클릭합니다.
2. 할당된 인스턴스 옆의 **Open JupyterLab**을 클릭합니다.
3. JupyterLab 런처의 *Other* 섹션 아래에 있는 **Terminal**을 클릭하여 새 터미널 세션을 엽니다.
4. 아래 명령어를 실행하여 리포지토리를 복제하고 의존성 관리 도구(`uv`)를 설치한 뒤 환경 설정 파일을 생성합니다.
   ```bash
   git clone https://github.com/hwangju1116/a2ui-agent.git
   cd a2ui-agent
   pip install uv
   touch .env
   ```
5. 에이전트가 외부 환경과 통신할 수 있도록 Authorization(권한) 리소스를 생성합니다. 아래 명령어에서 대괄호(`[...]`) 안의 내용을 본인의 값으로 변경한 후 터미널에서 실행하세요.
   ```bash
   export PROJECT_ID=$(gcloud config get-value project)
   export AUTH_ID=agent-auth
   export CLIENT_ID=[Task 1에서 복사한 Client ID]
   export CLIENT_SECRET=[Task 1에서 복사한 Client Secret]

   curl -X POST      -H "Authorization: Bearer $(gcloud auth print-access-token)"      -H "Content-Type: application/json"      -H "X-Goog-User-Project: ${PROJECT_ID}"      "https://global-discoveryengine.googleapis.com/v1alpha/projects/${PROJECT_ID}/locations/global/authorizations?authorizationId=${AUTH_ID}"      -d '{
           "name": "projects/'"${PROJECT_ID}"'/locations/global/authorizations/'"${AUTH_ID}"'",
           "serverSideOauth2": {
              "clientId": "'"${CLIENT_ID}"'",
              "clientSecret": "'"${CLIENT_SECRET}"'",
              "authorizationUri": "https://accounts.google.com/o/oauth2/v2/auth?client_id='"${CLIENT_ID}"'&redirect_uri=https%3A%2F%2Fvertexaisearch.cloud.google.com%2Fstatic%2Foauth%2Foauth.html&scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloud-platform&response_type=code&access_type=offline&prompt=consent",
              "tokenUri": "https://oauth2.googleapis.com/token"
           }
        }'
   ```
   > **⚠️ 중요:** 위 API 호출 성공 후 응답 결과에서 `"name"` 키로 반환된 값을 복사하여 보관하세요. 다음 단계의 `AGENT_AUTHORIZATION` 값으로 사용됩니다.

6. JupyterLab 좌측 파일 브라우저에서 `a2ui-agent` 폴더 안의 `.env` 파일을 열고 아래 내용을 작성합니다.
   ```env
   PROJECT_ID=[본인의_PROJECT_ID]
   LOCATION=us-central1
   STORAGE_BUCKET=[본인의_PROJECT_ID]-a2ui-bucket
   GEMINI_ENTERPRISE_APP_ID=[Task 2에서 복사한 Gemini App ID]
   AGENT_AUTHORIZATION=[위 5번 단계에서 복사한 authorization name 값]
   ```
7. 대괄호(`[...]`) 안의 값을 알맞게 수정한 뒤 파일을 저장하고 닫습니다.
8. 터미널로 돌아와 아래 명령어를 실행하여 에이전트를 배포합니다.
   ```bash
   uv run deploy.py
   ```
9. 배포 완료 후 터미널에 출력되는 `REASONING_ENGINE_ID`를 확인하고 기록해 둡니다.

---

### Task 4. Configure Agent Access Control
배포된 에이전트를 Gemini Enterprise 등의 외부 시스템에서 에러 없이 호출할 수 있도록 호출 주체(서비스 계정)에 권한을 부여합니다. 권한이 없을 경우 `401 Unauthorized` 에러가 발생합니다.

아래 두 가지 방법 중 하나를 선택하여 진행하세요.

#### 방법 1: Google Cloud Console (웹 GUI) 이용
1. **IAM 페이지 이동**: Google Cloud 콘솔 메뉴에서 **IAM & Admin > IAM**으로 이동합니다.
2. **서비스 계정 확인**: 에이전트를 호출할 주체가 되는 서비스 계정을 찾습니다.
   - *Tip:* Gemini Enterprise를 사용하는 경우 일반적으로 `service-PROJECT_NUMBER@gcp-sa-discoveryengine.iam.gserviceaccount.com` 형태의 계정입니다.
3. **권한 수정**: 해당 서비스 계정 우측의 **수정(연필 모양 아이콘)**을 클릭합니다.
4. **역할 추가**: **Add Another Role (다른 역할 추가)**을 클릭합니다.
5. **역할 선택**: 검색창에 `Vertex AI User` 또는 `aiplatform.user`를 입력하고 **Vertex AI 사용자 (Vertex AI User)** 역할을 선택합니다.
6. **저장**: **Save**를 눌러 권한 반영을 완료합니다.

#### 방법 2: gcloud CLI (터미널) 이용
터미널 환경에서 아래 명령어를 실행하여 한 줄로 권한을 부여할 수 있습니다. (`--member` 부분의 이메일 주소를 본인의 타겟 서비스 계정으로 변경하세요.)
```bash
# 현재 활성화된 프로젝트에 자동으로 권한을 부여하는 명령어 예시
gcloud projects add-iam-policy-binding $(gcloud config get-value project)    --member="serviceAccount:student-04-930edeff2819@qwiklabs.net"    --role="roles/aiplatform.user"
```

---

### Task 5. Build and Deploy the Atlassian MCP Server
Atlassian(Jira/Confluence) 연동을 위한 MCP 서버를 빌드하고 Cloud Run에 소스 배포합니다.

1. 기존 터미널 창에서 상위 디렉터리로 이동한 후, 두 번째 리포지토리를 복제합니다.
   ```bash
   cd ~
   git clone https://github.com/hwangju1116/jira_confluence_mcp.git
   cd jira_confluence_mcp
   touch .env
   ```
2. `jira_confluence_mcp` 폴더 안의 `.env` 파일을 열고, 실제 Atlassian 계정 정보에 맞추어 아래 내용을 입력 후 저장합니다.
   ```env
   JIRA_URL=[본인의-지라-URL]
   JIRA_EMAIL=[본인의-지라-이메일]
   JIRA_API_TOKEN=[본인의-지라-API-토큰]
   ```
3. 아래 명령어를 실행하여 MCP 서버를 Google Cloud Run에 배포합니다. `--source .` 옵션을 통해 소스코드를 자동으로 빌드, 이미지화 및 배포까지 원스톱으로 처리합니다.
   ```bash
   gcloud run deploy atlassian-mcp-server      --source .      --set-env-vars=JIRA_URL="$(grep JIRA_URL .env | cut -d '=' -f2-)",JIRA_EMAIL="$(grep JIRA_EMAIL .env | cut -d '=' -f2-)",JIRA_API_TOKEN="$(grep JIRA_API_TOKEN .env | cut -d '=' -f2-)"      --region us-central1      --allow-unauthenticated      --project=$GOOGLE_CLOUD_PROJECT
   ```
4. 배포가 완료되면 터미널 화면에 표시되는 **Service URL**을 복사하여 보관합니다.

---

### Task 6. Add Permissions to Cloud Run
배포된 Cloud Run 서비스가 외부(Gemini Enterprise)로부터 요청을 받을 수 있도록 미인증 호출 권한을 명확히 설정합니다.

1. Google Cloud Console 검색창에서 **Cloud Run**을 검색하여 이동한 후 **Services** 목록을 확인합니다.
2. 배포한 `atlassian-mcp-server` 인스턴스 왼쪽의 **체크박스**를 선택한 후, 우측 패널 또는 상단 메뉴에서 **Permissions(권한)** 탭으로 진입합니다.
3. **Add Principal (구성원 추가)**을 클릭합니다.
4. 설정을 다음과 같이 입력한 후 저장합니다.
   - **New principals:** `allUsers`
   - **Role:** `Cloud Run Invoker (Cloud Run 호출자)`

---

### Task 7. Register the MCP Data Store in Gemini Enterprise
Gemini Enterprise에 배포한 Custom MCP 서버를 등록합니다.

1. 새 브라우저 탭을 열고 **Gemini Enterprise 콘솔**로 이동합니다.
2. 데이터 스토어(Data Store) 메뉴에서 **Custom MCP Server 추가**를 클릭합니다.
3. 인증 및 연결을 위해 다음 필수 정보들을 입력합니다.
   - **MCP 서버 URL:** `https://<본인의_Cloud_Run_도메인>/mcp`
   - **승인 URL:** `https://accounts.google.com/o/oauth2/v2/auth`
   - **승인 URL 매개변수:** `&access_type=offline`
   - **토큰 URL:** `https://oauth2.googleapis.com/token`
   - **클라이언트 ID / 비밀번호:** Task 1에서 발급받은 Credential 정보 입력
   - **범위 (Scopes):** `https://www.googleapis.com/auth/cloud-platform`
4. **로그인** 버튼을 눌러 인증 권한을 승인한 후 저장합니다.
5. **Description** 란에 아래 텍스트를 정확히 입력합니다. *(Gemini Enterprise가 MCP 서버의 목적을 이해하고 알맞게 호출하도록 유도하는 설정입니다.)*
   ```text
   This MCP server connects to Atlassian services (Jira and Confluence). It provides tools to read and search Confluence pages, and create new pages. It also includes tools to search, retrieve, and create Jira issues. Use this server to analyze planning documents in Confluence and manage tasks in Jira.
   ```
6. **Agent Instructions**에 다음 내부 가이드라인 내용을 추가합니다.
   ```text
   1. For read-only actions (searching or reading Confluence pages and Jira issues), proceed immediately without asking for user confirmation.
   2. When asked to CREATE a Jira ticket or a Confluence page, you MUST generate a draft of the content first and ask the user for confirmation before executing the creation tool.
   3. If a space key or project key is not provided by the user, use list_confluence_spaces or list_jira_projects to find the appropriate key, and confirm with the user before proceeding with that key.
   ```
7. **Create**를 눌러 데이터 스토어 등록을 완료합니다.

---

### Task 8. Activate Custom MCP Method
1. **App > Data stores**로 이동한 후 방금 등록한 **Custom MCP**를 클릭합니다.
2. **Actions > Refresh Custom Actions**를 클릭하여 MCP 도구(Tools)들을 최종 활성화합니다.

---

### Task 9. Automate Workflows with Gemini
모든 설정이 완료되었습니다. 이제 에이전트와 연동하여 실제 업무 자동화 워크플로우를 테스트합니다.

1. **Gemini Enterprise App**의 채팅 인터페이스(Chat UI)로 이동합니다.
2. **Connector** 아이콘을 클릭한 후, 등록한 **mcp data connector** 옆의 **Authorize** 버튼을 눌러 연동을 승인해 줍니다.
3. 채팅창에 아래 프롬프트를 입력하고 연동 시나리오가 정상 작동하는지 확인합니다.
   
   **📝 테스트 프롬프트 예시:**
   > "컨플루언스에서 '카메라 개발 계획' 문서를 검색해서 읽은 뒤, 그 내용을 바탕으로 Jira에 티켓들을 생성해줘."
