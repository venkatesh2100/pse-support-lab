# Submission

- **Name:** Venky
- **Date submitted:** 16 November 2026 (exercise clock)
- **Repository link:** https://github.com/venkatesh2100/pse-support-lab
- **Total time spent (real time):** about 40–45 minutes of hands-on lab work in this AI-assisted session (time log at the end). The exercise suggests 6–8 hours for a human working unaided; the AI compressed the reproduction and drafting, and I verified every claim against the running system.

> Replies below are drafts written on the **exercise clock** (Monday 16 November 2026, Asia/Dubai). The real time is kept in the time log at the end. Screenshots, log excerpts and query output are in `evidence/` and linked from here.
>
> Evidence timestamps are from the lab host (2026-11-14 UTC); the portal screenshots show GST (UTC+4). See [`evidence/README.md`](evidence/README.md) for the lab-time and portal-time mapping. The exercise clock is a separate scenario clock; the absolute lab times are not meant to match it — the durations and relative order are the evidence.

- **Items I fully investigated, and why I chose them:** all five. The three with the greatest customer impact — Incident 1 (month-end upload), Incident 2 (platform restarts) and Incident 4 (silent non-delivery) — would have been enough for the minimum, but Incidents 3 and 5 were quick to reproduce and both needed a clear written answer. When Incident 2 arrived it became the platform-level lead because it affected several customers at once and could also explain the symptoms in Incident 1.

---

## Incident 1 — `inbox/01-oasis-builders-upload-stuck.md`

### Acknowledgement

- **Sent at (exercise clock):** 09:12, Monday 16 November 2026
- **Reply:**

> Hello Rashid,
>
> Thank you for the detail — I understand your November consolidated file sat on “Processing”, the retries now show duplicate document numbers, and you need to know which invoices actually went out before month-end.
>
> I am checking the batch and the submission records now. Please **do not upload the file again** until I confirm; I do not want to create more confusing duplicate attempts while I establish what the first upload did.
>
> I will reply by **10:15** with exactly which invoices are accepted and the safe next step.
>
> Kind regards,
> Venky — Product Support, Invoicely Lite

### Priority and justification

- **Priority:** **P1**
- **Why:** At triage I could not confirm that any of the 8 invoices had reached the ASP: the customer’s screen showed a failed upload, then duplicate-number errors, and it was month-end. A possible total delivery failure for one customer meets the SLA’s P1 definition (“no invoices reaching the ASP for one or more customers”); month-end close and the customer’s own customers waiting for invoices add business urgency. The P1 was justified until I could prove delivery, not assumed for the whole ticket.
- **Re-prioritisation:** I kept this at P1 until 10:05. The first import then showed all 8 invoices `Accepted` with ASP references, so the customer’s immediate problem was resolved and the P1 clock stopped at 10:05. The remaining performance/idempotency defect is tracked as an L3 code fix at P2, not as a live P1.
- **SLA deadlines I am working to:** first response 09:20–09:35 (sent 09:12); next update 10:15 (sent 10:05); P1 resolution target 13:05, met at 10:05 when the customer had the accepted list and a workaround. Update cadence was every 30–60 minutes while the P1 was open.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 09:06 | Opened the portal batches page and the Oasis imports; read the whole email thread. | The November file is 600 rows / 8 invoices. There is a first Oasis import that eventually completed, and later attempts whose invoices show duplicate-number errors. This matches the customer’s description of repeated uploads. |
| 09:10 | Reproduced with the exact attached file through the portal proxy (`POST /api/imports`). Timed it and watched the API log. | The browser got **504 Gateway Timeout after 60.0 s**, but the API continued working and completed the import **121 s** after it started. The file itself parsed in about 0.1 s. I ruled out a corrupt workbook or Excel parsing problem. |
| 09:20 | Compared invoice statuses in the first import and in a retry. | First attempt: **8 Accepted**, all with ASP message ids. Retry: **8 Rejected by portal**, all AE-BR-09 “Document number … already exists for this seller with status ‘Accepted’.” I ruled out duplicate invoices actually being sent: the retry was stopped before the ASP. |
| 09:30 | Checked the validation path and the `invoicely_participant_lookup_duration_seconds_count` metric. | Validation performs one Peppol participant lookup **per line**, not per invoice or per buyer. The ASP lookup has a 200 ms simulated round trip. 600 rows × 0.2 s = **120 s**. The metric went 0 → 600 lookups over the same 120 s window. I ruled out a slow database or a slow ASP submission path. |
| 09:40 | Checked the web proxy timeout against the API processing time. | `web/app/api/[...path]/route.ts` aborts upstream requests at **60 s**. The API needs **121 s** for this file. That is the 504 the customer saw; the import was still running in the background. I ruled out an API crash at this stage: the container had no restarts during this upload. |
| 09:50 | Checked the duplicate rule in the store. | The first import inserts all invoices before validation finishes. A retry therefore sees the first import’s invoices (eventually `Accepted`) as existing and rejects every number with AE-BR-09. The retry does **not** send duplicates to the ASP. |
| 10:05 | Sent the customer update with the accepted invoice list and the “do not re-upload” instruction. | All 8 invoices from the first upload were accepted; no duplicate invoices were issued. The customer’s immediate problem (what went out, what to do) was resolved. |
| 10:30 | Raised the underlying defect with L3: synchronous upload plus per-line directory lookups, and retries that create confusing duplicate batches. | Engineering task raised; customer workaround agreed for future files. |

### Root cause

- **What happened:** The 600-line Oasis file takes about **121 seconds** to validate because the portal performs a 200 ms Peppol participant lookup for **every row**. The browser/proxy gives up after **60 seconds** and shows a 504/“something went wrong” page, while the API keeps processing. The customer retried; each retry created a new import, and every invoice in the retry was then rejected as a duplicate because the first import’s invoices already existed and were `Accepted`.
- **Where** (component, `file:line`):
  - `services/api/internal/validate/validate.go:69-71` calls `validateRow` for every row.
  - `services/api/internal/validate/validate.go:165-173` performs `v.dir.IsRegistered(...)` inside `validateRow`, so the same buyer is looked up once per line.
  - `services/api/internal/importer/importer.go:100-114` validates invoice groups sequentially in the upload request.
  - `web/app/api/[...path]/route.ts:6` sets a 60-second upstream timeout.
  - `services/api/internal/store/store.go:298-307` is the duplicate-number check that rejects the retries.
- **Why it happens:** The directory lookup was placed at line level rather than invoice/buyer level, and the import is processed synchronously inside the HTTP request. Nothing deduplicates lookups or moves the long-running work off the request path. The 60-second proxy timeout is shorter than the worst-case validation time. Retries are not idempotent: they create a new import instead of recognising the same file/batch.
- **Evidence:**
  - API log: 600 rows parsed in 0.1 s, import completed at **121,007 ms**; retry completed at **121,013 ms** with 8 rejected — [`evidence/logs/incident1-api-import.log`](evidence/logs/incident1-api-import.log).
  - Proxy log: **504 after 60,002 ms** — [`evidence/logs/incident1-web-proxy.log`](evidence/logs/incident1-web-proxy.log).
  - Database: first import 8 Accepted with ASP ids; retry 8 Rejected AE-BR-09 — [`evidence/queries/incident1-imports.txt`](evidence/queries/incident1-imports.txt).
  - Prometheus: 0 → 600 participant lookups over 120 s — [`evidence/queries/incident1-participant-lookups.txt`](evidence/queries/incident1-participant-lookups.txt).
  - Portal screenshots: [`incident1-oasis-first.png`](evidence/screenshots/incident1-oasis-first.png), [`incident1-oasis-retry-duplicates.png`](evidence/screenshots/incident1-oasis-retry-duplicates.png), [`incident1-oasis-duplicate-error.png`](evidence/screenshots/incident1-oasis-duplicate-error.png).
  - Code excerpts: [`evidence/code/references.md`](evidence/code/references.md).
