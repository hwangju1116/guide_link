# guide_link
a2ui &amp; mcp  실습가이드 문서입니다
# A2UI Agent and Atlassian MCP Server Deployment

# **Overview**

이 실습에서는 Gemini Enterprise Agent Platform을 기반으로 두 가지 에이전트를 배포하고 연동합니다. 첫 번째는 A2UI 기반 삼성 제품 스펙 비교 에이전트이며, 두 번째는 Atlassian Jira 및 Confluence와 연동하여 업무를 자동화하는 MCP 서버입니다.

# **Setup and requirements**

## **Qwiklabs setup & Activate Cloud Shell**

\[\[import start-qwiklab\]\]

# **Task 1\. Enable APIs and create OAuth credentials**

1\. Google Cloud console의 내비게이션 메뉴에서 APIs & Services \> Library를 클릭합니다.

2\. 다음 4개의 API를 각각 검색하여 Enable을 클릭합니다:  
  \- Discovery Engine API

3\. APIs & Services \>  Credentials를 클릭합니다.

4\. Create Credentials \> OAuth client ID를 클릭합니다.

5\. Application type에서 Web application을 선택하고 Name은 “agent-auth”로 지정합니다.

6\. Authorized redirect URIs (승인된 리디렉션 URI) 섹션에서 Add URI를 클릭하고 다음 URL을 추가합니다:

| https://vertexaisearch.cloud.google.com/oauth-redirect |
| :---- |

7\. Create를 클릭합니다.

8\. 화면에 나타난 Client ID와 Client Secret을 복사하여 안전한 곳에 보관합니다. 이 정보는 다음 작업들에서 공통으로 사용됩니다.

# **Task 2\. Create and Set Gemini Enterprise App**

앞으로 생성할 agent와 mcp서버를 붙여서 사용할 gemini enteprise app을 만드는 과정입니다.

1\. Gemini Enterprise로 검색을 합니다.

2\. 검색하면 보이는 “30-day free trial”을 선택 후 create app을 합니다.  
 (이름은 원하시는대로 정하셔도 됩니다. Default 로 되어있는 명칭을 사용하셔도됩니다.)  
![][image1]

3\. App 생성 후 들어간 화면 하단에 “Set up identity”를 클릭하여 “Use Google Identity”를 선택후 “Confirm”을 눌러서 설정을 완료합니다.   
\*\* 이과정은 “Gemini Enterprise를 안전하게 사용하기 위해, 로그인과 데이터 접근 권한을 관리할 '구글 클라우드 아이덴티티(Google Cloud Identity)'를 연동하겠다”는 것을 설정한것입니다. \*\*

4\. 생성한 앱의 ID 적어두셔서 다음 스텝들 설정파일에 추가하면 됩니다.

![][image2]

# **Task 2\. Deploy the A2UI Agent**

1\. 내비게이션 메뉴에서 Gemini Enterprise Agent Platform \> Workbench를 클릭합니다.

2\. 할당된 인스턴스 옆의 Open JupyterLab을 클릭합니다.

3\. JupyterLab 런처의 Other 섹션 아래에 있는 Terminal을 클릭하여 새 터미널 세션을 엽니다.

![][image3]

4\. 리포지토리를 복제하고 uv를 설치하기 위해 다음 명령어를 실행합니다:

| git clone https://github.com/hwangju1116/a2ui-agent.gitcd a2ui-agentpip install uvtouch .env |
| :---- |

5\. 에이전트가 외부 환경과 통신할 수 있도록 Authorization(권한) 리소스를 생성해야 합니다. 터미널에 아래 명령어들을 복사하여 실행하세요. (대괄호 \[...\] 안의 내용을 본인의 값으로 바꾼 뒤 실행해야 합니다.)

| export PROJECT\_ID=$(gcloud config get-value project)export AUTH\_ID=agent-authexport CLIENT\_ID=\[Task 1에서 복사한 Client ID\]export CLIENT\_SECRET=\[Task 1에서 복사한 Client Secret\]curl \-X POST    \-H "Authorization: Bearer $(gcloud auth print-access-token)"    \-H "Content-Type: application/json"    \-H "X-Goog-User-Project: ${PROJECT\_ID}"    "https://global-discoveryengine.googleapis.com/v1alpha/projects/${PROJECT\_ID}/locations/global/authorizations?authorizationId=${AUTH\_ID}"    \-d '{       "name": "projects/'"${PROJECT\_ID}"'/locations/global/authorizations/'"${AUTH\_ID}"'",       "serverSideOauth2": {          "clientId": "'"${CLIENT\_ID}"'",          "clientSecret": "'"${CLIENT\_SECRET}"'",          "authorizationUri": "https://accounts.google.com/o/oauth2/v2/auth?client\_id='"${CLIENT\_ID}"'\&redirect\_uri=https%3A%2F%2Fvertexaisearch.cloud.google.com%2Fstatic%2Foauth%2Foauth.html\&scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloud-platform\&response\_type=code\&access\_type=offline\&prompt=consent",          "tokenUri": "https://oauth2.googleapis.com/token"       }    }' |
| :---- |

