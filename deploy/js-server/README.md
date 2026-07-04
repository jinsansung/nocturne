# `noc.whalesound.net`에 Nocturne 설치 (js-server 공유 엣지)

이 디렉터리는 남이 소유한 apex(`whalesound.net`)의 **서브도메인**에서, 공유
**js-server Caddy** 엣지 뒤로 Nocturne을 돌리는 **공유 엣지 · 단일 테넌트**
배포입니다. [`../docker-compose/`](../docker-compose/)의 번들 Caddy는 쓰지 않습니다.

## 왜 이게 필요한가 — 지난 서브도메인 설치 실패 원인

기본 번들(`../docker-compose/docker-compose.yaml`)은 **전용 apex**를 전제로 합니다.

- **자체 Caddy**가 호스트 포트 **80/443**을 점유하고,
- `*.{BASE_DOMAIN}`용 **와일드카드 on-demand TLS**(테넌트 서브도메인마다 인증서 1장)를 발급하며,
- `.env.example`에 *"Root domain only … subdomains are generated per tenant"* 라고 명시돼 있습니다.

js-server 드롭릿에서는 이 전제가 성립하지 않습니다. **공유 Caddy가 이미 80/443과
모든 공개 라우팅을 독점**(6규칙 계약)하므로 두 번째 Caddy를 띄울 수 없고, 서브도메인
호스트에는 테넌트별 와일드카드 모델을 얹을 공간이 없습니다. 이 충돌이 지난 서브도메인
설치 실패의 원인입니다.

**앱 코드 자체는 문제가 아니었습니다.** `BASE_DOMAIN`은 설정된 호스트를 기준으로
*상대적으로* 계산됩니다.

- `rpId = BASE_DOMAIN`(포트 제외) → WebAuthn/패스키가 `noc.whalesound.net`과 그
  서브도메인에 스코프됩니다 (`ServiceRegistrationExtensions.cs`).
- `SubdomainParser.Extract(host, BASE_DOMAIN)`이 base domain 기준으로 테넌트를
  해석합니다 — 멀티라벨 base domain(`nocturne.theconen.de`)으로 이미 유닛 테스트됨.
- 테넌트가 정확히 1개면 `TenantResolutionMiddleware`가 apex 호스트에서 자동 해석하므로,
  **서브도메인 없이** 단일 호스트에서 전체 앱이 동작합니다.

따라서 해결은 순전히 **엣지 계층**에 있습니다: 번들 Caddy를 빼고, YARP **게이트웨이**만
공유 `js-server_edge` 네트워크에 고정 컨테이너 이름으로 노출한 뒤, 공유 Caddy가
`noc.whalesound.net`을 그쪽으로 리버스 프록시하게 합니다. **단일 테넌트**로 돌리므로
`noc.whalesound.net`이 앱 전체가 되고 **와일드카드 DNS/TLS가 전혀 필요 없습니다.**

## 이 스택이 하는 것 / 안 하는 것

| | |
|---|---|
| 공개 앱 | `https://noc.whalesound.net` (단일 테넌트, apex 자동 해석) |
| 번들 Caddy | **제거** — 공유 js-server Caddy가 TLS 종료 |
| 호스트 포트 publish | **없음** — 게이트웨이는 `js-server_edge`로만 접근 |
| 엣지 업스트림 | `nocturne-gateway:5000` (고정 `container_name`) |
| 영속성 | external + 고정 이름 볼륨 `nocturne-noc-postgres-data` |
| 테넌트별 서브도메인 (`*.noc…`) | **꺼짐** (단일 테넌트) — 켜려면 와일드카드 route 필요 |
| 공개 공유 링크 (`{token}.share.noc…`) | **꺼짐** — 아래 "공개 공유 링크 활성화" 참조 |

## 관련 upstream 논의 (이번에 조사한 것)

서브도메인/외부 프록시 설치에 대한 upstream 논의를 조사했고, 이 스택 설계와 일치합니다.

