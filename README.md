# A2UI Agent and Atlassian MCP Server Deployment

### **Overview**
이 실습에서는 Gemini Enterprise Agent Platform을 기반으로 두 가지 에이전트를 배포하고 연동합니다. 첫 번째는 A2UI 기반 삼성 제품 스펙 비교 에이전트이며, 두 번째는 Atlassian Jira 및 Confluence와 연동하여 업무를 자동화하는 MCP 서버입니다.

---

### **Task 1. Enable APIs and create OAuth credentials**
1. Google Cloud console의 내비게이션 메뉴( ☰ )에서 **APIs & Services > Library**를 클릭합니다.
2. 다음 API들을 각각 검색하여 **Enable**을 클릭합니다:
   - **Discovery Engine API**
   - **Vertex AI API**
   - **Cloud Run API**
   - **Artifact Registry API**
3. **APIs & Services > Credentials**를 클릭합니다.
4. **Create Credentials > OAuth client ID**를 클릭합니다.
5. Application type에서 **Web application**을 선택하고 Name은 “agent-auth”로 지정합니다.
6. **Authorized redirect URIs (승인된 리디렉션 URI)** 섹션에서 **Add URI**를 클릭하고 다음 URL을 추가합니다:
   `https://vertexaisearch.cloud.google.com/oauth-redirect`
7. **Create**를 클릭합니다.
8. 화면에 나타난 **Client ID**와 **Client Secret**을 복사하여 안전한 곳에 보관합니다. 이 정보는 다음 작업들에서 공통으로 사용됩니다.

---

### **Task 2. Create and Set Gemini Enterprise App**
앞으로 생성할 agent와 mcp서버를 붙여서 사용할 gemini enterprise app을 만드는 과정입니다.
1. Gemini Enterprise 콘솔에서 **Create App**을 실행합니다. (이름은 원하시는 대로 지정하거나 기본 명칭을 사용하셔도 됩니다.)
2. App 생성 후, 화면 하단 왼쪽에 있는 **Set up identity**를 클릭하고 **Use Google Identity**를 선택한 후 **Confirm**을 눌러 설정을 완료합니다.
   * **설명:** 이 과정은 "Gemini Enterprise를 안전하게 사용하기 위해, 로그인과 데이터 접근 권한을 관리할 '구글 클라우드 아이덴티티(Google Cloud Identity)'를 연동하겠다"는 의미입니다.
3. 생성한 앱의 ID를 적어두어 다음 단계의 설정 파일에 추가합니다.

---

