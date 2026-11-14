# Evidence index

All evidence was produced against the running Docker Compose stack in this repository.

**Clocks.** Evidence timestamps are from the lab host (2026-11-14 UTC). The portal screenshots show GST (UTC+4). The Grafana screenshot shows the browser’s local time (IST, UTC+5:30); its 11:20–11:40 window is 05:50–06:10 UTC. The exercise clock (Monday 16 November 2026, Asia/Dubai) is a separate scenario clock used for the replies; the absolute lab times are not meant to match the exercise times. The durations and the relative order in each log are the evidence.

| Evidence window (UTC) | Portal time (GST) | What it covers |
|---|---|---|
| 05:54–05:58 | 09:54–09:58 | Incident 1 reproduction (first upload and duplicate retry) |
| 05:58–05:59 | 09:58–09:59 | Incident 3 (Al Noor) and Incident 4 (Sahara) reproductions |
| 06:01–06:05 | 10:01–10:05 | Incident 2 `make load` peak and OOM restarts |
| 06:11 | 10:11 | Incident 5 corrected-file verification |

## Incident 1 — Oasis upload stuck, then duplicate document numbers

- `logs/incident1-api-import.log` — API log: 600 rows parsed in 0.1 s, import completed after 121,007 ms; retry completed after 121,013 ms with 8 rejected.
- `logs/incident1-web-proxy.log` — Next.js proxy returned 504 after 60,002 ms.
- `queries/incident1-imports.txt` — imports and invoices: first attempt 8 Accepted with ASP message ids; retry 8 Rejected AE-BR-09.
- `queries/incident1-participant-lookups.txt` — Prometheus counter: 0 → 600 participant lookups over 120 s (200 ms per lookup).
- `screenshots/incident1-oasis-first.png` — portal: first batch, 8 Accepted.
- `screenshots/incident1-oasis-retry-duplicates.png` — portal: retry, 8 Rejected by portal.
- `screenshots/incident1-oasis-duplicate-error.png` — invoice detail with the AE-BR-09 duplicate message.

## Incident 2 — API restarts / 502s / stuck invoices

- `logs/incident2-api-startup.log` — three extra API startup sequences at 06:03:37, 06:04:13 and 06:05:15 UTC.
- `logs/incident2-docker-events.txt` — Docker OOM event and exit 137; `RestartCount=3`.
- `logs/incident2-loadgen.log` — peak run: 240 uploads, 34 failed with EOF / connection refused / timeout; 20 imports left Processing, 1 Uploaded.
- `queries/incident2-prometheus.txt` — `changes(process_start_time_seconds[15m]) = 3`; process start times; working set climbing to the 128 MiB limit.
- `queries/incident2-memory-samples.txt` — memory samples in the minutes before each restart.
- `queries/incident2-stuck.txt` — invoice and import status counts after the peak.
- `screenshots/incident2-grafana-overview.png` — Grafana dashboard for 05:50–06:10 UTC (shown as 11:20–11:40 in the browser’s IST): process restarts = 3, container restarts, memory, import processing time, participant lookups.

## Incident 3 — Al Noor BR-CO-15

- `queries/incident3-alnoor.txt` — six invoices: 0041/0043/0045 Accepted, 0042/0044/0046 Rejected by ASP with BR-CO-15; payload totals.
- `logs/incident3-asp.log` — ASP log lines with the exact got/expected amounts.
- `screenshots/incident3-alnoor-rejected.png` — portal batch page: 3 Accepted, 3 Rejected.
- `screenshots/incident3-alnoor-br-co-15-error.png` — invoice ANF-2611-0042: total with VAT 15,022.42 vs expected 15,022.43.

## Incident 4 — Sahara invoices stuck in Submitted

- `queries/incident4-sahara.txt` — original import: 32 Submitted, 8 Accepted; ASP only has the 8 original invoice numbers; retry endpoint requeues 0; Prometheus shows 32 throttled submissions.
- `logs/incident4-asp-429.log` — 8 accepted immediately followed by 32 `rate limit exceeded` warnings at 05:59:49 UTC.
- `screenshots/incident4-sahara-submitted.png` — portal batch page: Submitted 32, Accepted 8, empty ASP references.

## Incident 5 — Ghaf date format

- `queries/incident5-ghaf.txt` — all four invoices rejected by the portal with AE-BR-04; the corrected file (same content, dates as YYYY-MM-DD) accepted all four.
- `queries/incident5-date-cells.txt` — cell-by-cell diff of the corrected workbook: 16 date cells changed, no other cell changed.
- `screenshots/incident5-ghaf-rejected.png` — portal batch page: Rejected 4.
- `screenshots/incident5-ghaf-date-error.png` — invoice detail with the date-format error.
- `ghaf-cafe-nov-2026-corrected.xlsx` — the corrected file used for verification (16 date cells changed to YYYY-MM-DD; no other change).

## Code references

- `code/references.md` — `file:line` excerpts for each root cause.
