# FastMCP OAuth 공개 호스트 이전 및 Secure MCP Tunnel 연결 복구

## 1. 목적과 기록 범위

Tailscale 장치명 변경 후 **OAuth 공개 호스트, Tailscale Funnel, Auth0 Callback, Gateway OAuth metadata, Tunnel trusted origin**의 주소가 달라진 문제를 복구했다. 기존 FastMCP Gateway의 사용자 인증·권한 경계는 유지하면서 외부 OAuth와 비공개 MCP를 재연결했다.

이 문서는 [FastMCP OAuth·Secure MCP Tunnel 연결 및 도구 권한 검증](fastmcp-oauth-secure-tunnel-authorization.md)의 **운영 복구 후속 사례**다. 초기 OAuth 구현이나 다른 Gateway의 scope 정책을 재설계한 기록이 아니다.

사용 기술: Windows PowerShell, FastMCP **4.0.11**, tunnel-client **0.0.15**, Auth0, Tailscale Funnel, OpenAI Secure MCP Tunnel, stdio 기반 Obsidian MCP upstream. 버전은 작업 당시의 확인값이다. 호스트명과 내부 경로는 환경에 종속되지 않도록 역할로 치환했다.

**최종 상태:** 현재 공개 OAuth 호스트를 통한 metadata 조회, 인증된 ChatGPT MCP 읽기, 격리된 테스트 노트 생성·읽기·삭제, Tunnel 프로파일 수정과 재시작 후 재호출이 성공했다. 개별 OAuth 4단계 HTTP 응답 및 refresh-token rotation은 이번 작업의 직접 검증에 포함되지 않는다.

## 2. 문제 발생과 초기 진단

장치의 Tailscale DNS 이름은 변경됐지만 기존 OAuth 구성은 이전 이름을 계속 참조했다.

| 항목 | 초기 관측 상태 |
|---|---|
| 이전 Tailscale 호스트 | DNS NXDOMAIN, 외부 OAuth 접근 불가 |
| 현재 Tailscale 호스트 | DNS 조회 가능, Funnel 미정합 상태에서 TLS 연결 실패 |
| Tailscale Funnel | 이전 호스트의 HTTPS 443을 로컬 OAuth Edge로 프록시 |
| Gateway / OAuth Edge | 로컬 응답은 정상이나 이전 공개 issuer 광고 |
| Auth0 Callback | 신규 주소를 HTTP 403 / Callback URL mismatch로 거부 |
| Tunnel 상태 | 로컬 health는 정상이었으나 OAuth 외부 주소 불일치를 해결하지 못함 |

로컬 Gateway의 미인증 MCP 요청은 HTTP 401, OAuth Edge의 MCP 경로는 HTTP 404였다. 즉 **로컬 인증 경계는 동작하지만 공개 OAuth 경로가 실패하는 상태**였다. 원인을 단일 프로세스 장애로 취급하지 않고 DNS, HTTPS Funnel, issuer, Auth0 허용 URL을 각각 검증했다.

## 3. 최종 연결 구조 및 보안 경계

~~~mermaid
flowchart LR
    Client["ChatGPT / OAuth client"]
    Browser["사용자 브라우저"]
    Tunnel["Secure MCP Tunnel"]
    Agent["로컬 tunnel-client"]
    Gateway["FastMCP Gateway: loopback /mcp"]
    Edge["OAuth Edge: loopback"]
    Funnel["Tailscale Funnel: 공개 HTTPS 443"]
    Auth0["Auth0"]
    Vault["Obsidian MCP upstream"]

    Client <-->|"인증된 MCP"| Tunnel
    Tunnel <--> Agent
    Agent --> Gateway
    Gateway <--> Vault
    Client -->|"OAuth 요청·토큰"| Funnel
    Browser <-->|"로그인·callback"| Funnel
    Funnel --> Edge
    Edge -->|"허용된 OAuth 경로만"| Gateway
    Gateway <-->|"upstream OIDC"| Auth0
~~~

