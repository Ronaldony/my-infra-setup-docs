# Obsidian Local REST API 기반 MCP 구축 및 원격 Gateway 연동

## 1. 목적과 검증 범위

Obsidian Vault의 Markdown 노트를 원격 MCP 클라이언트에서 검색·조회·수정할 수 있도록 Local REST API와 Python MCP 어댑터를 연결했다. 외부 클라이언트에는 Vault나 REST API를 직접 공개하지 않고, 인증된 FastMCP Gateway와 Secure MCP Tunnel을 통해 접근하도록 구성했다.

실제 검증 대상은 **단일 Vault의 독립 실행 인스턴스 하나**였다. 같은 MCP 어댑터를 별도 설정으로 여러 번 실행하는 설계는 가능하지만, 다중 Vault 동시 운영을 검증한 것으로 확대하지 않는다.

| 구성요소 | 검증 시 기술 기준 | 역할 |
|---|---|---|
| Obsidian + Local REST API | Local REST API 4.1.3 | 실행 중인 Vault에 HTTP REST 인터페이스 제공 |
| Python MCP 어댑터 | `mcp-obsidian` 0.2.2, Python 3.11 이상, MCP SDK 1.x | REST API를 MCP 도구로 변환 |
| 패키지 관리 | `uv`와 프로젝트별 가상환경 | 어댑터의 독립적인 실행환경 관리 |
| FastMCP Gateway | FastMCP 4.0.11, `create_proxy`의 `legacy` 모드 | MCP stdio upstream 연결 및 인증된 도구 호출 중계 |
| 원격 접속 | Secure MCP Tunnel Client 0.0.15 | 로컬 Gateway를 외부 MCP 클라이언트에 연결 |
| 인증·공개 분리 | OAuth, Auth0, Tailscale Funnel | 인증 절차와 MCP 데이터 경로를 분리 |

버전은 **해당 작업에서 확인한 기준선**이지 다른 버전의 호환성을 보장하는 범위가 아니다. OAuth 자체의 정책과 검증 세부사항은 [FastMCP OAuth·Secure MCP Tunnel 문서](../fastmcp/fastmcp-oauth-secure-tunnel-authorization.md)에서 다룬다.

---

## 2. 구성요소와 신뢰 경계

최종 구조에서 `mcp-obsidian`은 Vault 파일을 직접 읽지 않고 Obsidian Local REST API에 HTTP 요청을 전달한다. FastMCP Gateway는 stdio MCP를 upstream으로 실행하며, 원격 접속과 인증은 별도 계층에서 처리한다.

~~~mermaid
flowchart TB
    Client["원격 MCP Client"]
    Tunnel["Secure MCP Tunnel"]
    Gateway["OAuth-protected FastMCP Gateway<br/>127.0.0.1:8765/mcp"]
    Adapter["mcp-obsidian<br/>MCP stdio process"]
    REST["Obsidian Local REST API<br/>127.0.0.1:27123"]
    Vault["Obsidian Vault<br/>Markdown notes"]

    OAuthEdge["OAuth Edge<br/>127.0.0.1:8770"]
    Funnel["Tailscale Funnel<br/>HTTPS 443"]
    Auth["Auth0"]

    Client <--> Tunnel
    Tunnel <--> Gateway
    Gateway <-->|"MCP / stdio"| Adapter
    Adapter <-->|"HTTP + Bearer API Key"| REST
    REST <--> Vault

    Client <-. "OAuth discovery / login" .-> Funnel
    Funnel <--> OAuthEdge
    OAuthEdge <--> Auth

    classDef local fill:#eef4ff,stroke:#5681bd,color:#23365a
    class Gateway,Adapter,REST,OAuthEdge local
~~~

