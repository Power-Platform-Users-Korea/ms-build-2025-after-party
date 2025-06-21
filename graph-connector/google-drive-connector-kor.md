# Google Drive를 Copilot(및 검색)용 Microsoft Graph에 연결하기

이 가이드는 Google Drive 인스턴스를 Microsoft Graph에 연결하여 Microsoft Copilot 및 Microsoft Search와 같은 서비스가 Google Drive 콘텐츠에 액세스하고 활용할 수 있도록 하는 단계별 프로세스를 제공합니다. 이는 Google Workspace와 Microsoft 365를 모두 사용하는 조직이 통합된 정보 액세스 및 AI 기능을 달성하는 데 매우 중요합니다.

### Gemini (또는 다른 AI 도우미)를 효과적으로 사용하는 일반적인 힌트

이 가이드 전반에서 문제가 발생하거나 추가 설명이 필요할 수 있습니다. AI를 효과적으로 사용하여 도움을 받는 방법에 대한 몇 가지 일반적인 팁은 다음과 같습니다:

  * **구체적으로 작성:** 문제에 대해 더 자세한 정보를 제공할수록 AI의 응답이 더 좋아집니다.
  * **오류 메시지 포함:** 항상 전체 오류 메시지를 복사하여 붙여넣으세요. 여기에는 중요한 진단 정보가 포함되어 있습니다.
  * **컨텍스트 제공:** 무엇을 하려고 했는지, 무엇을 예상했는지, 그리고 대신 어떤 일이 발생했는지 설명합니다.
  * **원하는 출력 지정:** 코드, 특정 형식(Markdown 또는 YAML과 같은), 또는 단계별 목록을 원하면 요청하세요.
  * **반복:** 첫 번째 응답이 완벽하지 않다면, 더 많은 정보나 다른 관점으로 프롬프트를 다듬으세요.

**일반적인 도움을 위한 프롬프트 예시:**

  * "이 오류가 발생했습니다: `[여기에 전체 오류 메시지 붙여넣기]`. 무슨 뜻이며 어떻게 해결할 수 있습니까?"
  * "`[개념 이름]`을 간단하게 설명해 주세요."
  * "`[도구/서비스 이름]`을 사용하여 `[동작 수행]`하는 방법은 무엇입니까?"
  * "`[목표 달성]`을 위한 `gcloud` 명령어를 알려주세요. 각 부분도 설명해 주세요."

### 전제 조건

시작하기 전에 다음을 확인하세요:

1.  **Google Workspace 도메인:** 활성화된 Google Workspace 도메인(예: `yourcompany.com`).
2.  **Google Cloud Platform (GCP) 액세스:** 프로젝트, 서비스 계정 및 조직 정책을 관리할 수 있는 권한이 있는 Google Cloud 계정. 이상적으로는 Google Workspace 도메인의 Super Administrator여야 합니다. 이는 GCP에서 권한 관리를 간소화합니다.
3.  **Microsoft 365 관리 센터 액세스:** Microsoft 365 테넌트에서 전역 관리자 또는 검색 관리자 역할이 있는 계정.

### 단계별 가이드

#### 1\. Google Cloud Platform (GCP) 설정

이 단계에서는 필요한 Google Cloud 리소스 및 권한을 설정합니다.