| 구성 요소 | 최종 조건 |
|---|---|
| 공개 OAuth base | HTTPS 현재 호스트 + Gateway 전용 OAuth path prefix |
| 공개 issuer | HTTPS 현재 호스트의 루트 origin |
| Auth0 Allowed Callback | 현재 공개 OAuth base 아래의 /auth/callback |
| Tailscale Funnel | 현재 호스트 HTTPS 443 → loopback OAuth Edge |
| OAuth Edge | metadata·authorize·consent·callback·token 등 allowlist만 전달 |
| Gateway MCP | loopback /mcp; 미인증 요청은 HTTP 401 |
| 공개 MCP 경로 | /mcp 및 OAuth prefix 아래 /mcp는 HTTP 404 |
| Tunnel trusted origin | HTTPS origin만 허용; path·query를 포함하지 않음 |
| Tunnel resource alias | 승인된 client-facing resource만 local canonical resource로 변환 |

Tunnel runtime API key와 사용자의 OAuth 자격은 별개다. Auth0 Client Secret은 로컬 보호 저장소에서 로드하고, 기존 **정확한 subject allowlist**와 resource 검증을 유지했다. 공개 연결 복구를 이유로 외부에 /mcp를 열거나 사용자 허용 조건을 완화하지 않았다.

### OAuth base와 trusted origin의 차이

~~~text
Public OAuth base
  https://<current-tailnet-host>/<oauth-prefix>

Auth0 Callback URL
  https://<current-tailnet-host>/<oauth-prefix>/auth/callback

Authorization Server Issuer
  https://<current-tailnet-host>/

Tunnel oauth_trusted_origins
  https://<current-tailnet-host>

Canonical MCP resource
  http://127.0.0.1:<gateway-port>/mcp
~~~

위 호스트, path prefix, 포트는 역할 예시다. **Tunnel trust 목록에는 OAuth base의 경로를 넣지 않는다.**

## 4. 실제 복구 순서

### 4.1 사전 확인과 백업

Git 기존 변경사항·미추적 런처를 확인하고 사용자 변경을 보존했다. 인증 소스·실행 스크립트, Funnel 상태 JSON 및 Tunnel 프로파일을 별도 백업해 복원 가능성을 확보했다. 관련 코드 변경 전부터 단위 테스트 21개와 로컬 접근 차단을 확인했다. 토큰·secret·실제 subject 원문을 문서나 Git에 기록하지 않았다.

### 4.2 현재 호스트로 공개 OAuth 이전

Gateway의 public OAuth base와 issuer 검증 조건, Tunnel launcher의 예상 Authorization Server, 관련 테스트와 문서를 현재 Tailscale 호스트로 통일했다. Gateway·OAuth Edge를 해당 변경에 맞게 재시작하고 메타데이터를 비교했다. 기존 로컬 MCP target과 Tunnel ID 및 Edge route allowlist는 유지했다.

현재 호스트의 HTTPS 443 → OAuth Edge Funnel 매핑을 활성화했다. 공개 OAuth metadata HTTP 200과 외부 MCP 경로 HTTP 404를 검증했다.

### 4.3 Auth0 Callback 전환

Auth0 Application에 현재 공개 호스트의 Callback을 등록했다. 새 callback은 처음 HTTP 403을 반환했지만 등록 후 **HTTP 302**를 반환했다. 이 302는 주소 허용 검증이지 실제 로그인·토큰 교환 전체의 직접 증거는 아니다.

ChatGPT 재연결 및 MCP 읽기 성공 이후 이전 Callback을 제거했다. 추가 조회에서 **현재 Callback 302 / 이전 Callback 403**을 확인했다. 다른 클라이언트의 전체 과거 사용 이력을 감사한 결과로 확대하지 않는다.

### 4.4 오래된 Funnel 매핑만 선택적으로 삭제

새 매핑을 추가했는데도 이전 호스트가 Serve/Funnel 설정의 Web 및 AllowFunnel에 남았다. 일반 Funnel CLI로 호스트별 삭제를 확인할 수 없었으므로 전체 reset은 사용하지 않았다.

