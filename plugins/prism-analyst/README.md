# prism-analyst

Ask questions about your Prism deals in plain language — NAV, cash flows, portfolio composition, risk metrics — and get answers with confidence scores and source citations down to the document and page.

**Installation** (Claude Code or the Claude app): see the [main README](../../README.md).

## What you can ask

- *What's the NAV for `<deal name>` this period, compared to a year ago?*
- *How did realisations and distributions evolve for `<deal name>`?*
- *What's the sector and geography allocation? Any concentration risk?*
- *What are the top 10 holdings, and what changed since last year?*
- *What's the leverage and FX exposure for `<deal name>`?*

The skill activates automatically when you mention a deal name, NAV, KPIs, portfolio data, or similar — no special syntax needed. If it doesn't kick in, ask explicitly: "Use prism-analyst to look up...".

## How it works

Behind each answer, several specialized agents retrieve and cross-check data independently (performance, cash flow, portfolio composition, risk), then a final agent reconciles their findings, applies a confidence score to every data point, and cites the source document and page.

| Score | Meaning |
|---|---|
| 🟢 | Reliable — from structured tables or audited statements. |
| 🟡 | Use with caution — corroborated but not from a primary table. |
| 🔴 | Verify manually — uncorroborated, often extracted from charts/images. |

Chart/image data is treated with extra caution and never scored 🟢, since it comes from automated extraction rather than structured data. If any data point comes back low-confidence, you'll be asked to confirm or correct it before the final report is generated.

## Support

Questions or issues — contact your CardoAI representative, or dev@cardoai.com.
