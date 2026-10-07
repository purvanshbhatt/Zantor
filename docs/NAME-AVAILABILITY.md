# Name availability check (M0)

Checked 2026-10-06 via web search (GitHub / Play / F-Droid / domains, best effort).

| Name | Verdict | Notes |
|---|---|---|
| Aegis | **Blocked** | Aegis Authenticator (well-known F-Droid app), same store |
| Sentinel / Sentinal | Blocked / misspelled | Many security products |
| Terminus | **Blocked** | Eugeny's Terminus (now Tabby) — major existing terminal app |
| Halcyon | Crowded | Android music player (Kifranei/Halcyon), halcyonterminal.com, Halcyon Android ROM org, SIP client |
| Axis | Crowded | Generic high-collision name (Apache Axis, Axis cameras) |
| Nullpoint | Blocked | nullpoint.engineering (AI studio), NullPoint-Client repo |
| TermForge | Blocked | Multiple GitHub projects (terminal setup tools) |
| Agentron | Blocked | agentron.rocks AI agent platform, PyPI package |
| Vektor/Onyx/Meridian terminal | Crowded | Existing repos/products |
| **Kersos** | **Clear** | No software presence found (only unrelated Greek/Indonesian social accounts) |

## Recommendation
Working codename stays **Terminus** for local dirs only (`com.zantor.terminal`
package ID must change before release — do not ship under a blocked name).
Final candidate: **Kersos** (clear), or invent a new coinage.
Blocked: Aegis, Terminus, Nullpoint, TermForge, Agentron, Halcyon, Axis.

## TODO (user action, needs accounts)
- [ ] Reserve GitHub org + domain for the chosen final name
- [ ] USPTO trademark search before public release