로컬 Tailscale LocalAPI로 설정을 읽어 백업하고, 다음 조건으로 이전 항목만 제거했다.

1. 실제 설정이 두 호스트·동일 백엔드라는 사전 조건과 일치하는지 비교
2. 현재 호스트의 Web, AllowFunnel 및 TCP HTTPS 443 설정 보존
3. 이전 호스트의 Web·AllowFunnel 두 키만 삭제
4. 읽은 ETag와 If-Match 헤더로 동시 변경 시 쓰기 거부
5. 적용 뒤 LocalAPI와 Funnel 상태 JSON으로 단일 현재 호스트만 남았는지 확인

LocalAPI의 변경 요청은 HTTP 200으로 성공했고, 새 HTTPS 프록시는 계속 동작했다. 이는 **당시 Tailscale 버전에서 실제 검증한 선택적 정리 방법**이다. 모든 버전의 API 동작을 보장하지 않는다.

### 4.5 Tunnel trusted origin 이전과 재시작

Gateway·Funnel·Auth0 이전 후에도 Tunnel 프로파일의 **mcp.oauth_trusted_origins**에 이전 호스트가 남아 있었다. 프로파일을 백업하고 해당 문자열 한 곳만 현재 HTTPS origin으로 바꿨다. 변경 전후 내용을 비교해 다른 필드가 그대로임을 확인했다.

~~~yaml
mcp:
  oauth_trusted_origins:
    - "https://<current-tailnet-host>"
  server_urls:
    - channel: main
      url: "http://127.0.0.1:<gateway-port>/mcp"
~~~

변경 후 Tunnel Doctor가 통과했다. 실행 파일·프로필·PID로 해당 Tunnel을 식별하고, 그 프로세스만 종료한 뒤 기존 런처로 단일 인스턴스를 재기동했다. 새로운 Tunnel에서 **live=true / ready=true**, 통합 런처에서 READY를 확인했다.

이전 trusted origin 참조가 기존 실사용 장애의 직접 원인이었다고 단정하지 않는다. 실제 연결은 수정 전에도 일부 성공했으며, 이번 수정의 검증 근거는 **프로파일 정합성·Doctor·재시작 후 호출 성공**이다.

### 4.6 ChatGPT 재인증 및 Obsidian CRUD 확인

ChatGPT 연결을 재인증·재생성한 뒤 실제 MCP에서 Obsidian Vault의 루트 디렉터리 목록을 조회했다. 격리된 일회성 테스트 노트 하나를 **생성 → 내용 확인 → 삭제 → 목록 부재 확인**하여 실제 쓰기·읽기·삭제 경로를 검증했다.

운영 노트에 대한 임의 변경이나 파괴적인 대량 시험은 수행하지 않았다.

## 5. 검증 방법과 실제 결과

| 검증 대상 | 확인 방법 | 결과 |
|---|---|---|
| 공개 OAuth 주소 | HTTPS Authorization Server metadata | HTTP **200** |
| 비공개 MCP 경계 | Funnel 호스트의 /mcp, OAuth prefix /mcp | 각각 HTTP **404** |
| 로컬 Gateway 인증 | 인증정보 없는 /mcp | HTTP **401** |
| OAuth Edge 경계 | loopback Edge의 /mcp | HTTP **404** |
| Auth0 Callback | 현재·이전 redirect URI를 무로그인 요청으로 비교 | 현재 **302**, 이전 **403** |
| Funnel 정리 | Web·AllowFunnel 설정 재조회 | 현재 호스트만 존재 |
| Tunnel 설정 | trusted origin 비교, Doctor | 새 origin, **PASS** |
| Tunnel 재기동 | 프로세스 재식별, health, 통합 런처 | 단일 프로세스, live/ready, READY |
| 자동 회귀 | uv run --frozen pytest -q | **21 passed** |
| 변경 형식 | git diff --check | **PASS** |
| 실제 ChatGPT 읽기 | Vault 루트 목록 | **성공** |
| 격리된 파일 작업 | 임시 노트 생성·읽기·삭제·목록 재조회 | **성공** |

