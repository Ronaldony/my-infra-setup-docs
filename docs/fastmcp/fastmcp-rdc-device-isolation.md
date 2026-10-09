# FastMCP RDC Device Pinning 및 Discovery Isolation 구성

## 1. 목적과 기록 범위

FastMCP Gateway를 통해 hosted Remote Desktop Commander(RDC)를 사용할 때, 동일한 RDC 계정에 여러 Remote Device가 등록되어 있어도 **Gateway 하나가 지정된 단일 Device만 대표하도록 실행·조회 경계를 추가했다.**

이 작업은 기존 provider-aware Tool Routing 위에 RDC 전용 device isolation을 추가한 후속 구성이다.

관련 기존 문서:

- [FastMCP MCP Tool Routing 및 단계적 스키마 조회 구성](fastmcp-mcp-tool-routing.md)
- [FastMCP Tool Routing 후속 Reliability 검증 및 회귀 기준선](fastmcp-routing-oauth-validation-baseline.md)

기존 OAuth/OIDC, Secure MCP Tunnel, ProxyProvider, app/provider hard routing 구조는 재설계하지 않았다.

이번 작업의 목표는 다음 두 가지였다.

- **Device Pinning**: RDC의 device-scoped tool 호출을 하나의 설정된 Device ID로 강제한다.
- **Device Discovery Isolation**: RDC 계정 전체 device 목록을 노출하지 않고 Gateway에 할당된 Device만 조회되도록 한다.

적용 범위는 `desktop_commander` provider와 해당 `rdc_*` 도구에 한정했다. 다른 provider에는 동일 정책을 자동 적용하지 않는다.

---

## 2. 기존 구조에서 확인한 문제

Gateway 프로세스가 특정 Remote Device와 같은 호스트에서 실행된다는 사실만으로 RDC 호출 대상이 그 호스트에 자동 고정되지는 않았다.

기존 요청 경로는 다음과 같았다.

~~~mermaid
flowchart TD
    C["MCP Client"]
    G["FastMCP Gateway"]
    R["Hosted Remote Desktop Commander MCP"]
    A["RDC Account Device Set"]
    T["Target Device"]
    O["Other Registered Device"]

    C --> G
    G --> R
    R --> A
    A --> T
    A --> O
~~~

hosted RDC upstream의 OAuth 연결은 Gateway가 실행되는 로컬 PC가 아니라 **RDC 계정의 접근 범위**를 기준으로 동작했다.

실제 확인 결과:

- RDC device 조회는 같은 계정에 등록된 복수 Device를 반환했다.
- 각 Device ID를 명시하면 서로 다른 Remote Device에 대한 호출이 가능했다.
- 따라서 Gateway host와 제어 대상 Remote Device 사이에는 자동으로 1:1 경계가 형성되지 않았다.

즉 기존 provider routing은 다음 경계까지는 보장했다.

~~~text
app/provider 선택
    ↓
desktop_commander provider
~~~

하지만 provider 내부에서는 다음 선택이 여전히 가능했다.

~~~text
desktop_commander
    ├─ Target Device
    └─ Other Device
~~~

provider isolation과 device isolation은 별도의 문제였다.

---

## 3. 최종 구조

최종 구조에서는 provider routing 이후에 RDC 전용 device policy를 추가했다.

~~~mermaid
flowchart TD
    C["MCP Client"]
    G["FastMCP Gateway"]
    P["desktop_commander provider"]
    D["RDC Device Isolation Policy"]
    H["Hosted RDC MCP"]
    T["Configured Target Device"]

    C --> G
    G --> P
    P --> D
    D -->|"deviceId 강제 / 검증"| H
    H --> T
~~~

Gateway는 다음 설정을 RDC의 단일 device 기준으로 사용한다.

~~~env
REMOTE_FASTMCP_DEVICE_ID=<target-device-id>
~~~

이 값은 OAuth secret이 아니라 routing 정책에 사용되는 Remote Device 식별자다.

핵심 원칙은 다음과 같다.

~~~text
provider 선택
    ↓
RDC tool schema 확인
    ↓
deviceId 대상 tool이면 configured ID 강제
    ↓
다른 ID 요청이면 Gateway에서 거부
    ↓
upstream 호출
~~~

---

## 4. Device Pinning

### 4.1 Schema 기반 적용

RDC 도구 이름을 별도 목록으로 하드코딩하지 않고, 선택한 tool의 input schema에 `deviceId` property가 존재하는지 확인한다.

~~~text
tool input schema
    ↓
deviceId 존재?
    ├─ Yes → pinning 정책 적용
    └─ No  → 일반 provider routed call 유지
~~~