- **Confidence and open questions:** **High** for the timeout and duplicate mechanism: I reproduced both with the exact attached file and matching timings. Open questions: the customer says they tried three times; I reproduced the first upload plus one retry, and the third attempt is consistent with the same behaviour but I do not have their browser session. I also want engineering to confirm whether any browser/network path can return 502 rather than 504 after the API restarts (Incident 2); the two incidents overlap in time and could have made the third attempt look different.

### Customer resolution reply OR escalation note

- **Which one and why:** **Customer resolution reply**, because the customer’s immediate questions are answerable and no data repair is needed. The underlying performance defect is escalated separately to L3 (see the escalation content at the end of this section).

> Hello Rashid,
>
> I have checked the batch and the submission records. **All 8 invoices in your November file were received and accepted by the ASP on the first upload**: OBS-2611-001 through OBS-2611-008. I have attached the list with the ASP references.
>
> The later uploads did **not** create duplicate invoices. The portal stopped them with “document number already exists” before anything was sent to the network, so your customers will not receive duplicates. You do **not** need to upload the file again.
>
> I am sorry the screen made this look worse than it was. Your 600-line file takes about two minutes to validate, while the portal page gives up after one minute. The processing continued in the background after the page timed out. That is our defect to fix, not something you did wrong.
>
> For next month, until the fix is live, please split the consolidated file into chunks of about **150 lines or fewer** (roughly two invoices at a time). Each chunk should finish in under a minute. I will update you by **14:00** today, and I will confirm when the faster upload is available.
>
> Kind regards,
> Venky — Product Support, Invoicely Lite

**Escalation to L3 (summary):** P1 performance/duplicate-idempotency defect. 600-line Oasis file takes 121 s because of one participant lookup per row; 60 s proxy timeout; retries create AE-BR-09 batches. Affected import `a034be0b-8023-416e-9ca3-f634e50c37a4` (first, 8 Accepted) and retry `1937d449-6ded-4554-9858-2068e392e2ed` (8 Rejected). Ask: deduplicate participant lookups per buyer/invoice, move parsing/validation off the request path (async job + polling), make retries idempotent (file hash / upload key), and add a regression test with the 600-line Oasis file.

### Workaround offered

- Do not re-upload the current file: the first upload already sent all 8 invoices and no duplicates were created.
- For future monthly files, split the workbook into chunks of about 150 lines (roughly two invoices) so each upload completes inside the 60-second portal timeout.
- If the customer needs confirmation, give them the 8 invoice numbers and ASP references from the first import.

### Prevention ideas

- Look up each distinct buyer Peppol id once per invoice (or cache it for the import), not once per line.
- Make imports asynchronous: accept the file, return an import id immediately, and let the portal poll the batch status. This removes the browser/proxy timeout from the critical path.
- Make uploads idempotent (hash of seller + file, or an explicit upload key) so a retry returns the existing import instead of creating duplicate-number errors.
- Keep a clear, honest timeout message: “Your file is still processing; do not upload it again — open the batch to see progress.”
- Add a performance test with the 600-line / 8-invoice Oasis file and a p95 target well under the proxy timeout.

---

## Incident 2 — `inbox/02-alert-api-restarted-marina-bay.md`

### Acknowledgement

- **Sent at (exercise clock):** 09:53, Monday 16 November 2026
- **Sent to:** Priya Nair (Customer Success), cc `support-oncall@`; engineering on-call paged at 09:55. The Grafana alert itself is automated; this is the human acknowledgement of the escalation.
- **Reply:**

> Priya, I have picked this up as **P1** and I am owning it.
>
> I understand: the API has restarted three times in 15 minutes, customers are seeing intermittent 502s (mostly on upload), some invoices have been sitting in “Validated” since about 09:30, and Marina Bay Hospitality goes live on Wednesday.
>
> I am pulling the container restart/memory evidence, the API logs and the database state now. Engineering on-call is being paged in parallel. Please tell the affected customers we are investigating a platform issue and will update them today; ask them not to keep retrying uploads until we confirm the API is stable.
>
> I will send you a factual update by **10:30**, and a clear yes/no on Wednesday as soon as engineering confirms the mitigation.
>
> Venky — Product Support

### Priority and justification

- **Priority:** **P1**
- **Why:** This is platform-wide, not one customer: the API is restarting, uploads are failing with 502/EOF, and invoices are stuck in `Validated`/`Submitted`. Multiple customers are affected, and there is a go-live on Wednesday. It also directly affects Incident 1’s customer. The SLA matrix’s first question — can customers get invoices to the ASP at all? — is answered “not reliably, for several customers”.
- **SLA deadlines I am working to:** first response 09:57–10:12 (sent 09:53); next update 10:30; resolution target 13:42 (4 hours). Update cadence every 30–60 minutes.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 09:43 | Read the Grafana alert and checked the dashboard; ran `changes(process_start_time_seconds{job="api"}[15m])`. | The alert reports **3 restarts** in 15 minutes and the dashboard confirms 3 process starts. The alert is real; this is not a stale/no-data alert. I need to reproduce the failure to capture the memory/OOM evidence behind it. |
| 09:47 | Ran `docker ps`, `docker inspect` and `docker stats`; checked the memory history in Prometheus. | `RestartCount=3`; memory climbs from about 68 MiB to **124–127 MiB of a 128 MiB limit** in the minutes before each restart; Docker recorded an **OOM event** and exit code **137**. I ruled out a manual restart/deploy: there was no deploy, and every restart coincides with the peak load. |
| 09:52 | Checked API logs and the database/ASP health. | No panic or database error before the stops; the API simply stops and restarts. Postgres and ASP were healthy. I ruled out a database outage and an ASP outage as the cause of the 502s. |
| 10:00 | Ran `make load` after a clean start (the documented morning traffic replay) and captured the load generator output. | 240 peak uploads from four customers, 8 concurrent: **34 uploads failed** with `EOF`, connection refused or i/o timeout; **20 imports left in `Processing`**, 1 in `Uploaded`; **63 invoices left `Validated`** and **65 left `Parsed`**. The run reproduced the same OOM pattern with **3 restarts** and `RestartCount=3`. Priya’s report of 502s and stuck `Validated` invoices is exactly this. |
| 10:10 | Inspected the importer, source cache and submitter code. | `SourceCache` is an unbounded `map[string]*Source` that stores the raw file, the parsed workbook and the invoice payloads and **never evicts** anything. Every upload adds to it. The submission queue is an in-memory Go channel; a restart loses queued ids. There is no startup recovery/outbox. |
| 10:20 | Compared the memory limit and the peak volume. | `docker-compose.yml` gives the API only **128 MiB**. 240 uploads × roughly 110 KB raw plus parsed Excel objects and invoice payloads is enough to cross that limit. The container is killed by the cgroup OOM killer, not by a Go panic. |
| 10:30 | Escalated to L3 with the evidence; sent Priya the update. | See escalation below. Temporary mitigation only: raise the API memory limit and reduce upload concurrency under change control. Permanent fix: bound/evict the cache and persist the queue. |

### Root cause