| 경계 | 적용한 원칙 |
|---|---|
| REST API ↔ Vault | 로컬 Obsidian 프로세스가 Vault의 실제 파일 접근을 담당 |
| MCP 어댑터 ↔ REST API | API Key 기반 Bearer 인증; HTTP는 loopback에서만 사용 |
| Gateway ↔ MCP 어댑터 | stdio 연결, 별도 Python 환경 사용, upstream 코드·의존성 분리 |
| 원격 Client ↔ Gateway | Gateway는 loopback 수신, Tunnel을 통해 연결 |
| 외부 OAuth ↔ 내부 MCP | Funnel은 OAuth Edge만 공개; REST API 및 `/mcp`를 직접 공개하지 않음 |

원격 Tunnel 하나는 Gateway의 연결 식별자로 사용한다. **MCP 서버별로 별도 Tunnel을 실행하지 않으며**, Vault별 API 접속정보와 프로세스는 Gateway 연결과 독립적으로 관리한다. Tunnel 식별자 자체를 MCP별 권한 격리 수단으로 취급하지 않는다.

---

## 3. Obsidian REST API 및 인스턴스 구성

### 3.1 플러그인 설정과 네트워크 제한

Vault에 설치된 Local REST API 커뮤니티 플러그인에서 HTTP 서버를 사용하고, 최종적으로 다음 값을 적용했다.

| 설정 | 적용 값 | 의미 |
|---|---|---|
| HTTP server | Enabled | MCP 어댑터가 로컬 HTTP로 접근 |
| Binding host | `127.0.0.1` | 다른 네트워크 인터페이스에서의 직접 수신 방지 |
| HTTP port | `27123` | MCP 인스턴스의 REST API 대상 포트 |
| API Key | 플러그인에서 발급 | REST API 요청 인증 |

최초 점검에서는 서버가 `0.0.0.0:27123`에서 수신했지만, 최종 구성에서는 `127.0.0.1:27123`만 수신하도록 변경했다. PowerShell `Get-NetTCPConnection`에서 루프백 수신을 확인했고, `Test-NetConnection` 및 인증된 REST 호출도 성공했다.

~~~powershell
Get-NetTCPConnection -State Listen -LocalPort 27123 |
    Select-Object LocalAddress, LocalPort, OwningProcess

Test-NetConnection 127.0.0.1 -Port 27123
~~~

TCP 연결 가능 여부는 **서비스 수신 확인**일 뿐 API 인증이나 MCP 도구 성공을 뜻하지 않는다. 따라서 Bearer 인증 요청과 MCP 도구 호출을 별도로 검증했다.

### 3.2 독립 MCP 실행 설정

API 접속정보는 인스턴스별 설정으로 분리하고, 실행 진입점이 환경변수를 채운 뒤 MCP 어댑터를 불러오도록 구성했다.

~~~dotenv
OBSIDIAN_PROTOCOL=http
OBSIDIAN_HOST=127.0.0.1
OBSIDIAN_PORT=27123
OBSIDIAN_API_KEY=<LOCAL_REST_API_KEY>
~~~

이는 **설정 형식 예시**이며 실제 API Key는 기록하지 않는다. 비밀정보를 포함하는 설정은 Git 관리에서 제외하고 파일 접근권한을 제한했다.

MCP 서버의 API Key·호스트 설정은 모듈 초기화 시 읽히므로, **환경변수를 서버 모듈 import보다 먼저 설정**해야 한다. 다른 Vault가 필요하면 동일 코드베이스를 공유하되 각 실행 인스턴스가 자신의 접속정보를 사용하도록 분리하는 구조를 유지한다.

### 3.3 실행환경과 의존성

MCP 어댑터는 Gateway의 Python 환경에 합쳐 설치하지 않고 별도 가상환경에서 실행했다. 검증된 패키지 조건에는 `mcp>=1.1.0,<2.0.0`이 포함된다.

작업 중 upstream `uv.lock`과 프로젝트 의존성 선언 사이에서 `uv sync --locked` 불일치가 확인되어, 기존 잠금파일을 자동 변경하지 않고 `--frozen` 방식으로 환경을 유지·검증했다.

~~~powershell
uv sync --frozen
uv run --frozen pytest -q
uv run --frozen pyright
~~~

---

## 4. FastMCP Gateway 및 Secure Tunnel 연결