상태 코드 및 통과 개수는 해당 환경의 실제 관측값이다.

## 6. 주요 이슈와 해결 과정

### 6.1 장치명 변경과 OAuth 주소 불일치

**발생:** 장치명 변경 후 기존 공개 OAuth 호스트의 DNS가 NXDOMAIN을 반환했다.

**원인 추적:** 실제 장치 DNS 이름과 Gateway issuer·Funnel 호스트가 일치하지 않았다.

**해결:** 공개 OAuth 설정, Funnel, Auth0 Callback 및 Tunnel trust 설정을 각각 현재 호스트와 일치시켰다.

**검증·결과:** 새로운 metadata 200, 공개 MCP 차단 404, Gateway·Edge issuer 일치와 실제 ChatGPT 도구 호출 성공.

### 6.2 Auth0 Callback URL mismatch

**발생:** 새 hostname을 사용한 OAuth 요청에 대해 Auth0가 HTTP 403을 반환했다.

**원인 추적:** 당시 허용된 Callback 목록에 새 URI가 없었다.

**해결:** 새 Callback 등록, 연결 확인 후 구 Callback 제거.

**검증·결과:** 현재 주소 302, 이전 주소 403 및 인증된 MCP 호출 성공.

### 6.3 이전 Funnel hostname 매핑 잔존

**발생:** 새 Funnel 매핑을 만든 뒤 기존 hostname의 Web·AllowFunnel도 남아 있었다.

**원인 추적:** 실제 Serve/Funnel 설정에서 두 hostname이 동시에 확인됐다.

**해결:** 전체 reset 대신 백업·ETag로 보호된 LocalAPI 갱신으로 구 hostname 두 키만 제거.

**검증·결과:** 현재 HTTPS 443 경로 유지, 이전 항목 부재, 공개 OAuth 및 ChatGPT 호출 성공.

### 6.4 Tunnel 프로파일의 이전 origin

**발생:** 런처·Gateway OAuth 주소를 변경한 뒤에도 Tunnel 프로파일의 trusted origin만 구 hostname이었다.

**원인 추적:** Tunnel 신뢰 origin은 Gateway 공개 base와 별도로 관리되는 설정이었다.

**해결:** 백업·단일 문자열 수정 후 식별된 Work Tunnel만 재기동.

**검증·결과:** Doctor, health 및 재시작 후 ChatGPT 읽기 성공.

## 7. 인증 단계별 관측 한계와 종료 상태

OAuth 연결 성공 후에도 아래 경로의 **개별 요청과 HTTP 상태가 하나의 신규 인증 세션에서 모두 기록됐다는 증거**는 확보하지 못했다.

~~~text
/authorize
/consent
/auth/callback
/token
~~~

경로·시각·HTTP 상태만 수집하는 임시 OAuth Edge 관측기를 실제 기동했으나, 인증 시점과 수집 창이 맞지 않아 대상 이벤트 기록은 **0건**이었다. 이 결과를 OAuth 실패로 해석하지 않았다. 임시 관측기·로그·일회성 스크립트를 제거하고 원래 Edge를 복구했다. 복구 후 통합 검사와 실제 MCP 읽기 호출을 다시 통과했다. Refresh token의 실제 만료 및 회전도 이번 작업에서는 별도 시험하지 않았다.

**최종 종료 상태:** 공개 OAuth hostname, Gateway·Edge issuer, Auth0 Callback, Funnel의 단일 mapping, Tunnel trusted origin이 서로 일치하며, 공개 MCP 비노출 경계와 기존 사용자 allowlist가 유지된다. 실제 ChatGPT MCP 읽기 및 격리된 생성·삭제가 성공했다. **인증 흐름의 단계별 HTTP 로그 미확보는 운영 기능 검증과 구분해 기록한다.**
