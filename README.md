# TollGate implementation map

Visual map of **every** TollGate implementation: what exists, which protocol each speaks, and how
they relate. Built because the fleet is larger than it looks — **six implementations across two
different protocols** — and the relationships were only discoverable by reading six repos.

**Evidence date: 2026-10-18.** Every claim here is measured, not inferred from a README. Method and
reproduction commands: [METHOD.md](METHOD.md).

---

## 1. The headline: two protocols share one name

```mermaid
flowchart TB
    subgraph SPEC["📜 OpenTollGate/tollgate — the spec repo"]
        direction TB
        TIP01["TIP-01<br/>Base events<br/>10021 / 1022 / 21023"]
        TIP02["TIP-02<br/>Cashu payments<br/>price_per_step"]
        HTTP01["HTTP-01<br/>POST / :2121<br/>raw token body"]
        HTTP02["HTTP-02<br/>GET /whoami"]
        HTTP03["HTTP-03<br/>GET /usage"]
        NOSTR01["NOSTR-01<br/>⚠️ STUB (5 lines)<br/>RESERVED"]
        WIFI01["WIFI-01<br/>⚠️ STUB (5 lines)<br/>RESERVED"]
        CORPUS["corpus/ v1<br/>33 fixtures<br/>decode→verify→checkstate→spend<br/>consumer_ids + known_divergences"]
    end

    subgraph V1["🟦 V1 protocol family — what is deployed today"]
        direction TB
        GO["tollgate-module-basic-go<br/>Go · 294 files · 1123 tests<br/>gonuts → CDK sidecar<br/>OpenWrt: apk/ipk · procd · LuCI"]
        RSBASIC["tollgate-module-basic-rust<br/>⚠️ DORMANT 2026-07-21<br/>18 commits · 58 tests<br/>personal fork · cdk 0.17"]
        PORTAL["tollgate-captive-portal<br/>browser decoder<br/>corpus consumer"]
    end

    subgraph V2["🟧 V2 protocol family — vouchers + channels"]
        direction TB
        RSORG["OpenTollGate/tollgate-rs<br/>rc.2 · 475 tests · 8 topologies<br/>CBOR + Spilman · mintd/merchantd<br/>no_std core"]
        SUITE["tollgate-suite-wrt<br/>OpenWrt + FIPS exit<br/>proxyd / exitd · 150 tests"]
        AMP["Amperstrand/tollgate-rs-ai-research<br/>⚠️ DIVERGED +94/-117<br/>has v1_compat/ 5,594 LOC"]
    end

    CORPUS -.->|"certifies"| GO
    CORPUS -.->|"certifies"| PORTAL
    CORPUS -.->|"NOT a consumer"| RSBASIC
    CORPUS -.->|"NOT a consumer"| RSORG

    TIP01 --> GO
    TIP02 --> GO
    HTTP01 --> GO
    HTTP02 --> GO
    HTTP03 --> GO
    TIP01 -.->|"partial: 1022 missing p+device-id"| RSBASIC
    HTTP01 -.->|"yes"| RSBASIC

    NOTIP["⛔ no TIP conformance<br/>0 hits for 10021/1022/21023<br/>no :2121 HTTP surface"]
    RSORG ==> NOTIP

    AMP ==>|"v1_compat/<br/>implements 10021/1022/21023"| TIP01

    GOCDK["go-cdk build tag<br/>cdk_wallet.go<br/>not on any policy rule"]
    GO -.-> GOCDK

    classDef spec fill:#1e3a5f,stroke:#4a90d9,color:#e8f0f8
    classDef v1 fill:#1a3a2a,stroke:#4caf50,color:#e8f8ee
    classDef v2 fill:#3a2a1a,stroke:#ff9800,color:#fdf0e0
    classDef warn fill:#3a1a1a,stroke:#f44336,color:#ffe8e8
    class TIP01,TIP02,HTTP01,HTTP02,HTTP03,CORPUS spec
    class NOSTR01,WIFI01,RSBASIC,AMP warn
    class GO,PORTAL,GOCDK v1
    class RSORG,SUITE v2
```

**The two protocols, stated plainly:**

