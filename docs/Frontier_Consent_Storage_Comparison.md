# Frontier Consent/Control — Persistent State Storage: Approach Comparison Matrix

> **Status:** Draft v0.1 — Created: 2026-05-08  
> **Author:** Abhishek Bag (abbag) — _Action item from SSC sync, May 5 2026_  
> **Context:** Comparing three candidate approaches for where and how the Frontier program consent/control state is persistently stored, so it is durable, cross-platform, and accessible at boot to all clients (WAC, Native Win32, Android, iOS).

---

## 1. Background

**Frontier** is a Microsoft 365 opt-in program enabling consenting users to share usage data for AI improvements. Before a user can be enrolled, three things must be established:

1. **Eligibility** — age gate (MSA "Adult" group), geo gate (~12 excluded countries), and license check via OLS.
2. **Consent** — user explicitly opts in via a settings UI (toggles, per CELA/privacy guidance).
3. **Persistent state** — the consent decision must be **stored durably** and surfaced to every client at boot, across devices and platforms.

This document compares the three candidate approaches for **#3 — where the persistent state lives**:

| # | Approach |
|---|----------|
| **A** | **UCS — Unified Consent Service** |
| **B** | **Roaming Client** (SISU team) |
| **C** | **ECS + Frontier Persistence Service** (current WAC approach) |

_Source discussions: SSC-Sync Internal chat (Apr–May 2026), Frontier: age/geo gating requirements meeting (Apr 7), UC Sync for Frontier consents (Apr 7), Continue Frontier requirements discussion (Apr 29–30), Frontier Consent Requirements.docx (Gabrielle Stadlen), Approach Comparison.loop (Shreeya Singh, May 5)._

---

## 2. Approach Descriptions

### A — UCS: Unified Consent Service
A dedicated **server-side consent management platform** designed to be the single source of truth for all user consent decisions across Microsoft 365 products. UCS provides APIs for clients to write (grant/revoke) consent and read consent state, with built-in audit trails and compliance support. The team is investigating this as a clean, purpose-built option for Frontier (ADO Feature #11445677 — UC Frontier consents).

### B — Roaming Client (SISU team / Sridhar Dantuluri's team)
The **Roaming Client** is an existing infrastructure component owned by the SISU/SSC team. It syncs per-user settings/state across devices using a roaming service. On web, the relevant abstraction is `IRoamingSettingCache` (`ooui/packages/ono-box4/src/Client/RoamingSetting/IRoamingSettingCache.ts`). POCs are planned for Win32, Android, and Web. Open questions: can the roaming client push data to external services? Can it serve web apps? Can it communicate with services beyond the roaming service itself?

### C — ECS + Frontier Persistence Service (Current WAC approach)
The **existing production approach** used by the WAC (Web App Client). The user's consent decision is written to a **dedicated Frontier Persistence Service** (owned by Elie Aoun / eaoun's team), which then **updates ECS** (Experiment Configuration Service). WAC reads the `FrontierAudience` ECS segment at boot. ECS is not the source of truth — it's the runtime read path. The Persistence Service is the source of truth.

---

## 3. Comparison Matrix