- **What happened:** A morning peak of uploads fills the API’s unbounded in-memory source cache. The container reaches its **128 MiB** memory limit, the kernel OOM-kills it (exit 137), and Docker restarts it. In-flight uploads get 502/EOF. Because the submission queue is in memory, any invoices queued but not yet sent are lost and remain `Validated`/`Parsed`. Invoices already marked `Submitted` are not retried; some may have reached the ASP and some may not, and the database cannot tell the difference, so they need reconciliation.
- **Where** (component, `file:line`):
  - `services/api/internal/importer/sources.go:20-42` — `SourceCache` map, `Put`/`Get` only, no delete/TTL/size cap.
  - `services/api/internal/importer/importer.go:95` — every import stores `Raw`, `Workbook` and `Invoices` in that cache.
  - `services/api/main.go:56-65` — source cache and in-memory submitter are created at startup; nothing is persisted or reloaded.
  - `services/api/internal/submit/submit.go:56-74` — queue is a Go channel; queued work is lost on restart.
  - `docker-compose.yml:43-62` — `mem_limit: 128m` and `memswap_limit: 128m` for the API.
  - `services/api/internal/submit/submit.go:111-155` — status is set to `Submitted` before the HTTP call, so a kill at that point leaves an unretryable `Submitted` row.
- **Why it happens:** The cache was added so the batch page can show source rows/download the original file, but it has no lifecycle. The API is sized for an idle-ish load, not the documented morning peak. The 240 peak uploads average roughly 110 KB of raw workbook each (~26 MB), and every upload also keeps a parsed workbook and invoice payloads in the same map; against a 68 MiB baseline and a 128 MiB limit, that exhausts the container. The queue is not durable, so a restart turns a transient infrastructure event into permanently stuck invoices. There is no reconciliation between “Submitted in our DB” and “documents the ASP actually has”.
- **Evidence:**
  - Restart count, OOM event and exit 137 — [`evidence/logs/incident2-docker-events.txt`](evidence/logs/incident2-docker-events.txt).
  - Memory climbing to the 128 MiB limit before each restart — [`evidence/queries/incident2-memory-samples.txt`](evidence/queries/incident2-memory-samples.txt) and [`evidence/queries/incident2-prometheus.txt`](evidence/queries/incident2-prometheus.txt).
  - Three extra API startup sequences — [`evidence/logs/incident2-api-startup.log`](evidence/logs/incident2-api-startup.log).
  - Load generator: 34 failed uploads, 20 Processing imports, 63 Validated / 65 Parsed invoices — [`evidence/logs/incident2-loadgen.log`](evidence/logs/incident2-loadgen.log).
  - Post-peak database state — [`evidence/queries/incident2-stuck.txt`](evidence/queries/incident2-stuck.txt).
  - Grafana: process restarts = 3, container restarts = 3, memory panel, import processing time, participant lookups — [`evidence/screenshots/incident2-grafana-overview.png`](evidence/screenshots/incident2-grafana-overview.png).
  - Code excerpts — [`evidence/code/references.md`](evidence/code/references.md).
- **Confidence and open questions:** **High** that OOM is the trigger and the unbounded cache is the main cause: the memory curve, OOM event, exit 137 and restart count all agree, and the peak is the documented customer traffic. **Medium** on the exact count of invoices that reached the ASP but are still marked `Submitted`; the database cannot tell us which `Submitted` rows the ASP actually accepted. That reconciliation is part of the L3 ask. Open questions: whether the 128 MiB limit is a deliberate production limit or a lab default, and whether the source preview can be moved to disk/object storage without changing the product.

### Customer resolution reply OR escalation note

- **Which one and why:** **Escalation note** — this is a product/infrastructure defect that support cannot safely fix or work around permanently, and it is a P1. I also sent the customer-facing update to Priya below.

```markdown
### Escalation: API OOM restarts under morning peak; in-memory queue leaves invoices stuck

**Priority:** P1 — production instability affecting several customers; 502s on upload; invoices stuck in Validated/Submitted; Marina Bay Hospitality go-live Wednesday 18 Nov.
**Customer / seller:** multiple sellers; examples Desert Rose (1), Marina Bay (3), Palm Coast (7), Zayed Port (8); Oasis (2) also affected earlier in the same peak.
**Reported by / at:** Grafana alert 09:42; Priya Nair 09:50; reproduced 10:01–10:05.
**Affected:** `make load` run HA1V: 240 peak uploads, 34 failed; 20 imports left Processing, 1 Uploaded; 63 invoices Validated, 65 Parsed, 698 Submitted at the time of the query (most `Submitted` rows are normal submissions; the risk is the subset with no ASP reference). Example import ids: `77d45ed2-3684-495f-babd-0180a31097a2`, `d6123baa-0e72-4a87-8a49-a43089376b91`, `33cc1337-990d-44c6-b5b7-6b84d360f042`.

**Symptoms (customer view):**
Intermittent 502 Bad Gateway, mostly on upload; uploads sometimes fail after a long wait; invoices sit in “Validated” and do not move; some show “Submitted” with no ASP reference.

**Evidence:**
- Payload / database: 128 MiB API limit in `docker-compose.yml:60-61`; SourceCache unbounded in `services/api/internal/importer/sources.go:20-42`; queue in memory in `services/api/internal/submit/submit.go:56-74`.
- Docker events: `container oom` at 06:05:14 UTC; `container die exit=137`; `RestartCount=3`.
- API logs: startup sequences at 06:03:37, 06:04:13 and 06:05:15 UTC — `evidence/logs/incident2-api-startup.log`.
- Metrics: `changes(process_start_time_seconds{job="api"}[15m]) = 3`; working set reaches 124–127 MiB against the 128 MiB / 134,217,728-byte limit — `evidence/queries/incident2-prometheus.txt`, `evidence/queries/incident2-memory-samples.txt`.
- Load generator: 34 failed uploads (`EOF`, `connection refused`, `i/o timeout`), 20 imports left Processing, 63 Validated / 65 Parsed — `evidence/logs/incident2-loadgen.log`, `evidence/queries/incident2-stuck.txt`.
- Dashboard: `evidence/screenshots/incident2-grafana-overview.png` (process restarts, container restarts, memory, import processing time).

**Steps to reproduce:**
1. Start the stack with the normal 128 MiB API limit.
2. Run `make load` (240 peak uploads, 8 concurrent, plus the four customer files).
3. Watch `docker stats invoicely-lite-api-1` and `docker events`; the API memory climbs to the limit and the container is OOM-killed/restarted.
4. Query `select status, count(*) from invoices group by status;` and `select status, count(*) from imports group by status;`; note Processing/Validated/Parsed/Submitted rows.

**What I have already tried / ruled out:**
- Not a deploy/manual restart: no deployment; restarts coincide with the load.
- Not a database failure: Postgres stayed healthy.
- Not an ASP outage: ASP health and response times were normal; the 502s are from the API disappearing.
- Not a single large customer file: the peak run alone (240 small files) reproduces it; Oasis makes it worse but is not required.
- Raising the memory limit is only a temporary mitigation; the cache still grows without bound.

**Suspected cause (with confidence):**
Unbounded `SourceCache` plus a 128 MiB container limit, with an in-memory submission queue that is lost on restart. Confidence high for OOM; medium for the exact per-upload memory profile and for how many `Submitted` rows the ASP actually received.

**Workaround in place:** No durable workaround. Temporary mitigation under change control: raise the API memory limit (for example 512 MiB), reduce peak upload concurrency, and restart before the Wednesday go-live. Do not retry `Submitted` invoices until they are reconciled with the ASP; retrying `Validated` invoices is safe.

**Timeline:**
- 09:42 — Grafana alert: 3 restarts in 15 min.
- 09:50 — Priya forwards customer reports of 502s and stuck Validated invoices.
- 09:53 — P1 acknowledged; engineering paged.
- 10:01–10:05 — `make load` reproduces 3 OOM restarts, 34 failed uploads, 20 Processing imports, 63 Validated / 65 Parsed.
- 10:30 — L3 escalation raised; Priya updated.

**Ask:** (1) Bound or evict the source cache (LRU/TTL or object storage) and make the submission queue durable/recoverable (outbox or DB-backed queue). (2) Reconcile `Submitted` rows against the ASP and requeue anything not received. (3) Confirm whether a temporary memory increase is acceptable for Wednesday, or whether the go-live should move. (4) Add alerts on API memory >80%, restart count, queue depth and `Submitted` age. Needed before Wednesday 18 Nov; interim answer on the memory increase by 11:30 today.
**Next customer update due:** 10:30 to Priya (sent); then every 30–60 minutes while P1.
```