- **[nightscout/nocturne#292 — Add Cosmos Cloud deployment support](https://github.com/nightscout/nocturne/issues/292)**
  (열림): 외부 리버스 프록시 뒤 설치를 정면으로 다루는, 우리 상황과 가장 가까운 논의.
  핵심 결론 — (1) **이중 프록시 금지**: 외부 route 하나 → `gateway:5000`, 내부 경로
  라우팅(`/api`, `/scalar`, `/auth/bot`, catch-all)은 YARP에 맡긴다. (2) **멀티테넌시는
  와일드카드 필수**: 단일 호스트 route는 apex 테넌트 하나만 서빙하고, 테넌트별 서브도메인은
  `*.example.com` 와일드카드 route + DNS-01 와일드카드 인증서가 필요하다. (3) 외부 플랫폼에
  자체 업데이터가 있으면 **watchtower 제거**.
- **[nightscout/nocturne#325 — bundled Caddy reverse proxy](https://github.com/nightscout/nocturne/pull/325)**
  (병합): 번들 Caddy가 apex는 HTTP-01, 테넌트 서브도메인은 on-demand HTTP-01
  (`/api/v4/platform/tls-authorize` 게이트)로 발급 — **와일드카드 인증서 없음**. byo-proxy
  override로 Caddy를 끄고 게이트웨이를 평문 HTTP로 노출하는 경로도 여기서 나옵니다.
- **[nightscout/nocturne#368 — resolve sole tenant for /api/v4/status on the apex](https://github.com/nightscout/nocturne/pull/368)**
  (병합): self-hoster가 **base domain에서 단일 테넌트**로 돌렸을 때 `/setup`으로 튕기던
  버그 수정. `BASE_DOMAIN` 미설정 시에도 같은 `/setup` 증상이 난다는 점을 문서화. → 단일
  테넌트 접근 방식을 검증하며, 이 수정은 우리 이미지에 이미 포함돼 있습니다.
- **[nightscout/nocturne#135](https://github.com/nightscout/nocturne/pull/135) ·
  [#318](https://github.com/nightscout/nocturne/pull/318)**: apex 단일 테넌트에서의 실시간
  (Socket.IO/bridge) 해석과 핸드셰이크 인증 — 프록시 뒤 WebSocket 동작의 근거.

정리: **외부 프록시 뒤 단일 호스트 = 단일 테넌트**가 upstream이 지지하는 형태이고,
**멀티테넌트/공유 링크만이 와일드카드**를 요구합니다. 이 스택은 정확히 그 지지되는
경로를 따릅니다.

## 배포 (드롭릿, `/opt/js-noc/`에서)

```bash
# 1. 공유 엣지 네트워크 조인 (js-server 소유; 우리는 조인만)
docker network create js-server_edge 2>/dev/null || true

# 2. 영속 데이터 볼륨 1회 생성 (compose down / 재생성에도 데이터 보존)
docker volume create nocturne-noc-postgres-data

# 3. 시크릿
cp .env.example .env
#   BASE_DOMAIN=noc.whalesound.net 설정, INSTANCE_KEY + POSTGRES_* 4개 채우기.
#   각 값 생성:  openssl rand -base64 32

# 4. 기동
docker compose up -d
```

그다음 아래 라우팅 요청서를 **js-server 서버운영 세션**에 전달하세요. 공유 Caddy에 route가
생기기 전까지 게이트웨이는 `js-server_edge` 내부에서만 접근됩니다(설계상 포트를 열지 않음).

`https://noc.whalesound.net` 첫 접속은 `503 setup_required`를 반환하고 UI가 `/setup`으로
이동합니다. 설정을 마쳐 단일 테넌트 + 소유자 패스키를 생성하세요.

> **`/setup`에서 안 벗어난다면** (upstream #368): `BASE_DOMAIN`이 실제 서빙 호스트
> (`noc.whalesound.net`)와 정확히 일치하는지, 그리고 컨테이너에 실제로 주입됐는지
> 확인하세요. `BASE_DOMAIN` 오설정/미설정은 설정을 마쳐도 `/setup` 루프를 유발합니다.
>
> **로그인 화면은 뜨는데 로그인만 계속 실패한다면** (지난 서브도메인 설치 실패의 실제
> 증상): 거의 항상 **`BASE_DOMAIN`을 apex로 잘못 넣어서** 생기는 SvelteKit CSRF 403입니다.
> `ORIGIN`이 `https://${BASE_DOMAIN}`로 만들어지므로, `BASE_DOMAIN=whalesound.net`(apex)로
> 두고 앱을 `noc.whalesound.net`에서 서빙하면 서버 오리진(`https://whalesound.net`)과
> 브라우저 오리진(`https://noc.whalesound.net`)이 어긋나 로그인 POST가 매번
> *"Cross-site POST form submissions are forbidden"*(403)로 거부됩니다. 로그인 GET은
> 멀쩡히 뜨지만 제출만 실패하는 게 특징입니다. 같은 오설정은 테넌트 해석도 깨뜨려(`noc`을
> 없는 테넌트 슬러그로 인식) API가 404를 냅니다.
> **해결: `BASE_DOMAIN`을 최상위 도메인이 아니라 "실제로 서빙되는 그 주소"(여기선
> `noc.whalesound.net`)로 설정하세요.** 이 `.env.example`에는 이미 그렇게 박혀 있습니다.
> 추가로, 공유 Caddy가 `X-Forwarded-Proto: https`와 원본 `Host`(또는 `X-Forwarded-Host`)를
> 전달하는지 확인하세요(누락 시 secure 쿠키가 붙지 않아 로그인 후 세션이 안 유지됩니다).

## 라우팅 요청서 — js-server 서버운영 세션에 전달

> "신규 스택 → 서버운영" 템플릿을 채운 것:
>
> 1. **원하는 서브도메인 슬러그:** `noc` → `noc.whalesound.net` *(noc은 드롭됨, 재사용 가능)*
> 2. **업스트림:** `nocturne-gateway:5000`
> 3. **업스트림 프로토콜:** `http` (평문; TLS는 공유 Caddy에서 종료)
> 4. **프록시 특이사항:**
>    - **WebSocket + SSE 필요** — SignalR 허브(알림, 실시간 위젯)는 WebSocket을,
>      실시간 스트림은 SSE를 씁니다. `flush_interval -1` + WebSocket 업그레이드 활성화.
>      게이트웨이의 web 클러스터에는 이미 5분 activity timeout이 설정돼 있습니다.
>    - 원본 **Host**와 **X-Forwarded-*** (Proto/Host)를 전달할 것 — 앱이 이를 신뢰합니다
>      (`ASPNETCORE_FORWARDEDHEADERS_ENABLED=true`, SvelteKit `ORIGIN=https://noc.whalesound.net`).
>    - 대용량 업로드 경로 없음; 기본 바디 크기 제한으로 충분.
> 5. **공개 여부:** 공개 (Caddy route).
> 6. **js-server_edge 조인 확인:** 예 — `nocturne-gateway`가 `js-server_edge`에 조인.
> 7. **메모리(mem_limit, 매니페스트 기록용):** postgres 512m · api 640m · web 512m ·
>    gateway 256m (합계 ≈ **1.9 GB** 상한).
> 8. **스택 성격:** 코드 스택(자체 레포: `jinsansung/nocturne`), 이미지는
>    `ghcr.io/nightscout/nocturne/*`.

공유 Caddy route 최소 예시:

```caddy
noc.whalesound.net {
    reverse_proxy nocturne-gateway:5000
}
```

(Caddy는 WebSocket을 자동 처리합니다. SSE가 버퍼링되면 `flush_interval -1` 추가.)

## 공개 공유 링크 활성화 (선택, 나중에)

공유 링크는 `{token}.share.noc.whalesound.net`에서 서빙됩니다 — 무한한 호스트 집합입니다.
켜려면 **공유 Caddy** 쪽에 다음이 필요합니다.

- `*.share.noc.whalesound.net` (멀티테넌트 서브도메인까지 원하면 `*.noc.whalesound.net`도)
  와일드카드 route → `nocturne-gateway:5000`,
- 실제 호스트에만 인증서를 발급하도록 Nocturne 인가로 게이트한 on-demand TLS:
  `on_demand_tls { ask http://nocturne-api:8080/api/v4/platform/tls-authorize }`
  (이때 `nocturne-noc-api` 컨테이너도 `js-server_edge`에 조인해야 함),
- Cloudflare DNS에 `*.share.noc` (및 `*.noc`) **회색구름(DNS-only)** A 레코드.

HTTP-01은 와일드카드를 발급할 수 없으므로, 단일 와일드카드 인증서가 아니라 Caddy의
**호스트별** on-demand 발급에 의존합니다(upstream #292의 결론과 동일). 이는 공유 Caddy
설정 변경이므로 서버운영 세션에 요청하세요. 이 스택 범위 밖입니다.

## 업데이트

이미지는 `:latest`로 고정돼 있습니다. 업데이트:

```bash
docker compose pull && docker compose up -d
```

(여기엔 Watchtower를 넣지 않았습니다 — 이미지 갱신은 js-server가 표준화하는 방식에
맡깁니다. upstream #292의 "이중 업데이터 금지" 권고와 동일.)