이 방식은 device-scoped RDC 도구와 account/global 성격의 도구를 구분하면서 기존 Tool Routing 구조를 유지하기 위해 사용했다.

### 4.2 deviceId 생략

클라이언트가 `deviceId`를 생략하면 Gateway가 설정된 ID를 자동 삽입한다.

~~~text
Client arguments
{}

        ↓ Gateway

Upstream arguments
{
  "deviceId": "<configured-target-id>"
}
~~~

따라서 클라이언트는 매 호출마다 Remote Device ID를 직접 선택할 필요가 없다.

### 4.3 동일한 deviceId 지정

클라이언트가 설정된 ID와 동일한 값을 명시하면 정상 호출한다.

~~~text
requested deviceId == configured deviceId
→ allow
~~~

### 4.4 다른 deviceId 지정

클라이언트가 다른 Remote Device ID를 지정하면 upstream으로 전달하기 전에 Gateway가 거부한다.

~~~text
requested deviceId != configured deviceId
→ reject at Gateway
→ no upstream device call
~~~

이 검사는 단순한 기본값 지정이 아니라 **다른 device로의 우회 호출을 막는 실행 경계**다.

---

## 5. Fail-Closed 동작

device isolation 설정이 빠졌을 때 account 전체 접근으로 되돌아가는 fallback은 허용하지 않았다.

`REMOTE_FASTMCP_DEVICE_ID`가 비어 있으면 다음 동작은 실패한다.

- input schema에 `deviceId`가 있는 device-scoped RDC tool
- Gateway가 제공하는 isolated device discovery

반면 `deviceId`를 사용하지 않는 provider-global tool은 기존 routed call 동작을 유지한다.

~~~mermaid
flowchart TD
    Call["RDC routed call"]
    S{"tool schema에 deviceId?"}
    P{"configured pin 존재?"}
    A["Configured Device로 실행"]
    F["Fail Closed"]
    G["기존 global tool 실행"]

    Call --> S
    S -->|Yes| P
    P -->|Yes| A
    P -->|No| F
    S -->|No| G
~~~

따라서 설정 누락이 **무제한 device 선택 권한으로 확대되는 방향**으로 실패하지 않는다.

---

## 6. Device Discovery Isolation

### 6.1 account-wide list_devices 문제

upstream의 `rdc_list_devices`는 입력 파라미터가 없고, 연결된 RDC 계정의 Device 목록을 반환하는 account-wide discovery tool이다.

이를 그대로 proxy하면 실행 경로를 pinning하더라도 다른 Remote Device의 존재와 ID가 계속 노출된다.

따라서 실행 경계와 조회 경계를 함께 맞춰야 했다.

### 6.2 최종 처리 방식

Gateway는 routed `rdc_list_devices` 요청을 일반 upstream call로 전달하지 않는다.

대신 configured Device ID를 기준으로 해당 Device에 `rdc_ping`을 수행하고, Gateway에 할당된 Device 정보만 합성해 반환한다.

~~~mermaid
sequenceDiagram
    participant C as MCP Client
    participant G as FastMCP Gateway
    participant R as Hosted RDC MCP
    participant T as Configured Device

    C->>G: call_tool(... rdc_list_devices ...)
    G->>G: configured deviceId 확인
    G->>R: rdc_ping(configured deviceId)
    R->>T: ping
    T-->>R: pong
    R-->>G: ping result
    G-->>C: configured Device ID + status only
~~~

중요한 점은 **account-wide `rdc_list_devices` 결과를 받아 문자열 후처리하는 방식이 아니라, upstream 목록 조회 자체를 호출하지 않는 것**이다.

최종 discovery 결과는 개념적으로 다음 범위만 포함한다.

~~~text
Desktop Commander device assigned to this gateway

Device ID: <configured-target-id>
Status: Online
~~~

다른 등록 Device의 이름이나 ID는 Gateway 응답에 포함되지 않는다.

---

## 7. Tool Discovery Metadata 정합성

실행 결과만 격리하고 Tool Search나 schema 설명에 account-wide 동작이 남아 있으면 클라이언트가 실제 Gateway 계약을 잘못 이해할 수 있다.

따라서 `rdc_list_devices`가 Gateway를 통해 노출될 때 description도 isolated 동작에 맞게 변경했다.

기존 의미:

~~~text
RDC 계정에 등록된 Device 목록 조회
~~~

Gateway에서의 최종 의미:

~~~text
이 Gateway에 할당된 Desktop Commander Device 조회
~~~

이 변경은 upstream tool 자체를 수정하는 것이 아니라, Gateway의 search/schema 표현에서 실제 routed behavior와 설명을 일치시키기 위한 것이다.

---

## 8. Provider Routing과의 결합

기존 public surface는 그대로 유지했다.