| Criterion | **A: UCS (Unified Consent Service)** | **B: Roaming Client (SISU team)** | **C: ECS + Frontier Persistence Service** |
|---|---|---|---|
| **Purpose-fit for consent** | ✅ Designed specifically for consent management across M365 | ⚠️ General-purpose settings roaming; not consent-specific | ⚠️ Frontier-specific persistence service; not a general consent platform |
| **Source of truth** | ✅ Yes — dedicated consent record per user | ✅ Yes — roaming store is authoritative for settings | ✅ Yes — Persistence Service is authoritative; ECS is derived |
| **Cross-platform (WAC, Win32, Android, iOS)** | ✅ Service-based API; any client can call it | ⚠️ Under investigation — POCs planned for Win32, Android, Web; coverage unconfirmed | ⚠️ WAC only today; extension to Native/Mobile requires Elie's team |
| **Cross-device / Roaming** | ✅ Server-side; device-agnostic | ✅ By design — built for roaming across devices | ✅ Server-side; not device-bound |
| **Boot-time availability** | ⚠️ Network call required to read consent state | ✅ Roaming cache available locally after first sync | ✅ ECS segment available at boot (after persistence sync) |
| **Read latency** | Medium — network call each time unless cached | Low — local roaming cache (fast local read) | Low — ECS segment read at boot is fast |
| **Write / update flow** | ✅ Direct API call to UCS; clean write path | ⚠️ Roaming push capability TBD — unclear if client can push writes outbound | ✅ Client calls Persistence Service; Persistence Service updates ECS |
| **Revoke / opt-out support** | ✅ First-class — consent services handle grant/revoke natively | ⚠️ Possible via roaming setting update, but not purpose-built for consent lifecycle | ✅ Supported — state change via Persistence Service propagates to ECS |
| **Tenant admin / org-level control** | ✅ Likely supports org-level policy; needs confirmation | ⚠️ Per-user roaming only; org-level not in scope today | ⚠️ User-level today; tenant admin control discussed but not implemented |
| **Audit trail / compliance** | ✅ Strong — consent timestamp, actor, and version natively tracked | ❌ Weak — roaming settings not audit-logged as consent records | ⚠️ Moderate — Persistence Service logs changes but audit trail less explicit |
| **Privacy / CELA alignment** | ✅ Strong — consent data clearly separated, purpose-built | ⚠️ Moderate — consent mixed with general settings; separation unclear | ⚠️ Moderate — Frontier-specific but requires careful data boundary enforcement |
| **Implementation complexity for SISU team** | Medium — new service integration; needs UCS team engagement | Low — SISU team owns the infrastructure; no new external dependency for writes | Medium–High — need to integrate with Elie Aoun's team; different ownership boundary |
| **Dependencies** | UCS team (owner TBD); consent API design | SISU/Roaming team internally; Alberto's team for API contracts | Elie Aoun (eaoun)'s team for Persistence Service; ECS team |
| **Current maturity** | ❓ Under investigation — not yet adopted for Frontier | ⚠️ Existing infra; POCs in progress (Win32, Android, Web) | ✅ Production on WAC; not yet on other platforms |
| **Scalability** | ✅ Service-based; scales independently | ✅ Roaming service already at M365 scale | ✅ Frontier Persistence Service built for M365 scale |
| **Test/override support** | ❓ TBD — test environment design needed | ⚠️ In investigation (dogfood/preauth issues seen in testing, Apr 2026) | ✅ URL param `ForcedCopilotFrontier` available; PPE/SIP event hub issues being resolved |
| **Extensibility (future consents)** | ✅ Designed to manage multiple consent types; future-proof | ⚠️ General settings store; not designed for multi-consent management | ❌ Frontier-specific only; would need re-engineering for other consents |
| **Cost** | ❓ TBD — SWAG in progress (Sangeetha Muthurajan, Apr 30) | ❓ TBD — depends on roaming service scale for new setting | ❓ TBD — ongoing cost of running Persistence Service |

---

## 4. Detailed Dimension Analysis

### 4.1 Cross-Platform Coverage
This is a critical requirement — Frontier must work across WAC, Win32, Android, and iOS:
- **UCS:** API-first; any platform can call it. Most flexible if the API is stable.
- **Roaming Client:** SISU team is running POCs for each platform (Madhu Kumar — Android, Abhishek Malviya + Shreeya — Win32, Shreeya — Web roaming). Coverage is promising but not yet proven.
- **ECS + Persistence:** Currently WAC-only. Extending to Native/Mobile requires Elie Aoun's team to expose new endpoints, adding external dependency.

