# Bias — MemWal 기반 장기 기억 아키텍처

> 코드 기준으로 정리한 MemWal 활용 문서. 인용된 경로/라인은 실제 구현을 가리킨다.

## 1. 우리가 푸는 문제

기존 캐릭터챗은 대화가 세션 단위로 끊겨, 캐릭터가 사용자를 **기억하지 못하고 매번 리셋**된다. Bias는 "캐릭터가 장기 기억을 유지하면서 관계가 깊어지는" 경험을 핵심 차별점으로 삼고, 이를 **MemWal**로 구현했다.

> 장기 기억은 Walrus에 저장하고 Seal로 보호하며 MemWal로 검색해, "기억이 있는 AI 관계"를 기술적으로 증명한다. — `docs/bm.md`

## 2. MemWal이란 무엇이고, 우리가 어떻게 쓰는가

MemWal은 Mysten의 SDK(`@mysten-incubation/memwal@^0.0.5`, `bias-web/package.json`)로, **Walrus 저장 + Seal 암호화 + 임베딩 기반 의미 검색**을 하나로 묶은 탈중앙 기억 레이어다. Bias는 두 진입점을 사용한다.

| 모듈 | 용도 | 사용 위치 |
|---|---|---|
| `@mysten-incubation/memwal/account` (`createAccount`, `addDelegateKey`) | 사용자 기억 계정 생성·서버 위임키 등록 | `bias-web/app/onboarding/account/page.tsx` |
| `@mysten-incubation/memwal/manual` (`MemWalManual`) | 기억 저장(`rememberManual`)·회수(`recallManual`) | `bias-web/app/api/chat/message/route.ts` |

### 2.1 온보딩 — 사용자 소유의 기억 지갑 만들기

기억은 서버가 아니라 **사용자 계정(Sui 기반)에 귀속**된다. `onboarding/account/page.tsx`의 흐름:

1. 사용자가 지갑으로 `BIAS_MEMWAL_ONBOARD_STATE` 메시지에 서명(소유 증명).
2. `createAccount(...)` → MemWal `accountId` 발급.
3. `addDelegateKey(...)`로 **서버 위임키 2종** 등록 — 라벨 `bias-server-memwal`(쓰기/검색), `bias-server-seal`(복호화).
4. 단계별 상태(`pending → creating → delegate_memwal → delegate_seal → done`)를 Supabase `users` 테이블에 기록(`lib/users.ts`의 `memwal_account_id`·`memwal_step`).

이 위임 구조 덕분에 **사용자가 기억의 소유권을 갖되, 서버가 대화 중 대신 읽고 쓸 수 있다.** 키/패키지 설정은 `config/route.ts`가 환경변수로 내려준다(`MEMWAL_PACKAGE_ID`, `MEMWAL_REGISTRY_ID`, `SERVER_DELEGATE_PUBKEY` 등).

### 2.2 클라이언트 생성

대화 요청마다 `accountId`별로 `MemWalManual` 클라이언트를 만들어 캐싱한다(`route.ts`). 임베딩은 OpenAI 키를 주입해 MemWal 내부에서 생성한다.

```ts
MemWalManual.create({
  key: requireEnv("MEMWAL_PRIVATE_KEY"),
  serverUrl: process.env.MEMWAL_RELAYER_URL ?? "https://relayer.staging.memwal.ai",
  suiPrivateKey: requireEnv("SERVER_SUI_PRIVATE_KEY"),
  embeddingApiKey: requireEnv("OPENAI_API_KEY"),   // 의미 검색용 임베딩
  packageId: requireEnv("MEMWAL_PACKAGE_ID"),
  accountId, namespace, suiNetwork, suiClient,
})
```

캐시는 TTL 10분·최대 200개, 만료 시 `client.destroy()`로 정리하고 에러가 나면 즉시 폐기 후 재생성한다.

## 3. 핵심 설계: Namespace로 기억을 격리한다

모든 기억은 다음 **네임스페이스 키**로 분리된다(`route.ts`의 `toNamespace`):

```
room:{roomId}:char:{characterId}:user:{userAddress}
```

기억의 단위는 **(방 × 캐릭터 × 사용자)** 조합이다. 이 설계가 중요한 이유:

- **1:1 기억이 단체방으로 새지 않는다** — 같은 캐릭터라도 방이 다르면 네임스페이스가 다름.
- **같은 캐릭터가 여러 사용자와의 관계를 구분한다** — `userAddress`가 키에 포함됨.
- privacy 측면에서 Seal 암호화 + namespace 분리가 함께 동작.

네임스페이스의 생명주기는 Supabase RPC로 조율한다(`lib/memwal-namespace-state.ts`): `ensureNamespaceState`로 초기화하고, 첫 메시지의 seed 쓰기는 `claimSeedWrite`로 **단 한 번만** 일어나도록 잠금(동시 요청 레이스 방지).

## 4. 메시지 1턴의 기억 흐름

`bias-web/app/api/chat/message/route.ts`의 `POST` 핸들러가 전 과정을 오케스트레이션한다.

