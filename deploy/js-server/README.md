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
| 엣지 업스트림 | `nocturne-noc-gateway:5000` (고정 `container_name`) |
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

## 사전 요구사항 (배포 전)

- **DNS:** Cloudflare에 `noc.whalesound.net` A 레코드를 드롭릿 IP(152.42.227.146)로,
  **회색구름(DNS-only)** 으로 생성. 공유 Caddy가 Let's Encrypt로 인증서를 발급하고
  라우팅하려면 이 레코드가 먼저 있어야 합니다. (호스트 정보상 apex도 회색구름 규칙.)
- **js-server_edge 네트워크:** 공유 Caddy가 소유하며 **보통 이미 존재**합니다(우리는
  조인만). 아래 `docker network create`는 없을 때를 대비한 멱등 폴백일 뿐이며, 서버운영
  세션에 이미 있는지 확인하는 편이 안전합니다(속성을 새로 만들면 안 됨).

## 배포 (드롭릿, `/opt/js-noc/`에서)

```bash
# 1. 공유 엣지 네트워크는 js-server 소유 — 조인만 한다.
#    (아래는 "없으면 만든다" 폴백; 이미 있으면 no-op)
docker network create js-server_edge 2>/dev/null || true

# 2. 영속 데이터 볼륨 1회 생성 (compose down / 재생성에도 데이터 보존)
docker volume create nocturne-noc-postgres-data

# 3. 시크릿
cp .env.example .env
#   BASE_DOMAIN=noc.whalesound.net 설정, INSTANCE_KEY + POSTGRES_* 4개 채우기.
#   시크릿은 URI-안전 생성기로:  openssl rand -hex 32
#   (plain `-base64 32`의 '/'·'+'는 nocturne_web의 Postgres URI를 깨뜨림 — .env.example 참고)

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
> 증상): 근본 원인은 **`BASE_DOMAIN`을 apex로 잘못 넣은 것**입니다. `ORIGIN`이
> `https://${BASE_DOMAIN}`로 만들어지므로, `BASE_DOMAIN=whalesound.net`(apex)로 두고 앱을
> `noc.whalesound.net`에서 서빙하면 서버 오리진(`https://whalesound.net`)과 브라우저
> 오리진(`https://noc.whalesound.net`)이 어긋납니다. 로그인 제출은 SvelteKit **remote
> function(POST)** 이라 이 오리진 불일치로 거부되고(오리진 검사 실패; 정확한 에러 문자열은
> form 제출용 *"Cross-site POST form submissions are forbidden"* 과 다를 수 있음, upstream
> #100 참고), 로그인 GET은 멀쩡히 뜨지만 제출만 실패하는 게 특징입니다.
> 같은 apex 오설정은 **한 번에 세 가지**를 동시에 깨뜨립니다: (a) 위 오리진(CSRF) 거부,
> (b) WebAuthn origin 불일치로 패스키 세리머니 실패 가능, (c) `SubdomainParser`가 `noc`을
> 없는 테넌트 슬러그로 인식해 API 404. 그래서 증상이 404와 섞여 보일 수 있습니다.
> **해결(셋 다 한 번에): `BASE_DOMAIN`을 최상위 도메인이 아니라 "실제로 서빙되는 그
> 주소"(여기선 `noc.whalesound.net`)로 설정하세요.** 이 `.env.example`에는 이미 그렇게
> 박혀 있습니다. 추가로, 공유 Caddy가 `X-Forwarded-Proto: https`와 원본 `Host`(또는
> `X-Forwarded-Host`)를 전달하는지 확인하세요(누락 시 secure 쿠키가 붙지 않아 로그인 후
> 세션이 안 유지되는, 진단이 까다로운 별개 증상이 납니다).
>
> **단일 테넌트 불변조건(중요):** 이 설치는 apex 자동 해석에 의존하며, **활성 테넌트가
> 정확히 1개**일 때만 `noc.whalesound.net`에서 동작합니다. 운영 중 **두 번째 테넌트나 데모
> 테넌트를 만들면** apex가 어느 테넌트인지 결정하지 못해 즉시 404가 되고 접근이 끊깁니다
> (`compose`는 `DemoService__Enabled=false`로 데모를 막아둠). 여러 테넌트가 필요하면
> 단일 호스트 모델을 벗어나 아래 와일드카드 경로가 필요합니다. 또, 최초 설치 직후 테넌트를
> 만들면 테넌트 캐시(약 5분) 때문에 잠깐 503/404가 남을 수 있습니다 — 몇 분 뒤 정상화.

## 라우팅 요청서 — js-server 서버운영 세션에 전달

> "신규 스택 → 서버운영" 템플릿을 채운 것:
>
> 1. **원하는 서브도메인 슬러그:** `noc` → `noc.whalesound.net` *(noc은 드롭됨, 재사용 가능)*
> 2. **업스트림:** `nocturne-noc-gateway:5000`
> 3. **업스트림 프로토콜:** `http` (평문; TLS는 공유 Caddy에서 종료)
> 4. **프록시 특이사항:**
>    - **WebSocket 필요** — SignalR 허브(알림, 실시간 위젯)가 WebSocket을 씁니다.
>      WebSocket 업그레이드를 허용하고, SignalR이 SSE 폴백 전송으로 협상될 수 있으니
>      프록시가 스트림을 버퍼링하지 않도록 `flush_interval -1`도 켜 두세요(무해).
>      게이트웨이의 web 클러스터에는 이미 5분 activity timeout이 설정돼 있습니다.
>    - 원본 **Host**와 **X-Forwarded-*** (Proto/Host)를 전달할 것 — 앱이 이를 신뢰합니다
>      (`ASPNETCORE_FORWARDEDHEADERS_ENABLED=true`, SvelteKit `ORIGIN=https://noc.whalesound.net`).
>    - 대용량 업로드 경로 없음; 기본 바디 크기 제한으로 충분.
> 5. **공개 여부:** 공개 (Caddy route).
> 6. **js-server_edge 조인 확인:** 예 — `nocturne-noc-gateway`가 `js-server_edge`에 조인.
> 7. **메모리(mem_limit, 매니페스트 기록용):** postgres 512m · api 640m · web 512m ·
>    gateway 256m (합계 ≈ **1.9 GB** 상한).
> 8. **스택 성격:** 코드 스택(자체 레포: `jinsansung/nocturne`), 이미지는
>    `ghcr.io/nightscout/nocturne/*`.

공유 Caddy route 최소 예시:

```caddy
noc.whalesound.net {
    reverse_proxy nocturne-noc-gateway:5000
}
```

(Caddy는 WebSocket을 자동 처리합니다. SSE가 버퍼링되면 `flush_interval -1` 추가.)

## 공개 공유 링크 활성화 (선택, 미검증 — 그대로 따라 하지 말 것)

공유 링크는 `{token}.share.noc.whalesound.net`에서 서빙됩니다 — 무한한 호스트 집합입니다.
이 스택 범위 밖이며, **아래는 검증되지 않은 방향성 메모**입니다.

> ⚠️ **중요 정정:** `on_demand_tls { ask …/api/v4/platform/tls-authorize }` 방식은 공유
> 호스트에 **그대로 쓸 수 없습니다.** 그 인가 엔드포인트(`TlsAuthorizationController`)는
> **apex와 "활성 테넌트 서브도메인"만** 200으로 승인하고, `{token}.share.…` 형태는
> `SubdomainParser`가 `{token}.share`를 테넌트 슬러그로 넘겨 매칭 실패 → **404**를 냅니다.
> 즉 on-demand ask 게이트에 물리면 Caddy가 공유 호스트 인증서를 **영영 발급하지 못합니다.**

따라서 공유 링크를 켜려면 on-demand ask가 아니라 **와일드카드 인증서** 전략이 필요합니다.

- `*.share.noc.whalesound.net` (멀티테넌트 서브도메인까지 원하면 `*.noc.whalesound.net`도)
  와일드카드 route → `nocturne-noc-gateway:5000`,
- 그 와일드카드에 대한 **DNS-01 와일드카드 인증서**(Cloudflare API 토큰 필요) — HTTP-01은
  와일드카드를 발급할 수 없습니다,
- Cloudflare DNS에 `*.share.noc` (및 `*.noc`) **회색구름(DNS-only)** 레코드.

이는 공유 Caddy의 인증서 발급 방식과 DNS 자격증명을 건드리는 변경이므로, 실제로 원할 때
서버운영 세션과 **별도로 설계·검증**하세요. (참고: 스톡 번들의 `tls-authorize` on-demand
경로도 apex + 테넌트 서브도메인만 커버하며 공유 호스트를 커버하지 않습니다.)

## 업그레이드 / 판올림 시 주의사항

간단 업데이트:

```bash
docker compose pull && docker compose up -d
```

여기엔 Watchtower를 넣지 않았습니다(upstream #292의 "이중 업데이터 금지"와 동일 —
이미지 갱신은 js-server 표준 방식에 맡김).

### 드리프트 주의 — 이 파일은 upstream 번들의 "손으로 파생한 사본"

이 `docker-compose.yaml`은 upstream 생성 번들(`deploy/docker-compose/`, `scripts/
publish-release.cs`가 Aspire AppHost에서 생성)에서 **손으로 옮겨 온** 파일입니다. 서비스
정의·게이트웨이 라우트 테이블·환경변수·postgres init SQL을 그 번들과 문자 그대로 맞춰
두었지만 **자동 동기화되지 않습니다.** 판올림 시 위험은 이 드리프트이지, 이 스택에 가한
하드닝(프로젝트명 고정·컨테이너명·hex 비번)이 아닙니다.

- **DB 스키마 마이그레이션은 자동**입니다(API 기동 시 migrator 역할이 실행) → 판올림에
  별도 조치 불필요.
- **`:latest`는 "고정"이 아닙니다.** `pull`이 예고 없이 상위 버전을 당길 수 있으므로,
  공유 드롭릿에서는 **릴리스 태그로 고정**하고 의도적으로 올리길 권장:
  ```
  NOCTURNE_API_IMAGE=ghcr.io/nightscout/nocturne/nocturne-api:vX.Y.Z
  NOCTURNE_WEB_IMAGE=ghcr.io/nightscout/nocturne/nocturne-web:vX.Y.Z
  ```
- **`container_name`·`js-server_edge`는 서버운영 세션과의 계약**입니다. 판올림 중에 이
  이름/네트워크를 바꾸면 공유 Caddy 라우트(`nocturne-noc-gateway:5000`)도 함께 갱신해야
  합니다. 되도록 유지하세요.

### 판올림 전 체크리스트

대상 릴리스의 upstream 번들(`deploy/docker-compose/docker-compose.yaml` 또는 portainer
번들)과 이 파일을 대조해, 아래가 바뀌었으면 이 파일에 반영한 뒤 배포하세요.

1. **환경변수** (api·web 서비스): 새/변경/삭제된 키 — 누락 시 기동 실패·오동작.
2. **게이트웨이 라우트 테이블**(`REVERSEPROXY__ROUTES__*`): 새 경로 프리픽스가 api/web로
   라우팅돼야 하는데 빠지면 404·오라우팅.
3. **새 서비스**: 예컨대 실시간 bridge/worker/redis 등이 별도 컨테이너로 분리됐는지 —
   빠지면 해당 기능(실시간 등)이 조용히 깨짐. (현재 토폴로지는 upstream과 동일: postgres·
   api·web·gateway. 실시간 bridge는 web 이미지 안에서 돎.)
4. **postgres 이미지 메이저 버전** 상승: 데이터 볼륨 `pg_upgrade` 필요 가능(현재 17.6 고정).
5. **DB init SQL / 역할**(`nocturne_migrator`/`app`/`web`) 변경.

빠른 대조:
```bash
# 저장소에서, 두 파일의 env 키만 뽑아 비교
git show <릴리스태그>:deploy/docker-compose/docker-compose.yaml > /tmp/up.yaml
diff <(grep -oE '^[[:space:]]+[A-Z0-9_]+:' /tmp/up.yaml | sort -u) \
     <(grep -oE '^[[:space:]]+[A-Z0-9_]+:' deploy/js-server/docker-compose.yaml | sort -u)
```