### 4.2 Boot-time Read Path
A key architecture question (Madhu Kumar's action item — "Boot flow: ECS vs Roaming"):
- **ECS:** Segment is surfaced at boot after sync; reliable for WAC. For new platforms, boot-time ECS integration needs to be validated.
- **Roaming Cache:** Local cache is fast; but first-boot sync delay applies. Need to confirm behavior when cache is cold (first install, new device).
- **UCS:** May need a network call at boot or a local cache layer.

### 4.3 Push/Write Capability
From Apr 30 action items — open question: _"Is the config client able to push data?"_ and _"Can the roaming client communicate with other services?"_
- **UCS:** Write = API call to UCS directly. Straightforward.
- **Roaming Client:** Native roaming syncs settings; unclear if outbound push to external services (e.g., for cascading effect to ECS or Persistence Service) is supported. Under investigation.
- **ECS + Persistence:** Client writes to Persistence Service; Persistence Service pushes to ECS. Well-understood flow.

### 4.4 Org/Tenant Admin Control
Discussed Apr 29 — "tenant admin control" is a key consideration:
- **UCS:** Likely has admin policy support; needs confirmation.
- **Roaming:** Per-user setting; org-level override may not be native.
- **ECS + Persistence:** Currently user-level; tenant admin control not implemented.

---

## 5. Preliminary Recommendation

> ⚠️ _This is a draft — to be discussed and validated with the team and Frontier stakeholders._

| Priority | Approach | Rationale |
|---|---|---|
| 🥇 **Preferred (long-term)** | **A — UCS** | Purpose-built for consent, extensible to future consent types, strong compliance/audit story, service-based cross-platform |
| 🥈 **Viable (near-term)** | **B — Roaming Client** | SISU team owns it — lowest external dependency; POCs in progress; but needs investigation on push, web, and audit |
| 🥉 **Baseline (WAC only)** | **C — ECS + Persistence** | Already in production on WAC; well-understood; but Frontier-specific, not extensible, adds dependency on Elie's team |

**Possible phased approach:**
1. **Phase 1 (WAC):** Use existing ECS + Persistence Service to unblock WAC quickly.
2. **Phase 2 (All platforms):** Adopt Roaming Client or UCS depending on POC outcomes and UCS availability.
3. **Long-term:** Migrate all to UCS as the single consent platform.

---

## 6. Open Questions

| # | Question | Owner | Status |
|---|----------|-------|--------|
| 1 | Can the Roaming Client push data/writes to external services? | Sahil / Madhu / Shreeya | ❓ Under investigation |
| 2 | Does the Roaming Client support web (WAC)? What is the web roaming story? | Shreeya (IRoamingSettingCache.ts owners) | ❓ Under investigation |
| 3 | Can UCS handle Frontier consent — what is the API and onboarding path? | UCS team (owner TBD) | ❓ Not started |
| 4 | What is the boot-time behavior of each approach on a cold start / new device? | Madhu Kumar | ❓ Boot flow work item in progress |
| 5 | Can the ECS + Persistence Service approach be extended to Native and Mobile clients? | Elie Aoun's team | ❓ Pending outreach |
| 6 | What is the tenant admin control story for each approach? | Sangeetha / Gabrielle | ❓ Open |
| 7 | What are the cost estimates (SWAG) for each approach at FY27 Q1–Q4 scale? | Sangeetha Muthurajan | ❓ In progress |
| 8 | Should we initiate conversation with Alberto's team on proposed architecture? | Madhu Kumar | ❓ Scheduled for next meeting |
| 9 | Does UCS support both MSA (consumer) and AAD (commercial) user populations? | UCS team | ❓ Open |
| 10 | What are the PPE/SIP testing environment needs for each approach? | Abhishek Bag / Rachel Purtill | ❓ Event-hub auth issues being resolved for Roaming/PDRS |

---

## 7. Stakeholders

| Name | Team / Role | Relevant Approach |
|------|-------------|------------------|
| Sridhar Dantuluri | SISU/SSC Eng Lead | Roaming Client (B) |
| T Madhu Kumar | SISU/SSC Eng — boot flow, Android POC | B, C |
| Shreeya Singh | SISU/SSC Eng — Win32 POC, Web Roaming | A, B |
| Sahil Sahil | SISU/SSC Eng — Roaming onboarding | B |
| Abhishek Malviya | SISU/SSC Eng — Win32 POC | B |
| Anmol Srivastava | SISU/SSC Eng | B |
| Abhishek Bag | SISU/SSC Eng — author of this doc | All |
| Elie Aoun (eaoun) | Frontier Persistence Service owner | C |
| Rob Rolnick | WAC Eng | C |
| Gabrielle Stadlen | PM — Frontier requirements | All |
| Risa Naka | PM — Data collection / UX | All |
| Sangeetha Muthurajan | PM — cost / requirements | All |
| Amit Jain | Eng — UC Frontier consents | A |
| Alberto's team | API contract / platform | B |
| Jason Zhang | WAC / OLS | C |
| Alyssa Nelson | Client UX | C |

---

## 8. References

- SSC-Sync Internal chat — Apr–May 2026 (Sridhar, Shreeya, Madhu, Sahil et al.)
- Approach Comparison.loop — Shreeya Singh (May 5, 2026) _(initial high-level comparison)_
- Requirements.loop — Shreeya Singh (May 5, 2026) _(Phase 1 requirements)_
- Frontier Consent Requirements.docx — Gabrielle Stadlen (Apr 29–30, 2026)
- ADO Feature #11445677 — UC Frontier consents
- Meeting: _Frontier: age/geo gating requirements_ — Risa Naka (Apr 7, 2026)
- Meeting: _UC Sync for Frontier consents_ — Sridhar Dantuluri (Apr 7, 2026)
- Meeting: _Continue Frontier requirements discussion_ — Gabrielle Stadlen (Apr 29–30, 2026)
- IRoamingSettingCache.ts — `ooui/packages/ono-box4/src/Client/RoamingSetting/`

---

_Draft v0.1 — Please review, add corrections, and update open questions as POC results come in._
