   # Google Drive Summarizer MCP Server and A2UI Agent Deployment

   ## 📋 Overview
   이 실습에서는 Gemini Enterprise Agent Platform을 기반으로 두 가지 에이전트를 배포하고 연동합니다.
   1. **구글 드라이브 파일 내용을 읽고 요약하는 MCP(Model Context Protocol) 서버**
   2. **A2UI 기반 삼성 제품 스펙 비교 에이전트**

   ---

   ## 🚀 Step-by-Step Deployment Guide

   ### Task 1. Create OAuth Credentials
   인증 및 연동을 위한 필수 API를 활성화하고 OAuth 2.0 클라이언트를 생성합니다.

   1. **APIs & Services > Credentials**로 이동합니다.
   2. **Create Credentials > OAuth client ID**를 클릭합니다. 
   3. **Application type**에서 `Web application`을 선택하고, Name은 `agent-auth`로 지정합니다.
   4. **Authorized redirect URIs (승인된 리디렉션 URI)** 섹션에서 **Add URI**를 클릭하고 다음 URL을 추가합니다.
      ```text
      https://vertexaisearch.cloud.google.com/oauth-redirect
      ```
   5. **Create**를 클릭합니다.
   6. 화면에 나타난 **Client ID**와 **Client Secret**을 복사하여 안전한 곳에 보관합니다. *(이 정보는 이후 작업에서 공통으로 사용됩니다.)*

   ---

   ### Task 2. Create and Set Gemini Enterprise App
   에이전트와 MCP 서버를 연동하여 사용할 Gemini Enterprise App을 생성하고 identity를 설정합니다.

   1. Google Cloud Console 검색창에서 **Gemini Enterprise**를 검색 후 **`Create App`**을 진행합니다. 
   *(이름은 기본값을 사용하거나 원하는 대로 지정 가능합니다.)*
   2. App 생성이 완료되면, 화면 하단의 **`Set up identity`**를 클릭합니다.
   3. **`Use Google Identity`**를 선택한 후 **`Confirm`**을 눌러 설정을 완료합니다.
      > **💡 참고:** 이 과정은 Gemini Enterprise를 안전하게 사용하기 위해 로그인과 데이터 접근 권한을 관리할 'Google Cloud Identity'를 연동하는 작업입니다.
   4. 생성된 앱의 **ID(`App ID`)**를 별도로 기록해 둡니다. *(이후 환경 설정 파일에 사용됩니다.)*

   ---

   ### Task 3. Open JupyterLab Terminal
   에이전트 배포 및 MCP 서버 배포를 위해 JupyterLab 터미널을 실행합니다.
   
   1. 내비게이션 메뉴에서 **Gemini Enterprise Agent Platform > Workbench**를 클릭합니다.
   2. 할당된 인스턴스 옆의 **Open JupyterLab**을 클릭합니다.
   3. JupyterLab 런처의 *Other* 섹션 아래에 있는 **Terminal**을 클릭하여 새 터미널 세션을 엽니다.
   <img width="1910" height="923" alt="Screenshot 2026-05-07 at 3 35 58 PM" src="https://github.com/user-attachments/assets/abc48e1b-04da-48e8-9265-fd61bca48b55" />

   ---

   ### Task 4. Build and Deploy Google Drive Summarizer MCP Server
   구글 드라이브 파일 요약을 수행하는 MCP 서버를 Cloud Run에 배포합니다.

   #### 📝 제공 기능 (Tools)
   
   해당 MCP 서버는 현재 다음의 도구를 지원합니다:
   * `summarize_drive_files`: 사용자의 구글 드라이브 내 텍스트 파일(상위 5개)을 가져와서 요약본을 제공합니다. (현재 `text/plain` 파일 형식 지원)
   
   1. JupyterLab 터미널에서 아래 명령어를 실행하여 리포지토리를 클론하고 배포를 진행합니다.
   
   ```bash
   git clone https://github.com/hwangju1116/gdrive_reader_mcp.git
   cd gdrive_reader_mcp
   
   gcloud run deploy gdrive-summarizer \
     --source . \
     --region us-central1 \
     --allow-unauthenticated \
     --set-env-vars CLIENT_ID="[Task 1에서 생성한 본인의_Client_ID]"
   ```
   
   > **⚠️ 중요 (보안 및 아키텍처):**
   > Cloud Run 의 접근 제어는 IAM 인증 방식이어서 Gemini Enterprise 의 OAuth 방식과 호환되지 않습니다. 따라서 서버 자체는 누구나 접근 가능하게 열어두되(`--allow-unauthenticated`), **FastMCP 서버 내부에서 구글 토큰의 유효성을 검증**하는 구조를 취하고 있습니다.

   ---
   
   ### Task 5. Register the MCP Data Store in Gemini Enterprise
   
   Gemini Enterprise에 배포한 Custom MCP 서버를 등록합니다.
   
   1. 새 브라우저 탭을 열고 **Gemini Enterprise 콘솔**로 이동합니다.
   2. 데이터 스토어(Data Store) 메뉴에서 **Create data store**를 클릭 후 **Custom MCP Server**를 클릭합니다.
   3. 인증 및 연결을 위해 다음 필수 정보들을 입력합니다.
      - **MCP 서버 URL:** `https://<본인의_Cloud_Run_도메인>/mcp` (또는 환경에 따라 `/mcp` 없이 입력)
      - **승인 URL:** `https://accounts.google.com/o/oauth2/v2/auth`
      - **승인 URL 매개변수:** `&access_type=offline&prompt=consent`
      - **토큰 URL:** `https://oauth2.googleapis.com/token`
      - **클라이언트 ID / 비밀번호:** 발급받은 Credential 정보 입력
      - **범위 (Scopes):** `https://www.googleapis.com/auth/userinfo.email https://www.googleapis.com/auth/userinfo.profile https://www.googleapis.com/auth/drive.readonly`

   4. **로그인** 버튼을 눌러 인증 권한을 승인한 후 저장합니다.
   5. **Description** 란에 아래 텍스트를 정확히 입력합니다. *(Gemini Enterprise가 MCP 서버의 목적을 이해하도록 돕는 설정입니다.)*
   
   ```text
   This MCP server connects to Google Drive and provides a tool to read and summarize the content of text files in the user's drive.
   ```

   6. **Create**를 눌러 데이터 스토어 등록을 완료합니다.

   ---

   ### Task 6. Activate Custom MCP Method
   1. **App > Data stores**로 이동한 후 방금 등록한 **Custom MCP**를 클릭합니다.
   2. **Actions > Refresh Custom Actions**를 클릭하여 MCP 도구(Tools)들을 최종 활성화합니다.
   <img width="1622" height="444" alt="Screenshot 2026-05-18 at 12 52 29 AM" src="https://github.com/user-attachments/assets/852fc039-f67c-473a-bbda-e79e7afebb9a" />

   #### 💡 [선택 사항] 활성화 실패 또는 연동 오류 발생 시 해결 방법 (Cloud Run 권한 설정)
   배포된 Cloud Run 서비스가 외부(Gemini Enterprise)로부터 요청을 받을 수 있도록 미인증 호출 권한을 명확히 설정합니다.

   1. Google Cloud Console 검색창에서 **Cloud Run**을 검색하여 이동한 후 **Services** 목록을 확인합니다.
   2. 배포한 `atlassian-mcp-server` 인스턴스 왼쪽의 **체크박스**를 선택한 후, 우측 패널 또는 상단 메뉴에서 **Permissions(권한)** 탭으로 진입합니다.
   3. **Add Principal (구성원 추가)**을 클릭합니다.
   <img width="1464" height="799" alt="Screenshot 2026-05-09 at 8 01 39 PM" src="https://github.com/user-attachments/assets/2ecfa29a-df28-4635-acc5-a4431bf026fa" />

   4. 설정을 다음과 같이 입력한 후 저장합니다.
      - **New principals:** `allUsers`
      - **Role:** `Cloud Run Invoker (Cloud Run 호출자)`
     

   > ** cloud shell 에서 한번에 설정하는 법:**
  ```bash
  gcloud run services add-iam-policy-binding gdrive-summarizer --region="us-central1" --member="allUsers" --role="roles/run.invoker"
  ```
   
   ---

   ### Task 7. Automate Workflows with Gemini

   모든 설정이 완료되었습니다. 이제 에이전트와 연동하여 실제 업무 자동화 워크플로우를 테스트합니다.
   
   1. **Gemini Enterprise App**의 채팅 인터페이스(Chat UI)로 이동합니다.
   2. **Connector** 아이콘을 클릭한 후, 등록한 **mcp data connector** 옆의 **Authorize** 버튼을 눌러 구글 드라이브 연동을 승인해 줍니다.
   3. **Google Search** 는 `off` 해줍니다.
   <img width="1460" height="824" alt="Screenshot 2026-06-07 at 12 35 51 PM" src="https://github.com/user-attachments/assets/b3f8805d-bb93-4b76-9f37-cca5de101efb" />

   4. 채팅창에 아래 프롬프트를 입력하고 연동 시나리오가 정상 작동하는지 확인합니다.
      
      **📝 테스트 프롬프트 예시:**
      > "내 구글 드라이브에 있는 최근 텍스트 파일들을 읽고 주요 내용을 요약해 줘."
      
   ---

   ### Task 8. Deploy the A2UI Agent
   JupyterLab 터미널을 통해 삼성 제품 스펙 비교 에이전트(A2UI Agent)를 배포합니다.
   
   #### 📝 주요 기능 (Features)
   해당 에이전트는 A2UI(Agent-to-User Interface)를 활용하여 다음 기능을 제공합니다:
   * **삼성 제품 스펙 비교**: 갤럭시 스마트폰, 노트북 등 다양한 삼성 제품의 세부 사양(스펙)을 다차원으로 비교 분석합니다.
   * **시각적 UI 제공 (A2UI)**: 단순 텍스트 답변을 넘어, 사용자가 한눈에 직관적으로 스펙을 비교할 수 있도록 구조화된 사용자 인터페이스(UI)를 동적으로 생성하여 제공합니다.

   1. 아래 명령어를 실행하여 리포지토리를 복제하고 의존성 관리 도구(`uv`)를 설치한 뒤 환경 설정 파일을 생성합니다.
      ```bash
      git clone https://github.com/hwangju1116/a2ui-agent.git
      cd a2ui-agent
      pip install uv
      ```
   2. `a2ui-agent/deploy.py`로 와서 아래 내용을 수정합니다.
      ```bash
      # 본인의 GEMINI_ENTERPRISE_APP_ID를 입력해주세요.
      GEMINI_ENTERPRISE_APP_ID = "[본인의_GEMINI_ENTERPRISE_APP_ID]"
      ```
      <img width="1459" height="685" alt="Screenshot 2026-06-07 at 1 05 18 PM" src="https://github.com/user-attachments/assets/36367cb2-e4f4-4629-acf1-1308da600fd6" />

      그리고 `a2ui-agent/generate_auth.sh`로 가서 아래 내용에 아까 복사해둔 Client ID & Client Secret을 넣어줍니다.
      ```bash
      # 아래 값들을 찾아서 실제 발급받은 값으로 수정해주세요.
      CLIENT_ID="[본인의_Client_ID]"
      CLIENT_SECRET="[본인의_Client_Secret]"
      ```
  
   3. 터미널로 돌아와 아래 명령어를 실행하여 에이전트를 배포합니다.
      ```bash
      chmod +x generate_auth.sh
      ./generate_auth.sh
      uv run deploy.py
      ```
   4. 배포 완료 후 Gemini Enterprise > 생성한 app > agent에서 배포된 agent를 확인하실 수 있습니다. 
   (`deploy.py` 안에 engine 배포 후 gemini enterprise app 안에 agnet 등록까지 다 구현되어있습니다.)