Gateway에서는 MCP 실행 파일을 단독 서버로 외부 공개하지 않고, **stdio upstream**으로 등록했다. MCP SDK 1.x 서버에 대응하기 위해 FastMCP proxy backend를 `legacy` 모드로 구성했다.

~~~python
# 연결 형태의 핵심만 표시한 예시.
upstream = {
    "mcpServers": {
        "notes": {
            "command": "<VENV_PYTHON>",
            "args": ["<INSTANCE_RUNNER>"],
        }
    }
}
gateway = create_proxy(upstream, mode="legacy", auth=auth)
~~~

Gateway는 loopback의 MCP HTTP endpoint를 제공하며, Tunnel Client가 그 endpoint에 연결한다. Tunnel Client 바이너리와 운영 실행 스크립트를 분리하고 **단일 실행 진입점**에서 프로필·Tunnel ID·OAuth 보호 상태·`doctor` 진단·중복 프로세스를 점검했다. 원격 식별자 변경 후에는 이전 프로세스를 중지하고 새 프로필로 다시 연결해 도구 호출을 확인했다.

OAuth 인증정보와 REST API Key는 서로 다른 권한 경계에 속한다. 외부 HTTPS OAuth Edge는 Tailscale Funnel로 연결하지만, 해당 Edge에서 공개 `/mcp` 경로는 HTTP 404가 반환되도록 유지했다. 로컬 Gateway의 인증 없는 MCP 요청은 HTTP 401로 거부됐다.

이 단락은 **Obsidian MCP가 Gateway 뒤에서 동작하는 접점**만 설명한다. Gateway의 복수 upstream 관리와 OAuth 상세 구현은 각각 [Gateway 구성 문서](../fastmcp/fastmcp-multi-upstream-headless-gateway.md) 및 [OAuth 연결 문서](../fastmcp/fastmcp-oauth-secure-tunnel-authorization.md)를 참조한다.

---

## 5. MCP 요청 흐름과 제공 기능

~~~mermaid
sequenceDiagram
    participant C as Remote MCP Client
    participant T as Secure Tunnel
    participant G as FastMCP Gateway
    participant M as mcp-obsidian
    participant R as Local REST API
    participant V as Vault

    C->>T: tools/call(name, arguments)
    T->>G: MCP 요청 전달
    G->>M: tools/call over stdio
    M->>R: HTTP /vault/{encoded-path}<br/>Authorization: Bearer API Key
    R->>V: Markdown 조회 또는 변경
    V-->>R: 실제 파일 결과
    R-->>M: HTTP 응답
    M-->>G: MCP Tool 결과
    G-->>T: 인증된 응답
    T-->>C: Tool 결과
~~~

MCP에서는 최종적으로 **15개 Tool이 등록**되는 것을 확인했다.

| 기능 범위 | 주요 도구 |
|---|---|
| 파일 탐색·읽기 | `obsidian_list_files_in_vault`, `obsidian_list_files_in_dir`, `obsidian_get_file_contents`, `obsidian_batch_get_file_contents` |
| 검색·메타데이터 | `obsidian_simple_search`, `obsidian_complex_search`, `obsidian_search_by_tag`, `obsidian_get_frontmatter` |
| 수정·삭제 | `obsidian_append_content`, `obsidian_patch_content`, `obsidian_put_content`, `obsidian_delete_file` |
| 주기 노트·변경 이력 | `obsidian_get_periodic_note`, `obsidian_get_recent_periodic_notes`, `obsidian_get_recent_changes` |

등록 여부와 실사용 가능 여부는 구분했다. 파일 탐색·검색·수정 기능은 시험 노트로 검증했지만, 최근 변경 및 주기 노트 기능은 현재 Vault 환경의 추가 조건 때문에 별도 제약이 확인됐다.

---

## 6. 실사용 검증 방법과 결과

검증에서는 기존 노트를 변경하지 않고 **고유한 임시 시험 노트만 생성·수정·삭제**했으며, 사용 후 파일과 시험 디렉터리가 제거됐는지 확인했다.