| | 🟦 V1 (TIP/HTTP) | 🟧 V2 (vouchers/channels) |
|---|---|---|
| Payment unit | a Cashu token worth **N steps** of time/bytes | **1 voucher = 1 byte**, from the node's own mint |
| Who mints | external mints (operator allowlist) | **the node itself** (`mintd`) |
| Settlement | direct token swap per purchase | **Spilman payment channel**, 2-of-2, rollover |
| Metering | whole steps, session allotment | **reserved rate + per-second draws**, `burst` |
| Budget durability | session metering **in memory** (documented gap) | **`budgets-<instance>.json`, survives restart** |
| Wire format | HTTP `:2121` + Nostr events 10021/1022/21023 | **CBOR over raw TCP**, normative CDDL |
| Browser path | native (captive portal is the point) | **fallback only** (`proxyd` in suite-wrt) |
| Canonical client | browser / any HTTP client | **`tollgate_net::client::run`** (agents, native) |
| Spec location | `OpenTollGate/tollgate` TIP-01/02, HTTP-01/02/03 | `tollgate-rs/docs/design/` + `tollgate.cddl` |

---

## 2. Lineage — and the bridge nobody tracks

```mermaid
flowchart LR
    ROOT["TollGate v1<br/>TIP/HTTP<br/>Go reference"]

    ROOT -->|"port to Rust<br/>CDK instead of gonuts<br/>2026-07-21 · 18 commits"| CLONE["tollgate-module-basic-rust<br/>🟦 V1 protocol<br/>⚠️ DORMANT 88 days"]
    ROOT -->|"redesign<br/>vouchers + Spilman<br/>2026-04-19"| OLD["tollgate-rs<br/>(design phase)"]

    OLD -->|"release 0.1.0-rc.2<br/>2026-10-13"| ORG["OpenTollGate/tollgate-rs<br/>🟧 V2 protocol"]
    OLD -->|"branch divergence<br/>+94 commits"| AMP["Amperstrand/tollgate-rs-ai-research<br/>🟧 V2 + 🟦 V1 bridge"]

    ORG -->|"OpenWrt packaging"| SUITE["tollgate-suite-wrt<br/>🟧 V2 protocol"]

    AMP -.->|"v1_compat/ 5,594 LOC<br/>added AFTER the fork<br/>NOT in ORG"| V1BRIDGE["🟦 speaks 10021/1022/21023<br/>price_per_step · crowsnest"]

    MERGE{{"common ancestor<br/>be5400f"}}
    ORG --- MERGE
    AMP --- MERGE

    NOTIN["⛔ v1_compat is on a fork<br/>no upstream PR<br/>0 mentions in ORG repo<br/>not in corpus consumer_ids"]

    V1BRIDGE --> NOTIN

    classDef v1 fill:#1a3a2a,stroke:#4caf50,color:#e8f8ee
    classDef v2 fill:#3a2a1a,stroke:#ff9800,color:#fdf0e0
    classDef warn fill:#3a1a1a,stroke:#f44336,color:#ffe8e8
    classDef neutral fill:#2a2a2a,stroke:#888,color:#eee
    class CLONE,V1BRIDGE v1
    class ORG,SUITE,OLD,AMP v2
    class NOTIN warn
    class ROOT,MERGE neutral
```

### The `v1_compat/` discovery

`Amperstrand/tollgate-rs-ai-research-and-experiments` contains
`crates/tollgate-net/src/v1_compat/` — **16 modules, 5,594 lines** — implementing the V1 surface:

```
adapter.rs   client.rs    crowsnest.rs  handlers.rs     http_client.rs  ln_quotes.rs
mac_resolver.rs  merchant.rs  mint_health.rs  mod.rs  nostr.rs  pricing.rs
recovery.rs  session_manager.rs  usage_tracker.rs  wallet.rs
```

`v1_compat/nostr.rs` declares the kind table explicitly:

```
| 10021 | TollGateAdvertisement | TollGate discovery (advertisement) |
| 1022  | SessionEvent          | Session grant after successful payment |
| 21023 | NoticeEvent           | Error / warning / info notice |
```

**Proven to be post-fork:** the path does not exist at the shared ancestor `be5400f`
(`git cat-file` → *"exists on disk, but not in be5400f"*), and it was added by three commits after
the divergence.

**Why this matters:** the two designs are *already reconciled in code* — by one person, on a personal
fork, with **no upstream PR**, **zero mentions in the org repo**, and **no entry in the corpus's
`consumer_ids`**. It is the cheapest path to "multiple interoperable implementations" and it is one
`rm -rf` from gone.