~~~text
search_tools
get_tool_schema
call_tool
~~~

Device policy는 이 routed call 경로 내부에 추가했다.

~~~mermaid
flowchart TD
    Search["search_tools(app, query)"]
    Schema["get_tool_schema(app, names)"]
    Call["call_tool(app, name, arguments)"]
    Owner["provider ownership 검증"]
    Device["RDC device policy"]
    Proxy["ProxyProvider"]
    Upstream["Hosted RDC MCP"]

    Search --> Schema
    Schema --> Call
    Call --> Owner
    Owner --> Device
    Device --> Proxy
    Proxy --> Upstream
~~~

따라서 기존 provider boundary는 유지된다.

- app ID 검증
- namespace ownership 검증
- 직접 namespaced upstream tool 호출 차단
- OAuth authorization 적용
- routed call 강제

그 이후 RDC provider에 한해서 device boundary가 추가된다.

~~~text
OAuth / authorization
        ↓
provider routing
        ↓
RDC device isolation
        ↓
ProxyProvider
        ↓
hosted RDC MCP
~~~

OAuth가 device pinning을 대신하지 않고, device pinning도 OAuth 권한을 대신하지 않는다.

---

## 9. 다른 Provider에 대한 영향 차단

Device Pinning은 Gateway 전체 공통 정책으로 적용하지 않았다.

현재 정책 범위는 다음과 같다.

~~~text
FastMCP Gateway
    ├─ desktop_commander → Device Pinning + Discovery Isolation
    └─ other provider    → 기존 routing 동작 유지
~~~

즉 `deviceId`라는 이름의 파라미터가 다른 provider에 존재한다는 이유만으로 자동 pinning하지 않는다.

device pin 설정은 app/provider ID에 매핑하고, RDC의 `list_devices` 특수 처리는 `desktop_commander` provider에 한정했다.

이 범위 제한은 provider별 의미가 다른 파라미터나 discovery semantics에 RDC 정책이 잘못 전파되는 것을 막는다.

---

## 10. 설정과 운영 흐름

### 10.1 설정

Gateway 실행 환경에 대상 Remote Device ID를 지정한다.

~~~env
REMOTE_FASTMCP_DEVICE_ID=<target-device-id>
~~~

실행 launcher가 Gateway용 환경변수를 로드하는 구조에서는 이 값도 동일한 설정 경로를 통해 주입한다.

### 10.2 실행

Gateway 시작 시 해당 설정이 routing transform에 전달된다.

~~~text
environment
    ↓
Gateway settings
    ↓
provider routing transform
    ↓
desktop_commander device pin
~~~

### 10.3 재시작

정책 변경 후에는 실행 중인 Gateway와 외부 tunnel 경로를 재시작해 새 설정과 코드를 실제 MCP 연결에 반영했다.

재기동 후 Gateway와 tunnel listener가 정상 상태임을 확인한 뒤, 최종 E2E 검증을 수행했다.

---

## 11. 검증

### 11.1 변경 전 기준선

작업 전 기존 전체 테스트를 실행해 **30개 테스트가 모두 통과**하는 상태를 기준선으로 확보했다.

이는 이후 발생하는 실패가 기존 문제인지 device isolation 변경으로 인한 회귀인지 구분하기 위한 기준이었다.

### 11.2 자동 테스트

Device isolation 동작을 검증하는 테스트를 추가했다.

검증 항목:

- `deviceId` 생략 시 configured ID 자동 주입
- configured ID를 직접 지정하면 정상 실행
- 다른 ID 지정 시 오류
- 다른 ID 호출이 upstream에 도달하지 않음
- `deviceId`가 없는 global tool은 기존 동작 유지
- isolated `rdc_list_devices` 응답에 configured ID만 포함
- account-wide upstream list tool이 호출되지 않음
- discovery 상태 확인은 configured ID에 대한 ping으로 수행
- `rdc_list_devices` schema 설명이 isolated semantics와 일치
- pin 미설정 시 device operation과 discovery가 fail closed
- pin 미설정 상태에서도 device와 무관한 global tool은 유지

변경 후 전체 테스트는 **33개 모두 통과**했다.

### 11.3 실제 hosted RDC smoke

실제 hosted RDC upstream을 사용해 routed path를 검증했다.

확인 결과:

- public surface는 기존 `search_tools`, `get_tool_schema`, `call_tool` 세 도구로 유지
- tool search에서 RDC configuration 도구 검색 성공
- selected schema 조회 성공
- isolated device discovery 호출 성공
- `deviceId`를 생략한 configuration 조회 성공

### 11.4 실제 E2E Device Isolation

실제 MCP Client → 인증된 Gateway → hosted RDC 경로에서 다음을 확인했다.