### 6.1 자동 테스트와 로컬 MCP 검증

| 검증 | 실제 결과 |
|---|---|
| 기존 테스트 | 81개 통과 |
| 특수 경로 인코딩 회귀 테스트 추가 | 35개 통과 |
| 최종 `pytest` | **116개 통과** |
| `pyright` | 오류 0개, 경고 0개 |
| 로컬 stdio MCP | 초기화·15개 Tool discovery 성공 |
| 일반 기능 | 한글 파일명/본문, 생성·조회·append·patch·put·delete, heading·frontmatter, 검색·태그·일괄 읽기 성공 |
| 부정 시험 | 모호하거나 없는 Heading, 확인 없는 삭제, 잘못된 입력에서 오류 반환 및 세션 유지 확인 |

로컬 안정성 표본으로 순차 읽기 20회·동시 읽기 8회가 일관된 결과를 반환했다. UTF-8 노트 약 144 KB도 정확히 왕복했다. 이때 순차 로컬 stdio 읽기 지연의 중앙값은 약 11.58 ms, 최대값은 약 38.20 ms였다.

이는 **짧은 로컬 표본**일 뿐 원격 Tunnel 지연, 장시간 가용성 또는 동시 쓰기의 안정성을 보증하지 않는다.

### 6.2 원격 클라이언트까지 검증

인증된 원격 MCP 클라이언트에서 실제 Vault 파일 목록·노트 읽기를 수행하고, 시험 노트의 생성·Heading/Frontmatter 수정·태그 조회·재조회·삭제까지 확인했다.

특수 파일명 수정 전에는 원격 호출이 다른 시험 파일을 읽는 오류를 재현했으며, 수정된 MCP 하위 프로세스로 갱신한 뒤 같은 경로에서 올바른 파일을 반환하는 것도 확인했다. 최종적으로 원격 PUT·GET·DELETE 후 **대상 파일만 변경·삭제되고 다른 시험 파일은 유지**된 것을 디스크에서 대조했다.

도구 등록, HTTP 200, Tunnel readiness만으로 최종 성공이라 판단하지 않고 **실제 Vault 파일의 결과와 일치하는지**를 완료 기준으로 삼았다.

---

## 7. 주요 이슈: URL 경로의 특수문자로 인한 잘못된 파일 덮어쓰기

### 7.1 이슈 내용과 발견 시점

실제 end-to-end 시험에서 파일명에 `#`이 포함된 시험 노트를 PUT 요청으로 저장할 때, MCP Tool은 성공을 반환했지만 **요청한 파일이 생성되지 않고 다른 시험 파일이 덮어써지는 오류**가 확인됐다.

예를 들어 다음 경로는 서로 다른 노트를 의미해야 한다.

~~~text
notes/collision.md
notes/collision.md#suffix.md
~~~

### 7.2 원인 추적

기존 어댑터는 파일명을 인코딩하지 않고 REST URL 문자열에 직접 끼워 넣었다.

~~~text
요청한 파일:       notes/collision.md#suffix.md
잘못된 HTTP 경로: /vault/notes/collision.md
정상 인코딩 경로: /vault/notes/collision.md%23suffix.md
~~~

URL의 `#` 뒤 문자열이 fragment로 취급되어 서버에 전달되지 않는 것이 원인이었다. 실제 REST request target을 확인했고, 동일 API에 올바르게 인코딩된 경로를 사용하면 원래 파일과 대상 파일이 별개로 유지됨을 검증했다.

### 7.3 해결 과정

경로를 구성하는 기능을 `_vault_url()`로 단일화했다.

~~~python
from urllib.parse import quote

def _vault_url(self, path: str) -> str:
    return f"{self.get_base_url()}/vault/{quote(path, safe='/')}"
~~~

여기서 `safe='/'`는 Vault 내부 디렉터리 구분자만 유지하고, `#`·`?`·`%`·공백 및 UTF-8 파일명 등 URL에 특별한 의미가 있는 문자를 인코딩한다.