1.  **GCP 프로젝트 생성:**

      * [Google Cloud Console](https://console.cloud.google.com/)으로 이동합니다.

      * 새 프로젝트를 생성합니다 (예: `google-drive-connector`).

      * 상위 조직/폴더를 선택하라는 메시지가 표시되고 '조직없음'이 표시되면, 공식적인 GCP 조직 리소스가 없는 경우 "조직없음"으로 진행합니다.

      * **Gemini 힌트:** '조직없음'의 의미가 불확실하거나 GCP의 리소스 계층 구조를 더 잘 이해하고 싶다면:

          * "Google Cloud 리소스 계층 구조(조직, 폴더, 프로젝트)와 '조직없음'으로 프로젝트를 생성하는 것의 의미를 설명해주세요."

2.  **필수 API 활성화:**

      * 새 프로젝트에서 **API 및 서비스 \> 라이브러리**로 이동합니다.

      * 다음 API를 활성화합니다:

          * **Google Drive API**
          * **Admin SDK API**

      * **Gemini 힌트:** API를 찾을 수 없거나 올바른 API인지 확실하지 않은 경우:

          * "제 도메인의 사용자를 대신하여 Google Drive의 모든 파일을 읽으려면 어떤 Google Cloud API를 활성화해야 합니까?"

3.  **서비스 계정 생성:**

      * **IAM 및 관리자 \> 서비스 계정**으로 이동합니다.
      * `+ 서비스 계정 만들기`를 클릭합니다.
      * 의미 있는 이름을 지정합니다 (예: `ms-graph-drive-connector`).
      * **역할 부여:** 생성 시 "뷰어" 또는 특정 "Drive" 역할을 부여하는 것을 고려할 수 있지만, 중요한 권한은 다음 단계의 도메인 전체 위임에서 나옵니다.

4.  **서비스 계정 키 (JSON) 생성:**

      * "서비스 계정" 페이지에서 새 서비스 계정의 이메일 주소를 클릭합니다.

      * **키** 탭으로 이동합니다.

      * `키 추가 > 새 키 만들기`를 클릭합니다.

      * `JSON`을 선택하고 `만들기`를 클릭합니다.

      * **JSON 파일을 다운로드하고 즉시 안전하게 보관하세요.** 이 파일에는 비공개 키가 포함되어 있습니다.

      * **문제 해결: `서비스 계정 키 생성이 사용 중지되었습니다` 오류**

          * **문제:** `조직에서 서비스 계정 키 생성을 차단하는 조직 정책이 적용되었습니다.` (`iam.disableServiceAccountKeyCreation` 또는 `iam.managed.disableServiceAccountKeyCreation`)와 같은 오류가 발생합니다. 이는 조직 수준의 보안 정책에 의해 직접 키 생성이 차단되었음을 의미합니다.
          * **해결책:** Google Cloud **조직 수준**(`lawbot4.me`의 경우)에서 `Organization Policy Administrator` 역할이 필요합니다.
            1.  **IAM 및 관리자 \> IAM**으로 이동합니다 (상단 드롭다운에서 범위가 **조직**으로 설정되어 있는지 확인합니다).
            2.  계정에 조직 수준에서 `Organization Policy Administrator` 또는 `Owner` 역할이 부여되어 있는지 확인합니다. 그렇지 않은 경우 (Google Workspace Super Administrator인 경우) 자신에게 해당 역할을 부여합니다.
            3.  **IAM 및 관리자 \> 조직 정책**으로 이동합니다 (범위가 **조직**으로 설정되어 있는지 확인합니다).
            4.  **정확한 차단 정책 식별:** Cloud Shell에서 `gcloud org-policies describe`를 사용하여 `enforce: true`인 활성 정책을 찾습니다.
                  * `constraints/iam.disableServiceAccountKeyCreation` 및 `constraints/iam.managed.disableServiceAccountKeyCreation` 모두 확인합니다. 오류 메시지가 어떤 정책이 활성화되어 있거나 선호되는지 안내할 것입니다.
                  * 예시 `describe` 명령어:
                    ```bash
                    gcloud org-policies describe constraints/iam.disableServiceAccountKeyCreation --organization=YOUR_ORGANIZATION_ID --format=yaml
                    # 또는 관리형 버전에 대해:
                    gcloud org-policies describe constraints/iam.managed.disableServiceAccountKeyCreation --organization=YOUR_ORGANIZATION_ID --format=yaml
                    ```
                      * **출력에서 `etag`와 전체 `name` (예: `organizations/YOUR_ORGANIZATION_ID/policies/...` 포함)을 복사합니다.**
            5.  **Cloud Shell에서 정책 YAML 생성/편집:**
                  * Cloud Shell을 엽니다.
                  * `nano` (또는 `vi`)를 사용하여 YAML 파일을 생성합니다 (예: `allow_sa_key.yaml`):
                    ```bash
                    nano allow_sa_key.yaml
                    ```
                  * `enforce: false` 및 `describe` 명령의 \*\*정확한 `name` 및 `etag`\*\*를 확인하여 콘텐츠를 붙여넣습니다:
                    ```yaml
                    name: organizations/YOUR_ORGANIZATION_ID/policies/iam.disableServiceAccountKeyCreation # 또는 iam.managed.disableServiceAccountKeyCreation
                    spec:
                      etag: YOUR_ETAG_FROM_DESCRIBE_COMMAND
                      rules:
                        - enforce: false
                    ```
                  * `nano`를 저장하고 종료합니다 (`Ctrl+O`, Enter, `Ctrl+X`).
                  * **Gemini 힌트:** `nano` 또는 `vi`에 어려움이 있거나 YAML 구문을 확인해야 하는 경우:
                      * "Cloud Shell에서 `nano`를 저장하고 종료하는 방법은 무엇입니까?"
                      * "이 YAML 구문은 Google Cloud 조직 정책에 대해 올바른가요: `[YAML 붙여넣기]`?"
            6.  **정책 변경 적용:**
                  * `gcloud` 프로젝트 컨텍스트가 액세스 권한이 있는 유효한 프로젝트로 설정되어 있는지 확인합니다: `gcloud config set project YOUR_VALID_PROJECT_ID`.
                  * `set-policy` 명령을 실행합니다 (YAML에 `name`이 있으면 `--organization` 또는 `--project`는 필요 없습니다):
                    ```bash
                    gcloud org-policies set-policy allow_sa_key.yaml
                    ```
                  * 전파를 위해 몇 분 정도 기다린 다음 키 생성을 다시 시도합니다.

5.  **도메인 전체 위임 구성 (매우 중요):**

      * 이 단계는 서비스 계정에 사용자를 대신하여 Drive 데이터에 액세스할 수 있는 권한을 부여합니다.

      * 도메인 (`lawbot4.me`)의 Super Administrator로 **Google Workspace 관리 콘솔**(`admin.google.com`)에 로그인합니다.

      * **보안 \> 액세스 및 데이터 제어 \> API 컨트롤**로 이동합니다.

      * "도메인 전체 위임" 섹션에서 **도메인 전체 위임 관리**를 클릭합니다.

      * **새로 추가**를 클릭합니다.

      * **클라이언트 ID:** Google Cloud 서비스 계정의 \*\*고유 ID (클라이언트 ID)\*\*를 붙여넣습니다.

      * **OAuth 범위:** 다음 범위를 쉼표로 구분하여 추가합니다 (공백 없음):
        `https://www.googleapis.com/auth/drive.readonly,https://www.googleapis.com/auth/admin.directory.user.readonly,https://www.googleapis.com/auth/admin.directory.group.readonly`

      * `승인`을 클릭합니다.

      * **Gemini 힌트:** 특정 OAuth 범위가 올바른지 또는 어떤 권한을 부여하는지 확실하지 않은 경우:

          * "Google OAuth 범위 `https://www.googleapis.com/auth/drive.readonly`는 서비스 계정에 무엇을 할 수 있도록 허용합니까?"
          * "Google Cloud 서비스 계정의 클라이언트 ID를 어떻게 찾나요?"

#### 2\. Microsoft Graph 커넥터 설정

이 단계에서는 Microsoft 365 테넌트에서 커넥터를 구성합니다.

1.  **Microsoft 365 관리 센터 액세스:**

      * `admin.microsoft.com`으로 이동합니다.
      * **설정 \> 검색 및 인텔리전스 \> 커넥터**로 이동합니다.
      * **Google Drive 커넥터**를 찾아 선택합니다.

2.  **커넥터 세부 정보 입력 (설정 탭):**

      * **표시 이름:** `Google Drive` (또는 사용자에게 설명적인 이름).

      * **Google 앱 도메인:** Google Workspace 도메인 (예: `lawbot4.me`).

      * **인증 유형:** `Google 서비스 계정` (기본값이어야 합니다).

      * **Google 앱 관리자 이메일:** Google Workspace 도메인의 Super Administrator 이메일 주소 (예: `admin@lawbot4.me`).

      * **서비스 계정 키:** 이전에 다운로드한 **JSON 키 파일의 전체 내용**을 붙여넣습니다.

      * **문제 해결: 권한 부여 후 "문제가 발생했습니다" 오류**

          * **문제:** 키를 붙여넣고 "승인"을 클릭하면 일반적인 오류가 발생합니다.
          * **해결책:** 이는 거의 항상 \*\*도메인 전체 위임 (1.5단계)\*\*이 올바르게 구성되지 않았음을 의미하며, 특히 필요한 OAuth 범위 중 하나 이상이 누락되거나 잘못되었습니다. 1.5단계를 매우 주의 깊게 다시 확인하십시오.
          * **Gemini 힌트:** 이 오류가 발생했고 1.5단계를 다시 확인했다면:
              * "Microsoft Graph Google Drive 커넥터 설정에서 '문제가 발생했습니다'라는 오류가 발생하고 있습니다. 다음 범위로 도메인 전체 위임을 구성했습니다: `[범위 나열]`. 또 무엇이 잘못되었을 수 있습니까?"

3.  **제한된 대상에게 배포:**

      * 초기 테스트를 위해 **이 토글을 ON으로 설정**하는 것이 좋습니다. 나중에 특정 Microsoft 365 그룹을 테스트 대상으로 선택할 수 있습니다. 이는 광범위한 배포 전에 기능의 유효성을 검사하는 데 매우 권장됩니다.

4.  **나머지 탭 완료:**

      * `다음`을 클릭하고 나머지 탭을 진행합니다:
          * **사용자:** Google Workspace 사용자/그룹을 Microsoft Entra ID (Azure AD) ID에 매핑하여 올바른 권한이 존중되도록 합니다.
          * **데이터:** Google Drive에서 포함/제외할 콘텐츠를 정의합니다 (예: 특정 폴더, 파일 형식).
          * **크롤링:** 전체 및 증분 크롤링 일정을 설정합니다.

### 설정 후 및 확인

  * **크롤링 상태 모니터링:** Microsoft 365 관리 센터에서 커넥터 상태를 모니터링합니다. 대용량 Drive 인스턴스의 경우 초기 전체 크롤링에 시간이 오래 걸릴 수 있습니다.

  * **검색 테스트:** 인덱싱이 완료되면 Microsoft Search (예: office.com 또는 SharePoint 검색)에서 Google Drive 콘텐츠를 검색해 봅니다.

  * **Copilot 테스트:** Copilot이 활성화된 경우 Google Drive에만 있는 콘텐츠와 관련된 질문을 해봅니다.

  * **보안 모범 사례:** 성공적으로 설정한 후 GCP에서 `iam.disableServiceAccountKeyCreation` 또는 `iam.managed.disableServiceAccountKeyCreation` 정책을 비활성화한 경우, 향상된 보안을 위해 **다시 활성화**하는 것을 고려하십시오. 필요한 경우 나중에 더 안전한 내부 조직 프로세스를 통해 서비스 계정 키를 관리할 수 있습니다.

      * **Gemini 힌트:** 정책을 다시 활성화해야 하는 경우:
          * "`gcloud` 및 YAML을 사용하여 Google Cloud 조직 정책 `iam.disableServiceAccountKeyCreation`을 다시 활성화하는 방법은 무엇입니까?"

-----

### P.S. Gemini를 위한 고급 문제 해결 프롬프트

막다른 골목에 부딪혀도 포기하지 마세요\! 이 프롬프트는 이 통합 과정에서 직면할 수 있는 특정 문제를 해결하기 위해 Gemini가 더 깊이 파고들도록 설계되었습니다. 받은 **정확한 오류 메시지를 복사하여 붙여넣는 것을 잊지 마세요.**

**1. `gcloud org-policies set-policy`가 `INVALID_ARGUMENT: Project 'projects/...' not found or deleted.` 오류와 함께 실패할 때:**
\* "제 `gcloud org-policies set-policy` 명령이 `INVALID_ARGUMENT: Project 'projects/[PROJECT_ID]' not found or deleted.` 오류와 `reason: USER_PROJECT_DENIED`와 함께 실패하고 있습니다. 조직 정책을 설정하려고 합니다. 조직 정책의 이 컨텍스트에서 `USER_PROJECT_DENIED`가 의미하는 바는 무엇이며, 기본 프로젝트 컨텍스트를 수정하기 위해 실행해야 하는 정확한 `gcloud` 명령은 무엇입니까?"
\* "조직 정책에 대해 `gcloud org-policies set-policy`를 실행하고 있습니다. `INVALID_ARGUMENT: Project 'projects/my-old-project' not found` 오류가 발생합니다. 특히 조직 정책 관리를 위해 `gcloud`가 다른 유효한 프로젝트를 사용하도록 강제하는 방법은 무엇입니까?"

**2. `gcloud org-policies describe` 또는 `set-policy`가 정책 버전 불일치를 나타낼 때:**
\* "Google Cloud 조직 정책을 업데이트하려고 합니다. `gcloud org-policies set-policy`에서 `constraints/iam.disableServiceAccountKeyCreation`이 이전 버전이며 `constraints/iam.managed.disableServiceAccountKeyCreation`을 제안한다는 경고가 표시됩니다. 그런 다음 `ABORTED`되었습니다. `managed` 버전이 제 조직에서 활성화되어 있는지 확인하고, 해당 ETAG를 가져오고, 버전과 상관없이 *현재 적용된* 정책을 성공적으로 비활성화하도록 YAML을 업데이트하기 위해 실행해야 하는 정확한 `gcloud` 명령은 무엇입니까?"

**3. MS Graph 커넥터 설정에서 '승인'이 GCP 구성이 올바른 것처럼 보인 후에도 실패할 때:**
\* "Google 서비스 계정 키를 Microsoft Graph Google Drive 커넥터 설정에 올바르게 복사했습니다. Google Workspace 관리자 이메일도 정확하게 입력했습니다. '승인'을 클릭하면 일반적인 '문제가 발생했습니다' 오류가 발생하며, Microsoft로부터 특정 오류는 없습니다. 도메인 전체 위임 및 서비스 계정의 OAuth 범위와 관련하여 Google Workspace 관리 콘솔에서 즉시 확인해야 할 상위 3가지 구체적인 사항은 무엇입니까? `admin.google.com`의 정확한 경로를 제공해 주세요."
\* "Microsoft Graph Google Drive 커넥터의 인증이 실패하고 있습니다. JSON 키와 관리자 이메일을 확인했습니다. OAuth 범위가 누락되었을 수 있다고 생각합니다. 도메인 전체 위임을 통해 Microsoft Graph Google Drive 커넥터를 활성화하기 위해 Google Cloud 서비스 계정에 필요한 *모든* 특정 OAuth 범위(사용자 및 그룹 디렉토리 읽기용 포함)를 나열해 주시겠습니까?"

**4. 서비스 계정 키가 생성되었지만 MS 커넥터 유효성 검사가 여전히 실패할 때:**
\* "조직 정책을 비활성화한 후 GCP에서 Google 서비스 계정 키를 성공적으로 생성했습니다. 전체 JSON 내용을 Microsoft Graph 커넥터 설정에 붙여넣었습니다. 인증 중에 여전히 오류가 발생합니다. Google Cloud 콘솔에서 유효하다고 표시되더라도 Microsoft Graph와 같은 외부 서비스가 유효한 JSON 서비스 계정 키로 인증에 실패할 수 있는 덜 명확한 이유는 무엇입니까?"

**5. 커넥터에 대한 GCP/Google Workspace 설정에 대한 일반 진단:**
\* "Microsoft Graph Google Drive 커넥터용 Google Cloud 서비스 계정에서 인증 오류가 발생하고 있지만, 특정 오류 메시지가 명확하지 않습니다. Google Cloud Console(API, 서비스 계정 세부 정보)과 Google Workspace 관리 콘솔(도메인 전체 위임, 관리자 역할)에서 이 유형의 통합에 대해 설정이 완전히 올바른지 확인하기 위한 중요한 항목 체크리스트를 제공해 주시겠습니까?"

**END KOREAN GUIDE**

-----