```
요청(text, roomId, userAddress, characterId)
  │
  ├─ namespace 계산 + ensureNamespaceState
  │
  ├─ [첫 메시지만] claimSeedWrite → writeNamespaceSeed(페르소나) → finalize
  │       seed payload = [Seed Type] + [Namespace] + [Character Persona]
  │       SHA-256 fingerprint로 멱등성 보장
  │
  ├─ recallManual(query=text, {limit:5, namespace})   ← 기억 회수
  │
  ├─ 방 최근 30개 메시지로 단기 history 구성
  │       user(...) / assistant(self:char) / character(otherChar) 로 화자 구분
  │
  ├─ LLM 호출: system = 페르소나 + [Recalled Memories] 주입
  │
  ├─ 중요도 판정 (LLM + 한국어 규칙 + guardrail)
  │
  └─ rememberManual(text, namespace)  ← LOW가 아니면 저장
          저장 형식: "user(addr): ...\nassistant(self:char): bubble1 | bubble2"
```

### 4.1 회수(Recall) — "저장만 하면 기억이 아니다"

응답 직전에 사용자 메시지를 쿼리로 `recallManual`을 호출해 **의미적으로 가까운 과거 기억 top 5**를 가져오고, 이를 시스템 프롬프트의 `[Recalled Memories]` 블록에 주입한다. 이 "검색→프롬프트 주입"이 캐릭터가 과거를 기억해 말하게 만드는 부분이다.

> 저장 ≠ 기억 — 의미가 생기려면 응답 직전에 관련 기억을 검색해서 프롬프트에 주입해야 한다. — `bias-chat/docs/LONGTERM_MEMORY.md`

### 4.2 저장(Remember) — 중요도 게이팅으로 자연 망각

모든 발화를 저장하지 않고, **중요도가 LOW면 버린다**. 중요도는 3중 판정:

1. **LLM 판정** — 응답 JSON에 `importance: HIGH|MED|LOW` 포함.
2. **규칙 폴백** — `기억/약속/비밀/좋아해/싫어해` 등 한국어 키워드로 추론.
3. **Guardrail** — 규칙이 LLM보다 높은 중요도를 잡으면 상향 보정(중요 정보 누락 방지).

이렇게 임베딩/저장 비용을 관리하면서 중요한 기억만 남긴다.

## 5. 단체 채팅 & 다른 캐릭터와의 대화 기억

우리 서비스의 확장 축이다. **현재 구현된 범위**와 **추후 강화할 범위**를 구분하면:

**지금 코드로 동작하는 범위:**
- 방(room) 모델은 `direct`/`group` 타입을 지원한다(`lib/rooms.ts`).
- 메시지 라우트는 단기 history 구성 시 **다른 캐릭터의 발화를 `character({characterId}): ...`로 구분해 LLM에 함께 전달**한다. 시스템 프롬프트가 "`character(...)`로 시작하는 줄은 다른 참가자이며 그들을 대신 말하지 말라"고 명시해 정체성 혼선을 막는다.
- 즉 **한 방 안에서 여러 캐릭터가 서로의 대화를 인지**하는 단체챗이 동작한다.

**추후 강화 예정 (현재 코드 기준 한계):**
- MemWal **장기 회수는 자기 네임스페이스 1개로 한정**되어 있다(`recallManual`이 단일 `namespace`만 받음). 다른 캐릭터와의 대화 인지는 지금은 **최근 30개 메시지라는 단기 윈도우**로만 이뤄지고, MemWal 장기 기억으로는 교차되지 않는다.
- 저장 시에도 자기 발화(`assistant(self:...)`)만 기록하고 다른 캐릭터 발화는 MemWal에 넣지 않는다.
- 강화 방향: `recallManual`을 다중 네임스페이스/권한 범위로 확장하거나, 방·캐릭터를 가로지르는 상위 네임스페이스를 추가해 "다른 캐릭터와의 대화까지 회수"하도록 만드는 것.

## 6. 데이터 역할 분리 (설계 원칙)

`bias-chat/docs/LONGTERM_MEMORY.md`의 핵심 결정:

- **정확해야 하는 값(팩트·친밀도)** → 로컬(Supabase)을 SoT(기준)로.
- **많이 쌓이고 검색으로 꺼내는 일화** → MemWal을 검색 백본으로.
- 저장소는 `store/retrieve/update/forget` 인터페이스 뒤로 추상화해 SQLite ↔ MemWal ↔ pgvector를 교체 가능하게 설계.

`bias-chat`의 Python 프로토타입(`builders/memory_store.py`)은 이 인터페이스를 SQLite/JSON으로 먼저 검증한 단계이고, 프로덕션 웹앱(`bias-web`)에서 **MemWal로 교체·고도화**한 형태다.

## 부록: 코드만으로 단정하기 어려운 부분

1. **`recallManual`/`rememberManual` 내부 동작** — Walrus 저장·Seal 암호화·임베딩 검색의 정확한 흐름은 SDK 내부 구현이라 우리 코드에선 보이지 않는다. "Walrus 저장 + Seal 보호 + 의미 검색"은 `docs/bm.md` 서술과 SDK 옵션(`embeddingApiKey`, seal 위임키)으로 추론한 것.
2. **단체방 다중 캐릭터 호출 방식** — 메시지 라우트는 캐릭터 1명 기준이다. 단체방에서 캐릭터별로 라우트를 반복 호출하는지 등 클라이언트 오케스트레이션은 `rooms/[id]/page.tsx`에서 추가 확인 필요.
3. **`closeness_total`/`level`** — 응답 스키마에 친밀도 누적·레벨 필드가 있으나 라우트에서는 0/`null`로 고정 반환된다. 실제 친밀도 누적 로직 위치는 별도 확인 필요.