적용 범위는 PUT 하나가 아니라 **디렉터리 목록, 단일 파일 읽기, append, patch, PUT, delete, frontmatter 조회**의 Vault 상대 경로를 사용하는 모든 동작이었다.

### 7.4 검증

- 5종 특수 경로 × 7종 동작 = **35개 신규 회귀 테스트 통과**
- 로컬 MCP에서 특수 파일명과 일반 파일명을 동시에 생성해 내용 독립성 확인
- 한글과 `#`이 들어간 폴더, `%`를 포함한 파일명에서 조회 성공
- 실행 중이던 Gateway는 그대로 두고 오래된 Obsidian MCP 하위 프로세스만 갱신
- 원격 MCP에서 특수 파일명에 PUT·GET·DELETE 수행 후 잘못된 형제 파일이 유지됨을 확인
- 시험 파일·빈 디렉터리 정리 확인

**결과:** 잘못된 파일을 조작하던 고위험 경로 처리 결함을 수정했고, 실제 로컬 MCP 및 인증된 원격 MCP 호출에서 회귀 검증했다. 기존 사용자 노트는 수정하지 않았다.

---

## 8. 운영 시 확인된 제약

| 항목 | 확인된 동작과 운영상 의미 |
|---|---|
| `put_content` | 기존 노트 전체를 덮어쓰므로 사용 전 대상 경로와 본문을 확인해야 함 |
| `append_content` | 동일 요청을 두 번 보내면 본문이 중복되므로 응답이 불명확할 때 먼저 재조회해야 함 |
| `patch_content` | Heading 경로가 모호하면 오류가 발생함. 가능한 경우 전체 Heading 경로 또는 특정 frontmatter 필드를 지정 |
| 일괄 읽기 | 요청 중 일부 파일이 없어도 결과 전체의 `isError=false`일 수 있고 본문 안에 `Error reading file:`이 포함됨. 파일별 결과 확인 필요 |
| 최근 변경 조회 | Dataview DQL Content-Type을 사용하는 요청이 HTTP 40012로 거부됨. 검증 Vault에서 Dataview가 활성화되지 않았음 |
| 현재·최근 주기 노트 | 해당 노트가 없어 HTTP 40461 반환. 주기 노트 기능 성공으로 판단하지 않음 |
| TLS 검증 | 어댑터의 `verify_ssl=False` 기본값을 외부 네트워크 HTTPS 연결에 그대로 재사용하지 않음. 본 구성은 루프백 HTTP를 사용 |
| 장시간·동시 쓰기 | 장시간 무중단, OneDrive 등 파일 동기화 충돌, 동시 쓰기 결과는 검증되지 않음 |

이 작업에서는 미검증 기능을 위해 추가 커뮤니티 플러그인을 설치하거나 Vault 데이터를 임의 생성하지 않았다.

---

## 9. 최종 확인 상태

다음 사항을 실제 실행·검증으로 확인했다.

1. Local REST API가 루프백에서만 수신하고 MCP 인스턴스의 Bearer 인증을 처리한다.
2. 독립된 Python MCP 서버가 `stdio`로 FastMCP Gateway에 연결된다.
3. Gateway·OAuth·Tunnel 계층을 통해 원격 MCP 클라이언트가 실제 Vault 데이터를 조회하고 시험 노트를 수정·삭제할 수 있다.
4. 15개 MCP Tool이 노출되며, 핵심 파일 조작·검색 기능은 실호출로 검증됐다.
5. 특수 파일명 URL 인코딩 결함을 수정했고, **116개 테스트 및 실제 원격 CRUD 회귀 검증**을 통과했다.
6. 비밀정보·절대 경로·특정 Vault 식별자 없이 동일한 네트워크 경계와 요청 흐름을 재구성할 수 있다.

핵심 운영 원칙은 **「검색·대상 확인 → 노트 읽기 → 범위를 제한한 patch → 재조회로 검증」**이다. 새로운 노트를 만들거나 삭제할 때에는 정확한 상대 경로와 영향 범위를 확인하고, 별도 시험 파일을 사용해 먼저 검증한다.