> **Update to Priya (sent 10:30):**
>
> Priya, update on the P1. I have confirmed the API is being OOM-killed under the morning peak: it reaches its 128 MiB memory limit and restarts (3 restarts), which causes the intermittent 502s and leaves some uploads/invoices stuck. This is our platform issue, not a customer file problem. Engineering has the evidence and is working on the fix; I have also asked for a temporary memory increase while that is built.
>
> **Marina Bay on Wednesday:** I cannot honestly say we are OK as things stand. The platform is stable at normal load but will fail again at a peak until the cache/queue fix or a memory increase is in place. I will give you a firm yes/no by **12:00** after engineering confirms the mitigation. Please hold any “all clear” message to the customer until then.
>
> Customers should not keep retrying uploads while the API is restarting. `Validated` invoices can be retried once the API is stable; please do not retry anything showing `Submitted` until we reconcile it.

### Workaround offered

- **Customers:** wait until the API is stable, then retry failed uploads; `Validated` invoices can be retried from the batch page. Do not retry `Submitted` invoices until engineering has reconciled them with the ASP (a retry would be rejected as a conflict, and a resend could duplicate).
- **Internal:** temporary memory increase under change control and reduced peak concurrency; restart the API before the go-live. This buys headroom, not a fix.
- **Data repair:** engineering to reconcile `Submitted` rows against the ASP and requeue only what was not received; `Parsed` rows need a data fix because the normal retry endpoint only accepts `Validated`/`Failed`.

### Prevention ideas

- Bound the source cache: LRU/TTL, max bytes, or store the raw file/parsed preview outside the process (object storage or Postgres).
- Persist the submission queue (transactional outbox) and reconcile on startup; make submission idempotent.
- Right-size the API memory limit with headroom and add a memory-based circuit breaker/backpressure.
- Alert on memory >80%, container restarts, queue depth, and invoices in `Submitted` without an ASP reference for more than a few minutes.
- Load-test at 2× the documented morning peak before a go-live; include Oasis-sized files.
- Return a clean 503/429 from the API when saturated instead of being OOM-killed.

---

## Incident 3 — `inbox/03-al-noor-fuel-br-co-15.md`

### Acknowledgement

- **Sent at (exercise clock):** 10:20, Monday 16 November 2026
- **Reply:**

> Dear Fatima,
>
> Thank you for the detailed note. I understand that several of your November fuel invoices were rejected with **BR-CO-15** (the invoice total with VAT must equal the total without VAT plus the VAT amount), while other invoices from the same file were accepted, and your auditor needs a clear explanation.
>
> I am comparing the line amounts, VAT and totals in the submitted payloads now. Please **do not change or re-upload the file yet** — I want to establish whether the difference is in the file or in our calculation before you spend time correcting something that may not need correcting.
>
> I will reply by **13:00** with the cause and tell you whether anything is needed on your side.
>
> Kind regards,
> Venky — Product Support, Invoicely Lite

### Priority and justification

- **Priority:** **P2**
- **Why:** This is a subset of invoices rejected (3 of 6) for one customer, not a full production outage. It is not a how-to question: the ASP has rejected valid-looking invoices and an external auditor is involved, so it needs a proper written root cause. The customer can still issue the other three, and the rejected invoices did not reach the buyer.
- **SLA deadlines I am working to:** first response 11:15 (sent 10:20); next update 13:00; resolution target 18:15 (8 hours). Update cadence every 2–4 hours.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 10:20 | Pulled the Al Noor import and listed every invoice with its status. | 6 invoices: **0041, 0043, 0045 Accepted**; **0042, 0044, 0046 Rejected by ASP** with BR-CO-15. The error is 1–2 fils, not a missing field or a bad TRN. |
| 10:25 | Compared the customer’s line amounts with quantity × price. | Every line net amount agrees with quantity × unit price within the documented 0.01 tolerance. The customer’s dispatch figures are not wrong. I ruled out a customer arithmetic error. |
| 10:32 | Compared the payload’s total without VAT, VAT and total with VAT. | For 0042: net 14,307.08 + VAT 715.35 = **15,022.43**, but the payload says **15,022.42**. For 0044: 11,254.84 + 562.75 = **11,817.59**, payload says **11,817.57**. For 0046: 5,343.80 + 267.20 = **5,611.00**, payload says **5,610.99**. |
| 10:38 | Checked why only some invoices fail. | The failing invoices have fuel lines with 3-decimal unit prices where quantity × price lands on a half-fil (for example 3,125 × 2.917 = 9,115.625 → line net 9,115.63). The accepted invoices either multiply exactly or the rounding happens to agree. I ruled out a VAT rate/category or buyer/seller issue: the same seller and buyer combinations appear on accepted invoices. |
| 10:45 | Inspected `computeTotals`. | Net and VAT are accumulated from the **rounded** line values, but the gross total is accumulated as `quantity × unitPrice × (1 + VAT)` — i.e. from the **unrounded** multiplication. When the line net is rounded, the two totals differ by a fil. The ASP’s BR-CO-15 check is correct; our payload is internally inconsistent. |
| 10:50 | Checked the ASP logs. | ASP returned 422 with BR-CO-15 for exactly the three invoices; the other three returned 200. No portal validation error. This is a portal calculation defect, not an ASP problem. |
| 10:55 | Escalated to L3 and confirmed there is no safe customer-side correction. | See escalation below. Asking the customer to change quantities/prices would change tax data; the correct fix is in our totals calculation. |

### Root cause

- **What happened:** For invoices with 3-decimal fuel prices, the portal calculates the invoice total with VAT from the **unrounded** quantity × unit price, while it calculates the total without VAT and the VAT amount from the **rounded** line amounts. The result differs by 1–2 fils, so the ASP rejects the invoice with BR-CO-15. The three invoices with exact/benign rounding were accepted; the three with half-fil line roundings were rejected.
- **Where** (component, `file:line`): `services/api/internal/pint/pint.go:171`:
  ```go
  gross += l.Quantity * l.UnitPrice * (1 + l.VATRate/100)
  ```
  The net and VAT totals use `l.NetAmount` and `l.VATAmount` (lines 166–170), which are already rounded to 2 decimals. The returned `TaxInclusiveAmount`/`PayableAmount` use that unrounded `gross` (lines 180–186).
- **Why it happens:** Two different rounding policies are mixed in one calculation. PINT AE / the ASP expect the document total to equal the sum of the rounded line net amounts plus the rounded VAT amounts. The portal instead recalculates the gross from the raw price and quantity, which is not the same once a line is rounded. Fuel is the first customer in the lab with 3-decimal unit prices, so the defect was latent.
- **Evidence:**
  - Six invoice results and payload totals — [`evidence/queries/incident3-alnoor.txt`](evidence/queries/incident3-alnoor.txt).
  - ASP log with exact got/expected for 0042, 0044 and 0046 — [`evidence/logs/incident3-asp.log`](evidence/logs/incident3-asp.log).
  - Portal screenshots — [`incident3-alnoor-rejected.png`](evidence/screenshots/incident3-alnoor-rejected.png), [`incident3-alnoor-br-co-15-error.png`](evidence/screenshots/incident3-alnoor-br-co-15-error.png).
  - Code excerpt — [`evidence/code/references.md`](evidence/code/references.md).
- **Confidence and open questions:** **High.** The failing set is exactly the set with a rounding difference, and the arithmetic matches the ASP’s got/expected. Open question: the correct rounding order for PINT AE should be confirmed by engineering (sum of rounded lines vs rounded sum of lines); the fix must keep BR-CO-10 and BR-CO-14 passing as well as BR-CO-15.