6\. 터미널 좌측의 파일 브라우저에서 a2ui-agent 폴더 안의 .env 파일을 열고 다음 내용을 모두 붙여넣습니다:

| PROJECT\_ID=\[PROJECT\_ID\]LOCATION=us-central1STORAGE\_BUCKET=\[PROJECT\_ID\]-a2ui-bucketGEMINI\_ENTERPRISE\_APP\_ID=AGENT\_AUTHORIZATION=\[Agent\_authorization\] |
| :---- |

7\. 괄호 \[...\]로 표시된 값들을 수정한 뒤, 파일을 저장(Save)하고 닫습니다.

8\. 터미널로 돌아와 다음 명령어를 실행하여 에이전트를 배포합니다:

uv run deploy.py

9\. 배포가 완료되면 터미널에 출력된 REASONING\_ENGINE\_ID를 확인합니다.

# **Task 3\. Configure Agent Access Control**  

배포된 Samsung Comparison Agent를 Gemini Enterprise나 다른 시스템에서 호출하여 사용하려면, 호출하는 주체(서비스 계정)에 적절한 권한을 부여해야 합니다. 권한이 없으면 401 Unauthorized 에러가 발생합니다.  
아래의 두 가지 방법 중 편한 방법을 선택하여 권한을 부여해 주세요.

**방법 1: 구글 클라우드 콘솔(웹) 이용하기**

1. **IAM 및 관리자 페이지 이동**: Google Cloud 콘솔에 로그인한 후, 메뉴에서 **IAM 및 관리자 \> IAM**으로 이동합니다.  
2. **호출자 서비스 계정 찾기**: 리스트에서 에이전트를 호출할 주체가 되는 서비스 계정을 찾습니다.  
   * 팁*: Gemini Enterprise*를 사용 중이라면 *service-PROJECT\_NUMBER@gcp-sa-discoveryengine.iam.gserviceaccount.com* 형태의 계정일 확률이 높습니다*.*  
3. **권한 수정**: 해당 서비스 계정 우측의 ***수정(연필 모양 아이콘)***을 클릭합니다.  
4. **역할 추가**: ***다른 역할 추가(Add Another Role)***를 클릭합니다.  
5. **권한 선택**: 역할 검색창에 Vertex AI User 또는 ***aiplatform.user***를 입력하고, **Vertex AI 사용자 (Vertex AI User)** 역할을 선택합니다.  
6. **저장**: **저장** 버튼을 눌러 설정을 완료합니다.

**방법 2: gcloud CLI(터미널) 이용하기**  
터미널 환경이 편하시다면 아래의 명령어를 통해 한 줄로 권한을 부여할 수 있습니다.

1. **터미널 열기**: 프로젝트 환경의 터미널을 엽니다.  
2. **명령어 실행**: 아래 명령어에서 \[SERVICE\_ACCOUNT\_EMAIL\] 부분만 실제 서비스 계정 이메일로 변경하여 실행합니다.

| \# 현재 활성화된 프로젝트에 자동으로 권한을 부여하는 명령어 gcloud projects add-iam-policy-binding $(gcloud config get-value project) \\    \--member="serviceAccount:student-04-930edeff2819@qwiklabs.net" \\    \--role="roles/aiplatform.user" |
| :---- |

# **Task 4\. Build and deploy the Atlassian MCP server**

1\. 동일한 터미널 창에서 상위 디렉터리로 이동한 후 두 번째 리포지토리를 복제합니다:

| cd \~git clone https://github.com/hwangju1116/jira\_confluence\_mcp.gitcd jira\_confluence\_mcptouch .env |
| :---- |

2\. jira\_confluence\_mcp 폴더 안의 .env 파일을 열고 다음 내용을 붙여넣은 뒤, 실제 Atlassian 계정 정보로 변경하여 저장합니다:

| JIRA\_URL=\[본인의-지라-URL\]JIRA\_EMAIL=\[본인의-지라-이메일\]JIRA\_API\_TOKEN=\[본인의-지라-API-토큰\] |
| :---- |

3\. MCP 서버 도커 이미지를 빌드하고 Cloud Run에 배포합니다:

아래명령어는 빌드와 배포를 하나로 합쳐서 처리하는 명령어입니다.

바로 \--source . 옵션을 사용하는 방법입니다.

아래 명령어를 실행하시면, 구글이 알아서 소스코드를 압축해 클라우드로 보낸 뒤 빌드하고, 이미지로 만들고, 배포까지 한 번에 다 해줍니다. (이 경우 Artifact Registry 저장소를 미리 만들 필요도 없습니다\!)

| gcloud run deploy atlassian-mcp-server \\   \--source . \\   \--set-env-vars=JIRA\_URL="$(grep JIRA\_URL .env | cut \-d '=' \-f2)",JIRA\_EMAIL="$(grep JIRA\_EMAIL .env | cut \-d '=' \-f2)",JIRA\_API\_TOKEN="$(grep JIRA\_API\_TOKEN .env | cut \-d '=' \-f2)" \\   \--region us-central1 \\   \--allow-unauthenticated \\   \--project=$GOOGLE\_CLOUD\_PROJECT |
| :---- |

4\. 배포가 완료되면 터미널에 표시된 Service URL을 복사합니다.

# **Task 4\. Register the MCP Data Store in Gemini Enterprise**

1\. 새 브라우저 탭을 열고 Gemini Enterprise 콘솔로 이동합니다.

2\. 데이터 스토어 메뉴에서 Custom MCP Server 추가를 클릭합니다.

3\. 인증 및 연결을 위해 다음 필수 정보를 입력합니다:  
  \- MCP 서버 URL: https://\<본인의\_Cloud\_Run\_도메인\>/mcp  
  \- 승인 URL: https://accounts.google.com/o/oauth2/v2/auth  
  \- 승인 URL 매개변수: \&access\_type=offline  
  \- 토큰 URL: https://oauth2.googleapis.com/token  
  \- 클라이언트 ID / 비밀번호: Task 1에서 발급받은 정보  
  \- 범위 (Scopes): [https://www.googleapis.com/auth/cloud-platform](https://www.googleapis.com/auth/cloud-platform)

\`\&access\_type=offline\` 를 넣는이유   
쉽게 말해서 Refresh Token 발급

인증 요청 시 이 파라미터를 넣으면, 서버는 Access Token 외에 Refresh Token이라는 특별한 토큰을 하나 더 줍니다.  
4\. 로그인 버튼을 눌러 권한을 승인한 후 저장합니다.

5\. Description 란에 아래 텍스트를 입력합니다:

| This MCP server connects to Atlassian services (Jira and Confluence). It provides tools to read and search Confluence pages, and create new pages. It also includes tools to search, retrieve, and create Jira issues. Use this server to analyze planning documents in Confluence and manage tasks in Jira. |
| :---- |

6\. Agent Instructions에 다음 내용을 추가합니다:

| 1\. For read-only actions (searching or reading Confluence pages and Jira issues), proceed immediately without asking for user confirmation.2\. When asked to CREATE a Jira ticket or a Confluence page, you MUST generate a draft of the content first and ask the user for confirmation before executing the creation tool.3\. If a space key or project key is not provided by the user, use list\_confluence\_spaces or list\_jira\_projects to find the appropriate key, and confirm with the user before proceeding with that key. |
| :---- |

7\. Create를 눌러 생성합니다.

# **Task 5\. Add Permissions To Cloud Run**

1. 검색에 Cloud Run 을 검색하여 진입 후 Service Click  
2. 해당 인스턴스 왼쪽 checkbox를 click하여 permissions로 진입![][image4]  
3. “Add Principal” click   
   “New principals” : “allUsers”  
   “Role” : “Cloud Run Invoker”**![][image5]**  
 


# **Task 5\. Active Custom MCP method**

1. App \> Data stores \> 해당 custom MCP click  
2. Actions \> Refresh Custom Actions를 클릭하여 도구를 활성화합니다.

![][image6]

# **Task 6\. Automate workflows with Gemini**

1\. Gemini 채팅 인터페이스로 이동합니다.

2\. 다음 프롬프트를 입력하고 테스트합니다:  
컨플루언스에서 '카메라 개발 계획' 문서를 검색해서 읽은 뒤, 그 내용을 바탕으로 Jira에 티켓들을 생성해줘.