---

## 3. What the test framework actually exercises

```mermaid
flowchart TB
    PRTA["physical-router-test-automation<br/>170 test files · 3 venues<br/>(QEMU VM / SHC cloud / physical router)"]

    subgraph BACKENDS["lib/backend.py — BACKEND_TYPES"]
        direction LR
        B1["go<br/>OpenTollGate/tollgate-module-basic-go"]
        B2["go-cdk<br/>same tree · -tags cdk_wallet<br/>needs CGO"]
        B3["rust<br/>Amperstrand/tollgate-rs-ai-research<br/>⚠️ not the org repo"]
        B4["rust-basic<br/>felixfelix-bot/tollgate-module-basic-rust"]
        B5["rust-embedded<br/>same repo · constrained target"]
    end

    PRTA --> BACKENDS

    B1 --> A["🟦 TIP/HTTP"]
    B2 --> A
    B4 --> A
    B5 --> A
    B3 --> B["🟧 vouchers/channels<br/>+ v1_compat bridge"]

    A --> TIPA["tests/api/* — TIP conformance<br/>:2121"]
    B --> TIPB["tests/api/test_rust_v1_api.py<br/>advertisement/pay/usage/balance/whoami"]

    NOTE["⚠️ test_rust_v1_api.py SKIPS itself<br/>for rust-basic and rust-embedded:<br/>'V1 API tests are for tollgate-rs,<br/>not tollgate-module-basic-rust'"]
    B3 -.-> NOTE

    classDef v1 fill:#1a3a2a,stroke:#4caf50,color:#e8f8ee
    classDef v2 fill:#3a2a1a,stroke:#ff9800,color:#fdf0e0
    classDef warn fill:#3a1a1a,stroke:#f44336,color:#ffe8e8
    classDef neutral fill:#2a2a2a,stroke:#888,color:#eee
    class B1,B2,B4,B5,A,TIPA v1
    class B3,B,TIPB v2
    class NOTE warn
    class PRTA neutral
```