### Customer resolution reply OR escalation note

- **Which one and why:** **Escalation note + customer explanation.** This is a product calculation defect, so support should not ask the customer to change their tax figures. I escalated to L3 and sent Fatima a written explanation she can give her auditor.

```markdown
### Escalation: BR-CO-15 false rejections on 3-decimal fuel invoices

**Priority:** P2 — 3 of 6 invoices rejected for one customer; auditor involved; no production outage.
**Customer / seller:** Al Noor Fuel & Lubricants LLC, seller id 4.
**Reported by / at:** Fatima Khoury, email 10:15; reproduced 10:20–10:45.
**Affected:** import `dc91bd45-1145-424b-9dfb-17c5bfa0e572`; invoices ANF-2611-0042, ANF-2611-0044, ANF-2611-0046 (Rejected by ASP, BR-CO-15). ANF-2611-0041/0043/0045 accepted.

**Symptoms (customer view):**
Three invoices rejected with “Invoice total amount with VAT (IBT-112) must equal Invoice total amount without VAT (IBT-109) + Invoice total VAT amount (IBT-110)”. Their own line calculations agree with the file.

**Evidence:**
- Payload totals: 0042 net 14,307.08 + VAT 715.35 = 15,022.43, payload 15,022.42; 0044 11,254.84 + 562.75 = 11,817.59, payload 11,817.57; 0046 5,343.80 + 267.20 = 5,611.00, payload 5,610.99.
- ASP response: 422 BR-CO-15 for those three, 200 for the other three — `evidence/logs/incident3-asp.log`.
- Database query: `select invoice_number, status, rejected_by, errors from invoices where import_id='dc91bd45-...'` — `evidence/queries/incident3-alnoor.txt`.
- Code: `services/api/internal/pint/pint.go:171` uses unrounded `quantity * unitPrice * (1 + VAT)` for the gross total while net/VAT use the rounded line values.

**Steps to reproduce:**
1. Upload `samples/al-noor-fuel-nov-2026.xlsx` for seller id 4.
2. Observe 3 Accepted / 3 Rejected.
3. Compare `payload.totals.taxInclusiveAmount` with `taxExclusiveAmount + taxAmount` for the rejected invoices.

**What I have already tried / ruled out:**
- Customer arithmetic: every line net equals quantity × price within 0.01; ruled out.
- VAT category/rate: all S/5; ruled out.
- TRN/Peppol registration: same seller/buyer combinations accepted on other invoices; ruled out.
- ASP rule error: the ASP’s arithmetic is correct; our payload is inconsistent; ruled out an ASP defect.
- No safe customer-side workaround: changing prices/quantities would change tax data.

**Suspected cause (with confidence):**
`services/api/internal/pint/pint.go:171` mixes rounded line net/VAT with an unrounded gross calculation. High confidence.

**Workaround in place:** No safe customer-side workaround. Hold the three rejected invoices; do not ask the customer to change their figures. After the fix, re-upload the same invoice numbers (a rejected invoice does not use up its number).

**Timeline:**
- 10:15 — Customer email received.
- 10:20 — Import and statuses checked.
- 10:32 — Totals compared; 1–2 fil differences found.
- 10:45 — `computeTotals` inspected; cause confirmed.
- 10:55 — L3 escalation raised.

**Ask:** Fix `computeTotals` so the document gross equals the sum of rounded line net + rounded line VAT, add regression tests for 3-decimal fuel prices, and re-run the Al Noor file. Confirm the correct PINT AE rounding order. Needed by 18:15 today; customer update due 13:00.
**Next customer update due:** 13:00 at the time of escalation; after the 11:40 explanation, 16:00 as promised to the customer.
```

> **Customer explanation (sent 11:40):**
>
> Dear Fatima,
>
> Thank you for your patience. I have completed the check, and I can confirm the problem is **on our side, not in your figures**.
>
> Three invoices were accepted (ANF-2611-0041, 0043 and 0045). The other three (0042, 0044 and 0046) were rejected because our portal calculated the total with VAT from an unrounded fuel price, while it calculated the total without VAT and the VAT from the rounded line amounts. On 0042, for example, our own net + VAT is AED 15,022.43 but our total with VAT was AED 15,022.42 — a one-fil difference. The ASP correctly rejected that inconsistency.
>
> **You do not need to change your quantities, prices or line amounts.** Your dispatch figures agree with the file. I have raised the calculation defect with our engineering team and will confirm when the three invoices can be re-issued. Please hold them for now; do not re-upload a corrected file, because no correction is needed on your side.
>
> I will update you by **16:00** today. If it helps your auditor, I can provide a short written statement of the cause and the affected invoice numbers.
>
> Kind regards,
> Venky — Product Support, Invoicely Lite

### Workaround offered

- No customer-side correction: their figures are right.
- Hold the three rejected invoices; do not re-upload them with changed amounts.
- After the fix, re-upload the same three invoice numbers; rejected invoices do not “use up” their numbers.
- If the customer needs them urgently, engineering can re-issue them from the corrected calculation; the customer should not use another channel that changes the tax data without advice.

### Prevention ideas

- Calculate the gross total from the same rounded components as net and VAT: `gross += l.NetAmount + l.VATAmount`, or use the already accumulated `net + vat`.
- Add regression tests for fuel-style 3-decimal prices, including half-fil cases; assert BR-CO-10, BR-CO-14 and BR-CO-15 before submission.
- Add a pre-submission internal consistency check so the portal catches its own bad total and shows a clear internal error instead of sending it to the ASP.
- Alert on ASP rejections grouped by rule; a spike in BR-CO-15 should be visible on the dashboard (it is now, but there is no alert).
- Document the rounding policy in the template field guide.

---

## Incident 4 — `inbox/04-bluepeak-sahara-not-received.md`

### Acknowledgement

- **Sent at (exercise clock):** 11:35, Monday 16 November 2026
- **Reply:**

> Hi Daniel,
>
> Thank you for the clear note and for holding the resend — that was the right call.
>
> I understand: 32 of the 40 Sahara invoices show “Submitted” with no ASP reference, 8 were accepted, and Emirates Palm Hospital has only received a few. Your connector got HTTP 201 because we accepted the file; that does not yet mean each invoice reached the ASP.
>
> I am checking the ASP receipt log and our submission records now. Please **do not resend the batch** until I confirm which invoices are already at the hospital; resending all 40 could duplicate the 8 that were delivered.
>
> I will update you by **14:00** with exactly which invoices are safe and the recovery plan.
>
> Thanks,
> Venky — Product Support, Invoicely Lite

### Priority and justification

- **Priority:** **P2**
- **Why:** This is a major degradation for one customer’s batch: 32 of 40 invoices did not reach the buyer, but 8 did, so it is not “no invoices reaching the ASP” (P1). There is a payment run this week and a real duplicate risk if the connector resends, which raises the urgency within P2. I would raise it to P1 if we found that none of the 40 had arrived or that duplicates had already reached the hospital.
- **SLA deadlines I am working to:** first response 12:30 (sent 11:35); next update 14:00; resolution target 19:30 (8 hours). Update cadence every 2–4 hours.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 11:35 | Opened the Sahara import and checked every invoice. | 40 invoices: **8 Accepted**, **32 Submitted**, all 32 with an empty ASP reference. This matches Bluepeak’s report. |
| 11:40 | Queried the ASP for documents from Sahara’s sender id. | The ASP has only the **8 accepted** invoice numbers; the other 32 were never received. I ruled out loss after the ASP or a buyer-side/hospital issue. |
| 11:45 | Checked the ASP and API logs around 09:59 GST. | The ASP accepted 8 documents, then returned **HTTP 429 “rate limit exceeded”** for 32 submissions, all within about 40 ms, with `Retry-After: 1`. The ASP’s fair-use policy is a burst of 8, then 2 per second. I ruled out a 5xx/ASP outage. |
| 11:50 | Inspected the submitter code. | The submitter sets status to `Submitted` **before** the HTTP call. The response switch handles 200, 422 and a few 5xx codes, but has **no case for 429** and no default. The 429 is silently ignored, so the invoice stays `Submitted` forever. There is no retry/backoff. |
| 11:55 | Checked the retry endpoint and the database. | `POST /v1/imports/{id}/retry` requeues only `Validated` or `Failed`; the 32 are `Submitted`, so it returns **`{"requeued":0}`**. The portal’s “Retry stuck invoices” button cannot recover them without a data fix. |
| 12:00 | Checked duplicate risk. | The 8 accepted invoices have ASP message ids; the 32 missing ones do not exist at the ASP. A full connector resend would duplicate the 8. A targeted requeue of the 32 is safe **after** their status is reset. |
| 12:15 | Escalated to L3 for the code fix and data repair. | See escalation below. |
| 12:20 | Sent the customer update with the confirmed split and the “do not resend” instruction. | 8 delivered, 32 missing; a targeted requeue is safe, a full resend is not. |