### **Task 3. Deploy the A2UI Agent**
1. Google Cloud console의 내비게이션 메뉴에서 **Gemini Enterprise Agent Platform > Workbench**를 클릭합니다.
2. 할당된 인스턴스 옆의 **Open JupyterLab**을 클릭합니다.
3. JupyterLab 런처의 **Other** 섹션 아래에 있는 **Terminal**을 클릭하여 새 터미널 세션을 엽니다.
4. 리포지토리를 복제하고 uv 패키지 매니저를 설치하기 위해 다음 명령어를 실행합니다:
   ```bash
   git clone https://github.com/hwangju1116/a2ui-agent.git
   cd a2ui-agent
   pip install uv
   touch .env
에이전트가 외부 환경과 통신할 수 있도록 Authorization(권한) 리소스를 생성해야 합니다. 터미널에 아래 명령어들을 복사하여 실행하세요. (대괄호 [...] 안의 내용은 본인의 실제 값으로 변경해야 합니다.)
터미널 좌측의 파일 브라우저에서 a2ui-agent 폴더 안의 .env 파일을 열고 다음 내용을 모두 붙여넣습니다:
괄호 [...]로 표시된 값들을 실제 환경에 맞게 수정한 뒤, 파일을 반드시 **저장(Save)**하고 닫습니다.
터미널로 돌아와 다음 명령어를 실행하여 에이전트를 배포합니다:
배포가 완료되면 터미널에 출력된 REASONING_ENGINE_ID를 확인합니다.

--------------------------------------------------------------------------------
Task 4. Configure Agent Access Control
배포된 Samsung Comparison Agent를 Gemini Enterprise나 다른 시스템에서 호출하여 사용하려면, 호출하는 주체(서비스 계정)에 적절한 권한을 부여해야 합니다. 권한이 없으면 401 Unauthorized 에러가 발생합니다. 아래의 두 가지 방법 중 하나를 선택하여 권한을 부여해 주세요.
방법 1: 구글 클라우드 콘솔(웹) 이용하기
IAM 및 관리자 페이지 이동: Google Cloud 콘솔에 로그인한 후, 메뉴에서 IAM 및 관리자 > IAM으로 이동합니다.
호출자 서비스 계정 찾기: 리스트에서 에이전트를 호출할 주체가 되는 서비스 계정을 찾습니다.
팁: Gemini Enterprise를 사용 중이라면 service-PROJECT_NUMBER@gcp-sa-discoveryengine.iam.gserviceaccount.com 형태의 계정일 확률이 높습니다.
권한 수정: 해당 서비스 계정 우측의 **수정(연필 모양 아이콘)**을 클릭합니다.
역할 추가: **다른 역할 추가(Add Another Role)**를 클릭합니다.
권한 선택: 역할 검색창에 Vertex AI User를 입력하고, Vertex AI 사용자 (Vertex AI User) 역할을 선택합니다.
저장: 저장 버튼을 눌러 설정을 완료합니다.
방법 2: gcloud CLI(터미널) 이용하기 터미널 환경이 편하시다면 아래의 명령어를 통해 한 줄로 권한을 부여할 수 있습니다.
터미널 열기: 프로젝트 환경의 터미널을 엽니다.
명령어 실행: 아래 명령어에서 [SERVICE_ACCOUNT_EMAIL] 부분만 실제 서비스 계정 이메일로 변경하여 실행합니다.

--------------------------------------------------------------------------------
Task 5. Build and deploy the Atlassian MCP server
동일한 터미널 창에서 상위 디렉터리로 이동한 후 두 번째 리포지토리를 복제합니다:
jira_confluence_mcp 폴더 안의 .env 파일을 열고 다음 내용을 붙여넣은 뒤, 실제 Atlassian 계정 정보로 변경하여 저장합니다:
MCP 서버 도커 이미지를 빌드하고 Cloud Run에 배포합니다. 아래 명령어는 빌드와 배포를 하나로 합쳐서 처리하는 최적화 명령어입니다:
배포가 완료되면 터미널에 표시된 Service URL을 복사합니다.

--------------------------------------------------------------------------------
Task 6. Register the MCP Data Store in Gemini Enterprise
새 브라우저 탭을 열고 Gemini Enterprise 콘솔로 이동합니다.
데이터 스토어 메뉴에서 Custom MCP Server 추가를 클릭합니다.
인증 및 연결을 위해 다음 필수 정보를 입력합니다:
MCP 서버 URL: https://<본인의_Cloud_Run_도메인>/mcp
승인 URL: https://accounts.google.com/o/oauth2/v2/auth
승인 URL 매개변수: &access_type=offline
토큰 URL: https://oauth2.googleapis.com/token
클라이언트 ID / 비밀번호: Task 1에서 발급받은 정보
범위 (Scopes): https://www.googleapis.com/auth/cloud-platform
&access_type=offline를 추가하는 이유: 인증 시 Refresh Token을 함께 발급받아, Access Token이 만료되더라도 백그라운드에서 주기적으로 갱신하여 끊김 없는 서비스를 유지하기 위함입니다.
로그인 버튼을 눌러 권한을 승인한 후 저장합니다.
Description 란에 아래 텍스트를 입력합니다: This MCP server connects to Atlassian services (Jira and Confluence). It provides tools to read and search Confluence pages, and create new pages. It also includes tools to search, retrieve, and create Jira issues. Use this server to analyze planning documents in Confluence and manage tasks in Jira.
Agent Instructions에 다음 내용을 추가합니다:
Create를 눌러 생성을 완료합니다.

--------------------------------------------------------------------------------
Task 7. Add Permissions To Cloud Run
Google Cloud console 검색창에 Cloud Run을 검색하여 진입한 후, 배포된 atlassian-mcp-server 서비스를 클릭합니다.
해당 서비스 인스턴스 상세 페이지에서 Security 또는 Permissions (권한) 탭으로 이동합니다.
**Add Principal (구성원 추가)**을 클릭합니다.
New principals (새 구성원): allUsers
Role (역할): Cloud Run Invoker (Cloud Run 호출자)
저장을 누릅니다.

--------------------------------------------------------------------------------
Task 8. Active Custom MCP method
Gemini Enterprise 콘솔의 App > Data stores에서 등록한 custom MCP 데이터 스토어를 선택하여 진입합니다.
**Actions > Refresh Custom Actions (커스텀 작업 새로고침)**를 클릭하여 도구를 활성화합니다.

--------------------------------------------------------------------------------
Task 9. Automate workflows with Gemini
Gemini 채팅 인터페이스로 이동합니다.
다음 프롬프트를 입력하고 연동 테스트를 진행합니다: 컨플루언스에서 '카메라 개발 계획' 문서를 검색해서 읽은 뒤, 그 내용을 바탕으로 Jira에 티켓들을 생성해줘.

---