**Naming trap:** PRTA's `--backend rust` does **not** mean `OpenTollGate/tollgate-rs`. It maps to
`Amperstrand/tollgate-rs-ai-research-and-experiments`. The org repo is not a test backend at all.
This cost a full wrong analysis before it was caught — see [METHOD.md](METHOD.md#correction-log).

---

## 4. Protocol layers and who fills them

```mermaid
flowchart TB
    subgraph L3["Medium — what physical link"]
        WIFI01["WIFI-01 beacon advertisement<br/>⚠️ STUB — 5 lines, RESERVED"]
    end
    subgraph L2["Interface — how peers talk"]
        HTTP01["HTTP-01 :2121"]
        HTTP02["HTTP-02 /whoami"]
        HTTP03["HTTP-03 /usage"]
        NOSTR01["NOSTR-01 relay :4242<br/>⚠️ STUB — 5 lines, RESERVED"]
    end
    subgraph L1["Protocol — what the data is"]
        TIP01["TIP-01<br/>10021 advert · 1022 session · 21023 notice"]
        TIP02["TIP-02<br/>price_per_step · multi-mint"]
    end

    subgraph IMPL["Who implements what"]
        direction LR
        GO["basic-go<br/>✅ all of L1+L2"]
        RSB["basic-rust<br/>◐ partial L1, ✅ L2"]
        RS["tollgate-rs<br/>⛔ none of L1/L2"]
        AMP["Amperstrand fork<br/>✅ L1 via v1_compat"]
    end

    L1 --> IMPL
    L2 --> IMPL
    L3 -.->|"nobody"| IMPL

    classDef stub fill:#3a1a1a,stroke:#f44336,color:#ffe8e8
    classDef ok fill:#1a3a2a,stroke:#4caf50,color:#e8f8ee
    classDef part fill:#3a3520,stroke:#ffc107,color:#fff8e1
    classDef none fill:#3a1a1a,stroke:#f44336,color:#ffe8e8
    class WIFI01,NOSTR01 stub
    class GO,AMP ok
    class RSB part
    class RS none
```

---

## 5. Capability matrix (measured)

| Capability | basic-go | basic-rust | tollgate-rs (org) | Amperstrand fork |
|---|---|---|---|---|
| TIP-01 10021 advertisement | ✅ | ✅ | ⛔ | ✅ via `v1_compat` |
| TIP-01 1022 session (`p`, `device-identifier`, `allotment`) | ✅ | ◐ `allotment`+`metric` only | ⛔ | ✅ via `v1_compat` |
| TIP-01 21023 notice (`level`, `code`) | ✅ | ◐ `{kind,content}` only | ⛔ | ✅ via `v1_compat` |
| TIP-02 `price_per_step` multi-mint | ✅ | ✅ | ⛔ | ✅ |
| HTTP-01 `POST /` :2121 raw body | ✅ | ✅ | ⛔ | ✅ |
| HTTP-02 `/whoami` | ✅ | ◐ | ⛔ | ✅ |
| HTTP-03 `/usage` | ✅ | ✅ | ⛔ | ✅ |
| Spilman payment channels | ⛔ | ⛔ | ✅ | ✅ |
| Budgets persisted across restart | ⛔ (session in memory) | ⛔ | ✅ | ✅ |
| Own mint (`mintd`) | ⛔ | ⛔ | ✅ | ✅ |
| External enforcer protocol | ⛔ | ⛔ | ✅ | ✅ |
| `no_std` core (embedded) | ⛔ | ⛔ | ✅ | ✅ |
| OpenWrt packaging | ✅ mature | ◐ | via suite-wrt | ✅ |
| LuCI web UI | ✅ | ? | ⛔ (`tolltop` TUI) | ? |
| Corpus consumer | ✅ | ⛔ | ⛔ | ⛔ |
| Tests (in-tree) | 1123 | 58 | 475 | ? |
| Last activity | 2026-10-17 | **2026-07-21** | 2026-10-13 | 2026-10-16 |

---

## 6. Known hazards, at a glance

| Hazard | Where |
|---|---|
| `NOSTR-01` and `WIFI-01` are **5-line RESERVED stubs**, yet the spec README offers recipes ("WiFi hotspot (full)", "Nostr-only") built on them | `OpenTollGate/tollgate` |
| `v1_compat/` bridge exists **only on a personal fork**, no PR, not in `consumer_ids` | `Amperstrand/…` |
| PRTA's `rust` backend is **not the org repo** | `lib/backend.py` |
| Known-good corpus state is **not green** — portal V2-keyset divergence open | `corpus/manifest.json` |
| `gonuts` upstream is **archived** (2025-08-16); fork can never merge back | `elnosh/gonuts` |
| `cdk-walletd` has **no packaging anywhere** yet policy points every target at its socket | `tollgate-module-basic-go` |
| `tollgate-rs` channels are **memory-only** across restart; buyer **cannot yet pay a seller** | rc.2 release notes |
| `tollgate-rs` is effectively **single-maintainer** (`Origami74` = 159/167 commits) | org repo |

---

## Reproduce this

```bash
# clone everything
git clone https://github.com/OpenTollGate/tollgate
git clone https://github.com/OpenTollGate/tollgate-module-basic-go
git clone https://github.com/OpenTollGate/tollgate-suite-wrt
git clone https://github.com/OpenTollGate/tollgate-rs
git clone https://github.com/Amperstrand/tollgate-rs-ai-research-and-experiments
git clone https://github.com/felixfelix-bot/tollgate-module-basic-rust
git clone https://github.com/OpenTollGate/physical-router-test-automation

# the protocol split, per repo (expect 0 for tollgate-rs)
for r in tollgate-module-basic-go tollgate-module-basic-rust tollgate-rs; do
  echo "== $r"; grep -ril "10021\|1022\|21023\|price_per_step" "$r" --include=*.rs --include=*.go | wc -l
done

# the bridge, and proof it is post-fork
ls tollgate-rs-ai-research-and-experiments/crates/tollgate-net/src/v1_compat/
cd tollgate-rs-ai-research-and-experiments
git remote add org ../tollgate-rs && git fetch org
git merge-base HEAD org/master          # be5400f
git cat-file -t be5400f:crates/tollgate-net/src/v1_compat   # fatal: not in be5400f
git rev-list --left-right --count HEAD...org/master          # 94  117
```

## Licence

MIT. The diagrams are Mermaid sources under `diagrams/` and are also embedded inline above
(GitHub renders them natively). If a diagram and a source repo disagree, **the repo wins** — please
open an issue with the evidence.