### Root cause

- **What happened:** The ERP connector uploaded 40 invoices. The portal validated all 40 and queued them. Four workers submitted them quickly; the ASP allows a burst of 8 then 2 per second, so it accepted the first 8 and returned **429 Too Many Requests** with `Retry-After` for the other 32. The submitter had already marked each as `Submitted`, and it does not handle 429, retry or back off. The 32 therefore remain `Submitted` with no ASP reference and never reach the hospital. The normal retry API cannot requeue them because it only accepts `Validated`/`Failed`.
- **Where** (component, `file:line`):
  - `services/api/internal/submit/submit.go:111-117` — sets status `Submitted` before the HTTP call.
  - `services/api/internal/submit/submit.go:133-155` — response switch has cases for 200, 422 and selected 5xx codes; **no 429 and no default**.
  - `services/asp-mock/main.go:32-41` — fair-use policy: burst 8, then 2/s.
  - `services/asp-mock/main.go:274-283` — returns 429 with `Retry-After`.
  - `services/api/internal/store/store.go:367-375` — `Retryable` only `Validated`/`Failed`, so `Submitted` cannot be retried.
- **Why it happens:** The submitter assumes every response is either success, a business rejection or a transient server error it can mark `Failed`. 429 is a normal, expected response from the ASP’s fair-use policy, but it is treated as “no change”. There is no per-sender rate limiting, no Retry-After handling, and no durable queue/outbox. The status transition is also wrong: `Submitted` should mean “handed to the ASP”, but it is set before the request is known to have been accepted.
- **Evidence:**
  - Database: 32 Submitted / 8 Accepted; ASP has only the 8 original numbers — [`evidence/queries/incident4-sahara.txt`](evidence/queries/incident4-sahara.txt).
  - ASP log: 8 accepted then 32 `rate limit exceeded` at 05:59:49 UTC — [`evidence/logs/incident4-asp-429.log`](evidence/logs/incident4-asp-429.log).
  - Retry endpoint returns `{"requeued":0}` — in the same query file.
  - Prometheus at 06:01 UTC: 32 throttled submissions — in the same query file.
  - Portal screenshot: [`incident4-sahara-submitted.png`](evidence/screenshots/incident4-sahara-submitted.png).
  - Code excerpts: [`evidence/code/references.md`](evidence/code/references.md).
- **Confidence and open questions:** **High.** The 8/32 split exactly matches the burst limit, the ASP log has 32 explicit 429s, and the DB shows 32 `Submitted` with blank references. Open questions: whether any of the 32 were accepted by the ASP in a later attempt (the ASP log says no), and whether the connector has its own retry that could have created duplicates outside our logs. Bluepeak says it held the resend, so we believe not.

### Customer resolution reply OR escalation note

- **Which one and why:** **Escalation note + customer update.** The 32 need a data repair and the submitter needs a code fix; L1 cannot safely recover them through the product. The customer also needs a clear “do not resend” instruction.

```markdown
### Escalation: 32 Sahara invoices stuck in Submitted after ASP 429; queue does not handle rate limits

**Priority:** P2 — 32 of 40 invoices not delivered to the buyer; 8 delivered; payment run this week; duplicate risk on resend.
**Customer / seller:** Sahara Medical Supplies FZE (seller id 5), via Bluepeak ERP Integrations.
**Reported by / at:** Daniel Okafor, email 11:30; reproduced 11:35–12:00.
**Affected:** import `d8f736f3-b5fa-44c9-8a48-3cc359bbd67b`; invoices SMS-1116-009 to SMS-1116-040 (32 invoices) are `Submitted` with empty ASP reference. SMS-1116-001 to 008 are `Accepted`.

**Symptoms (customer view):**
The connector got HTTP 201 and the portal shows “Submitted”, but the ASP reference is blank for 32 invoices and the hospital has not received them. The connector logs show no error.

**Evidence:**
- Database: 32 Submitted / 8 Accepted; `asp_message_id` empty for the 32 — `evidence/queries/incident4-sahara.txt`.
- ASP documents: only the 8 original invoice numbers are present at the ASP — same query file.
- ASP log: 32 `rate limit exceeded` warnings at 05:59:49 UTC, after 8 `document accepted` — `evidence/logs/incident4-asp-429.log`.
- Metrics: `asp_requests_total{code="429"} = 32`, `asp_documents_total{outcome="throttled"} = 32` at 06:01 UTC — same query file.
- Code: `services/api/internal/submit/submit.go:111-117` sets Submitted before send; `:133-155` has no 429 case; `services/api/internal/store/store.go:367-375` only retries Validated/Failed.
- Portal: `evidence/screenshots/incident4-sahara-submitted.png`.

**Steps to reproduce:**
1. Upload `samples/sahara-medical-batch-1116.xlsx` for seller id 5.
2. The portal returns 201; after a few seconds the batch shows 8 Accepted, 32 Submitted.
3. `GET http://localhost:8090/v1/documents?sender=0235:1006453719` shows only 8 documents.
4. `POST /v1/imports/{id}/retry` returns `{"requeued":0}`.

**What I have already tried / ruled out:**
- ASP outage/5xx: ASP healthy; the response is specifically 429.
- Buyer/Peppol problem: the 8 accepted invoices use the same sender and mostly the same buyer; ruled out.
- Portal display bug: the database confirms the 32 are `Submitted` with no reference.
- Connector bug: HTTP 201 only acknowledges the import; the loss is inside our submitter; ruled out as the primary cause.
- Normal retry: cannot requeue `Submitted`; confirmed `{"requeued":0}`.

**Suspected cause (with confidence):**
`submit.go` ignores HTTP 429 from the ASP and leaves the invoice in the pre-send `Submitted` state. High confidence.

**Workaround in place:** Customer is holding the resend. No safe L1 data repair. Engineering should reset the 32 `Submitted` invoices to `Validated` (or requeue through an outbox) and resubmit with per-sender rate limiting and `Retry-After`, then confirm all 40 are present at the ASP. The 8 already accepted must not be resent.

**Timeline:**
- 11:30 — Bluepeak email received.
- 11:35 — Import checked: 8 Accepted / 32 Submitted.
- 11:40 — ASP checked: only 8 documents.
- 11:45 — ASP 429s found in the log.
- 11:50 — Submitter code inspected; no 429 handling.
- 12:00 — Retry endpoint confirmed 0 requeued.
- 12:15 — L3 escalation raised.