~~~text
isolated device discovery
→ configured Device ID 존재
→ 다른 등록 Device ID 없음

deviceId 생략 호출
→ configured Target Device에서 실행

다른 Device ID 강제 호출
→ Gateway에서 거부

list_devices schema
→ isolated Gateway semantics로 표시
~~~

최종 재기동 후 실제 연결된 Gateway에서도 동일 결과를 다시 확인했다.

---

## 12. 주요 이슈와 해결 과정

### 12.1 이슈 내용과 발생 시점

Gateway가 특정 Remote Device에서 실행되고 있었지만 `rdc_list_devices`에서 다른 Remote Device까지 함께 조회됐다.

추가 확인에서 다른 Device ID를 명시한 실제 `rdc_ping` 호출도 성공했다.

### 12.2 발생 원인 추적

Gateway host와 RDC target은 같은 개념이 아니었다.

~~~text
Gateway host
    ↓
hosted RDC OAuth / account
    ↓
account에 등록된 Remote Device들
~~~

upstream OAuth가 account-scoped이므로, Gateway가 어느 PC에서 실행되는지만으로 Remote Device가 제한되지 않았다.

기존 Tool Routing 코드도 provider ownership까지만 검사했고 tool arguments의 `deviceId`를 제한하지 않았다.

테스트에 사용하던 별도 Device ID 환경변수는 smoke validation 용도였으며 production routing policy가 아니었다.

### 12.3 해결 과정

다음 순서로 보완했다.

1. production 설정으로 `REMOTE_FASTMCP_DEVICE_ID` 추가
2. tool schema 기반 `deviceId` 자동 주입
3. 다른 `deviceId` 요청의 upstream 이전 차단
4. pin 누락 시 device operation fail closed
5. account-wide `rdc_list_devices` proxy 차단
6. configured ID에 대한 ping 기반 isolated discovery 구성
7. Tool Search/schema description 정합성 보완
8. 자동 테스트와 실제 hosted RDC E2E 검증
9. 실행 Gateway/Tunnel 재기동 후 실제 MCP 연결 재검증

### 12.4 검증

자동 테스트, smoke, E2E를 모두 사용했다.

특히 다른 Device ID 요청이 단순히 최종 실행에서 실패하는 것이 아니라 **Gateway 정책 단계에서 거부되는지** 확인했다.

또한 device discovery에서 다른 등록 Device ID가 결과에 포함되지 않는지 확인했다.

### 12.5 결과

최종 Gateway는 RDC 계정 자체를 단일 Device 계정으로 변경하지 않고도, Gateway 경로에서는 하나의 configured Remote Device만 대표하게 됐다.

~~~text
RDC Account
    ├─ Target Device
    ├─ Other Device A
    └─ Other Device B

Gateway A
    ↓
Target Device only
~~~

---

## 13. 보안 경계와 적용 범위

이 구성의 device isolation은 **FastMCP Gateway 경로의 정책**이다.

따라서 다음 사실을 구분해야 한다.

- RDC 계정에서 다른 Remote Device가 삭제되는 것은 아니다.
- 다른 RDC 클라이언트가 같은 계정 권한으로 직접 접근하는 경로까지 변경하지 않는다.
- 이 Gateway를 통과하는 RDC routed call과 discovery만 configured Device로 제한한다.

즉 보안 경계는 다음 위치에 있다.

~~~text
MCP Client
    ↓
FastMCP Gateway
    ↓  ← Device Isolation Boundary
Hosted RDC MCP
    ↓
Configured Remote Device
~~~

이 구조는 **하나의 Gateway 인스턴스를 하나의 RDC Remote Device에 대응시키는 목적**에 맞춰 적용됐다.

---

## 14. 최종 결과

작업 전에는 provider routing만 존재해 `desktop_commander` 내부의 여러 Remote Device를 선택할 수 있었다.

작업 후에는 다음 두 경계가 결합됐다.

~~~text
Provider Boundary
desktop_commander only

        +

Device Boundary
configured Remote Device only
~~~

최종 요청 흐름은 다음과 같다.

~~~mermaid
flowchart LR
    C["MCP Client"]
    O["OAuth / Authorization"]
    R["Provider Routing"]
    D["RDC Device Pinning"]
    P["ProxyProvider"]
    H["Hosted RDC MCP"]
    T["Configured Remote Device"]

    C --> O
    O --> R
    R --> D
    D --> P
    P --> H
    H --> T
~~~

결과적으로 RDC 계정에 여러 Remote Device가 등록되어 있어도, 해당 FastMCP Gateway를 사용하는 클라이언트는 **실행과 discovery 모두 설정된 단일 Remote Device로 제한**된다.