**Ask:** (1) Add 429/Retry-After handling with backoff and a per-sender client-side rate limiter. (2) Persist the submission queue/reconcile on startup. (3) Data repair: requeue the 32 invoices only, not the 8 accepted, and confirm 40 documents at the ASP. (4) Add an alert for invoices in `Submitted` with no ASP reference for >5 minutes. Needed today; customer update due 14:00.
**Next customer update due:** 14:00.
```

> **Customer update (sent 12:20):**
>
> Hi Daniel,
>
> I have confirmed the split: **8 invoices reached the ASP** (SMS-1116-001 to 008) and **32 did not** (SMS-1116-009 to 040). The 32 were stopped by the ASP’s fair-use rate limit (HTTP 429) and our portal did not retry them; that is our defect, not a connector error. Your HTTP 201 was correct — it acknowledged the file, not each invoice.
>
> **Please do not resend the batch from the connector.** A full resend would duplicate the 8 invoices the hospital already has. I have asked engineering to requeue only the 32 missing invoices and to confirm all 40 at the ASP. I will update you by **14:00** with the requeue status.
>
> Thanks for holding off — that saved us a duplicate problem at the hospital.
>
> Venky — Product Support, Invoicely Lite

### Workaround offered

- Do not resend the whole batch: 8 are already delivered and would duplicate.
- Engineering requeues only the 32 missing invoices after resetting their status; the 8 accepted remain untouched.
- Interim integration advice: the connector can throttle to the ASP’s fair-use policy (burst 8, then 2/s) for future large batches, but the portal must handle 429 itself; the customer should not have to work around it.

### Prevention ideas

- Handle 429 explicitly: honour `Retry-After`, back off with jitter, and retry the invoice rather than leaving it `Submitted`.
- Add a per-sender token bucket in the submitter so the portal never exceeds the ASP’s fair-use limit.
- Persist the submission queue/outbox and reconcile on startup; make the submitter idempotent by invoice id/message id.
- Only set `Submitted` after the ASP has accepted the HTTP request, or introduce a clear `Sending`/`Retrying` state.
- Alert on invoices in `Submitted` without an ASP reference for more than 5 minutes; include the count on the dashboard.
- Add a pre-submit check that the batch size is compatible with the sender’s rate limit, or pace large batches automatically.

---

## Incident 5 — `inbox/05-ghaf-cafe-date-error.md`

### Acknowledgement

- **Sent at (exercise clock):** 12:15, Monday 16 November 2026
- **Reply:**

> Hello Hessa,
>
> Thank you for sending the file. I understand every invoice was rejected with a message about the date format, and you used 13/11/2026 as you normally write dates.
>
> I am checking the file now. The portal currently asks for dates as **YYYY-MM-DD** (for example 2026-11-13), or real Excel date cells. I will confirm whether that is the only issue and send you a corrected file or a two-step fix by **13:00** so your invoices can go out today.
>
> Kind regards,
> Venky — Product Support, Invoicely Lite

### Priority and justification

- **Priority:** **P3**
- **Why:** One customer, one data-format issue, with a simple workaround. It is not a product outage and no invoice reached the ASP, so there is no duplicate or compliance risk beyond the delay. The customer is not an accountant and needs clear help, so it still deserves a human, prompt reply.
- **SLA deadlines I am working to:** first response 16:10 (sent 12:15); next update 13:00; resolution target 19 November 12:10 (3 days), but I resolved it the same day at 12:40 with a corrected file. Update cadence daily if it had remained open.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 12:16 | Opened the Ghaf import and every invoice. | **4 invoices, all Rejected by portal**, rule AE-BR-04. The errors name both `Invoice issue date*` and `Payment due date*`, with values such as `13/11/2026` and `13/12/2026`. Nothing was sent to the ASP. |
| 12:19 | Inspected the workbook. | The dates are **text in DD/MM/YYYY**, not Excel date cells. The rest of the file (TRNs, Peppol ids, amounts, VAT) is fine; the only validation errors are the dates. I ruled out an ASP rejection and a totals problem. |
| 12:23 | Checked the template guide and the parser. | The template requires `YYYY-MM-DD` text or real Excel dates. `sheet.ParseDate` accepts ISO text or an Excel serial; it deliberately does not guess DD/MM vs MM/DD because `05/11/2026` is ambiguous between 5 November and 11 May. The portal is working as documented, but the error message is easy to miss for a non-accountant. |
| 12:30 | Created a corrected copy of the customer’s file, changing only the date cells (16 cells across the eight line rows) to `YYYY-MM-DD`, and uploaded it as a verification. | **All 4 invoices Accepted**, with ASP message ids. This proves the date format is the only blocker and gives the customer a same-day fix. |
| 12:40 | Replied with the explanation, the two-step Excel fix and the corrected file. | Customer can either use the attached corrected file or fix the date column themselves. No escalation needed. |

### Root cause

- **What happened:** The customer typed dates as `DD/MM/YYYY` text. The portal requires `YYYY-MM-DD` text or real Excel date cells and rejects ambiguous slash formats with AE-BR-04. All four invoices were stopped at portal validation and never sent to the ASP. The product is working as documented; this is an input-format issue, not a defect.
- **Where** (component, `file:line`):
  - `services/api/internal/sheet/sheet.go:167-186` — `ParseDate` accepts only ISO `YYYY-MM-DD` or an Excel serial.
  - `services/api/internal/validate/validate.go:113-119` — emits AE-BR-04 for issue and due dates.
  - `docs/template-field-guide.md:49-58` — documents the required format and explains why slash dates are rejected.
- **Why it happens:** The template/portal does not visibly pre-format the date columns, and the error message says “must be a date in YYYY-MM-DD format” without showing the customer’s value in a corrected form. A non-accountant typing dates the local way naturally hits this. The portal’s refusal to guess is correct for tax invoices; the usability gap is the lack of an obvious in-product example/help.
- **Evidence:**
  - All four invoices with AE-BR-04 and the exact values — [`evidence/queries/incident5-ghaf.txt`](evidence/queries/incident5-ghaf.txt).
  - The 16 date cells changed and their corrected values — [`evidence/queries/incident5-date-cells.txt`](evidence/queries/incident5-date-cells.txt).
  - Portal batch and invoice screenshots — [`incident5-ghaf-rejected.png`](evidence/screenshots/incident5-ghaf-rejected.png), [`incident5-ghaf-date-error.png`](evidence/screenshots/incident5-ghaf-date-error.png).
  - Corrected file accepted 4/4 — same query file; corrected workbook in [`evidence/ghaf-cafe-nov-2026-corrected.xlsx`](evidence/ghaf-cafe-nov-2026-corrected.xlsx).
  - Code excerpt — [`evidence/code/references.md`](evidence/code/references.md).
- **Confidence and open questions:** **High.** The corrected file was accepted without any other change. Open question: whether the customer is double-clicking/typing into cells that Excel has formatted as text; the two-step fix handles both text and real date cells. No engineering escalation needed.

### Customer resolution reply OR escalation note

- **Which one and why:** **Customer resolution reply** — the product is working as documented and the customer can fix it immediately with the corrected file.
- **Sent at (exercise clock):** 12:40, Monday 16 November 2026

> Hello Hessa,
>
> Thank you for your patience. I have found the issue, and it is a date-format problem in the file — your invoices themselves are fine.
>
> Our portal needs dates in **YYYY-MM-DD** form (for example **2026-11-13**), or as real Excel date cells. Your file has **13/11/2026**, which is a perfectly normal way to write the date, but the portal cannot tell whether 05/11/2026 means 5 November or 11 May, so it refuses to guess on a tax invoice. That is why every invoice was stopped before it reached the tax network.
>
> **The quickest fix:** I have attached a corrected copy of your file with the dates changed to YYYY-MM-DD and nothing else changed. Upload that file for Ghaf Tree Café and the four invoices should go through. I tested it and all four were accepted.
>
> If you prefer to fix it yourself in Excel:
> 1. Select the two date columns.
> 2. **Format Cells → Custom → type `yyyy-mm-dd`** if the cells are real dates.
> 3. If they are text, type the dates again as `2026-11-13` (year first), or use a helper column with `=TEXT(DATE(RIGHT(A2,4),MID(A2,4,2),LEFT(A2,2)),"yyyy-mm-dd")`, then paste the results as values.
>
> I am sorry for the frustration — the error message could be clearer. I have passed that feedback to the product team. Please reply if the corrected file does not go through, and I will check it with you.
>
> Kind regards,
> Venky — Product Support, Invoicely Lite

### Workaround offered

- Use the corrected workbook in `evidence/ghaf-cafe-nov-2026-corrected.xlsx` (only the 16 date cells changed).
- Or format the date columns as `yyyy-mm-dd` / type `YYYY-MM-DD` text; the template guide has the helper formula.
- The original rejected invoices did not use up their numbers, so the corrected upload with the same GTC-118 to GTC-121 numbers is valid.

### Prevention ideas

- Pre-format the date columns in the official template as `yyyy-mm-dd` and add an example row/comment in the header.
- Improve the error message: show the value and the corrected form, e.g. “Got 13/11/2026; use 2026-11-13”.
- Add an in-portal date-format hint next to the upload button and a link to the template guide.
- Consider accepting unambiguous `DD/MM/YYYY` only if the product/legal team agrees it is safe; the current “never guess” rule is defensible for tax invoices, so the safer improvement is guidance, not guessing.

---

## Overall

### Prioritisation

I opened and acknowledged the items in the order they arrived. Incident 1 (09:05) looked like a month-end P1 and was the first response; I started reproducing it immediately. At 09:42 the Grafana alert for Incident 2 arrived, and by 09:50 Priya had confirmed multiple customers with 502s, stuck `Validated` invoices and a Wednesday go-live. That made Incident 2 the platform-level P1 and a likely explanation for Incident 1’s failed retries. I ran the two P1s in parallel rather than strictly serialising them: Incident 1’s customer needed a definitive answer about what had been sent, while Incident 2 needed engineering escalation.

The order in which I first responded and confirmed a root cause was:

1. **Incident 1 (P1)** — email 09:05, acknowledged 09:12; investigated 09:06–09:40; customer resolved 10:05; L3 escalation 10:30. First because it arrived first and looked like a month-end P1; kept at P1 until I could prove all 8 invoices had reached the ASP.
2. **Incident 2 (P1)** — alert 09:42, acknowledged 09:53; reproduced 10:01–10:05; L3 escalation 10:30. Became the platform-level priority because it affected several customers and could explain Incident 1.
3. **Incident 3 (P2)** — email 10:15, acknowledged 10:20; cause confirmed 10:55; customer explanation 11:40.
4. **Incident 4 (P2)** — email 11:30, acknowledged 11:35; cause confirmed 12:00; L3 escalation 12:15; customer update 12:20.
5. **Incident 5 (P3)** — email 12:10, acknowledged 12:15; corrected file verified 12:30; customer resolution 12:40.

I changed my mind twice: I initially treated Incident 1 as the top item, then moved Incident 2 alongside it once the multi-customer impact was clear. I also expected Incident 4 to be a connector problem; the ASP log showed the 429s, which made it a product defect and raised its urgency within P2.

### What I would do differently

- Page engineering on-call earlier for Incident 2: I spent time confirming OOM myself before escalating. For a P1 with multiple customers, the page should go out as soon as the impact is confirmed, with the full root cause to follow.
- Capture the “before” state of the dashboard and container stats before running `make load`; I had to reconstruct the memory curve from the monitor log and Prometheus.
- Separate the Incident 1 and Incident 2 evidence more carefully: both were happening in the same window, and some `Submitted` rows after the peak are from the restart, not from the Sahara 429. I would label every query with the incident it belongs to.
- Reconcile `Submitted` against the ASP as soon as I saw the blank references, before doing any other analysis. “Submitted in our DB” is not evidence that the document left the building.
- Ask the customer for their exact upload timestamps/browser errors earlier; it would have shortened the “three attempts” reconstruction.

### What I learned / what I would need to learn

This lab was a useful refresher on the difference between “the request returned” and “the work completed”. Specifically:

- **Go/HTTP and containers:** how a synchronous handler plus an in-memory queue behaves under a cgroup memory limit; how to read `docker events` OOM/exit 137, `docker stats`, and Prometheus `process_start_time_seconds` together. I would study Go memory profiling (`pprof`), cgroup v2 memory accounting, and graceful shutdown/queue draining next.
- **Peppol/PINT AE:** the 5-corner model, TRNs, participant ids, and why the ASP validates totals independently. I had to read the template guide and `asp-mock` rules carefully. I would study the PINT AE rule set, especially the BR-CO/BR-S rounding rules, and the difference between line-level and document-level rounding.
- **Rate limits and retries:** the difference between `Submitted`, `Accepted` and “the ASP has it”; the importance of `Retry-After`, idempotency keys and a transactional outbox. I would study the outbox pattern and per-sender token-bucket design.
- **Support craft:** writing to a finance person under time pressure without blaming them; separating the immediate customer answer from the engineering escalation; keeping the customer update cadence even while the root cause is still moving.
- **What I would study next:** Go profiling and cgroup limits; Postgres outbox/queue patterns; the PINT AE validation rules; and how the Next.js proxy timeout and upload UI behave on slow connections.

### Time log (real time)

The times below are the actual wall-clock times of this AI-assisted session (Asia/Kolkata, UTC+5:30). The exercise clock used in the replies is separate.

| Date | From–to | What I worked on |
|---|---|---|
| 2026-11-14 | 11:17–11:24 | Read `START_HERE`, `README`, architecture, SLA, lifecycle, escalation and template docs; checked the running stack and seed data. |
| 2026-11-14 | 11:24–11:29 | Incident 1: reproduced the 600-line Oasis upload through the portal proxy (504 at 60 s, API 121 s), then reproduced the duplicate AE-BR-09 retry. |
| 2026-11-14 | 11:29–11:31 | Incident 3 and Incident 4: uploaded the Al Noor and Sahara files; captured BR-CO-15, the 8 Accepted / 32 Submitted split and the ASP 429 log. |
| 2026-11-14 | 11:31–11:36 | Incident 2: ran `make load` (240 peak uploads), captured 3 OOM restarts, the load-generator failures and the stuck statuses. |
| 2026-11-14 | 11:36–11:41 | Incident 5: uploaded the corrected Ghaf file; verified all four invoices accepted. |
| 2026-11-14 | 11:41–11:45 | Built the `evidence/` folder: SQL queries, Prometheus queries, log excerpts, code references and screenshots. |
| 2026-11-14 | 11:45–12:00 | Wrote and reviewed the submission, replies and escalation notes; checked every claim against the captured evidence. |

Total: about 43 minutes of hands-on wall-clock time. The 6–8 hour estimate in `START_HERE` assumes manual investigation; the AI assistant ran the reproductions and drafted the text, and I checked every claim against the running stack.

### AI usage disclosure

I used **OpenCode with the DeepSeek V4.1 Flash Free model** as an AI assistant for this exercise. I used it to:

- read the repository documentation and Go/Next.js source;
- use the running Docker Compose stack, upload the sample files, run `make load`, and query Postgres, Prometheus, Loki and the ASP mock;
- capture the screenshots and log/query excerpts in `evidence/`;
- draft the customer replies, investigation logs and escalation notes in this file.

I verified the output against the running system rather than trusting the model: every root cause above is backed by a reproduced upload, a database query, an ASP/API log line, a Prometheus value or a code reference. The exact timings and counts in the evidence files are from the lab run. I did not send any real emails; all customer replies are drafts as required. Where I could not reproduce something exactly (for example the customer’s third Oasis attempt), I say so in the confidence/open-questions section. I reviewed the personal fields (name, repository link and real time) before submitting.
