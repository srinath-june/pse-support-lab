# Submission

- **Name:** Srinath S
- **Date submitted:** 14 November 2026
- **Repository link:** https://github.com/srinath-june/pse-support-lab (create this repo and push before submitting)
- **Total time spent (real time):** <fill in>

> DRAFT NOTE: root causes below come from reading the code and sample files (code refs are real). Items marked **[VERIFY]** must be confirmed by running `make up` and saving output into `evidence/`. Do not submit them as proven until you have done that.

- **Items I fully investigated, and why:** Incident 2 (P1, production impact, go-live Wed), Incident 1 (month-end, possible duplicates, shares cause with 2), Incident 4 (silent invoice loss, a payment run), Incident 3 (rejections, auditor). Incident 5 is the product working as documented, so I answered it as a P4-style how-to.

**Work order:** 2 → 1 → 4 → 3 → 5 (reasons in Overall).

---

## Incident 1 — Oasis Builders upload stuck

### Acknowledgement
- **Sent at:** 09:15 (received 09:05)
- **Reply:**
> Hi Rashid, thank you for the detail. I understand this is month-end and you don't want duplicates. Please don't upload the file again: the earlier attempts may already have created invoices, and another upload could add more messages like the "already exists" ones. I'm checking exactly which of your November invoices were sent and will email you the list by 11:00. I'll call you if I need anything. – Support

### Priority and justification
- **Priority:** P2 (raise to P1 if Oasis cannot get any invoice to the ASP once the fix/retry is tried).
- **Why:** Subset of invoices affected, but month-end close and duplicate risk. Same underlying slowness also hits other customers (see Incident 2).
- **SLA:** first response 1 h (by 10:05), update every 2 h, resolution 8 h (by 17:05).

### Investigation log
| Time | Checked / did | Found |
|---|---|---|
| 09:20 | Read email, opened `samples/oasis-builders-nov-2026.xlsx` | 600 rows, 8 invoices, ~75 lines each. Normal sample has a few rows. |
| 09:30 | `validate.go` `validateRow` | Per row, calls `v.dir.IsRegistered` for the buyer Peppol id. Rows are processed one after another; no caching. |
| 09:40 | `asp-mock/main.go:40,335` | Every lookup sleeps 200 ms. 600 × 0.2 s ≈ 120 s per upload. |
| 09:45 | `web/app/api/[...path]/route.ts:6` | Portal proxy gives up after 60 s → returns 504; `UploadForm.tsx:8` shows the timeout message. |
| 09:50 | `importer.go:34` | Import uses `context.Background()` with 10-min budget, so it keeps running after the browser gave up. |
| 09:55 | `importer.go` order of steps / `store.go:298 ExistingStatus` | Invoices are inserted (status Parsed) before validation. The duplicate check excludes only its own import, so a retry sees the first attempt's invoices → AE-BR-09. |
| – | Dead end considered: customer actually issued duplicates | No; same file uploaded 3 times. |

### Root cause
- **What happened:** The file is large, so validation takes ~2 minutes. The portal times out at 60 s while the first import carries on. Rashid re-uploaded twice. Each re-upload re-validated and hit AE-BR-09 against the invoices of the earlier, still-running import.
- **Where:** `validate.go` (per-row lookup, no cache), `route.ts:6` (60 s), `importer.go` (synchronous import), `store.go:298`.
- **Why:** Work is done inside the upload request, repeating the same buyer lookup for every line instead of once per distinct buyer.
- **Evidence [VERIFY]:** `docker compose logs api | grep import` (import received/completed with `duration_ms` ~120000 and three import_ids); Grafana "import duration" panel; `make psql`:
  ```sql
  SELECT i.id, i.status, i.created_at, i.invoice_count FROM imports i JOIN sellers s ON s.id=i.seller_id WHERE s.name ILIKE 'Oasis%' ORDER BY created_at;
  SELECT invoice_number, status, import_id FROM invoices WHERE invoice_number LIKE 'OBS-2611-%' ORDER BY invoice_number, created_at;
  ```
- **Confidence:** High on mechanism; confirm the actual durations and which import's invoices were accepted.

### Customer reply + escalation
- **Both.** Reply to the customer; escalate the product defect (slow synchronous import, duplicate check against in-flight import) to L3 with the template in `docs/escalation-policy.md`. Ask: cache participant lookups per upload, run import asynchronously, return the import id immediately, make the duplicate check ignore a concurrent attempt with a clear message.

> Hi Rashid, here's where things stand. Your file contains 600 lines, which our portal takes about two minutes to check — longer than the page waits, so it showed a timeout even though the first upload kept running. Your later uploads then collided with the first one, which is what the "already exists" messages mean: **you did not issue anything twice, and no customer received duplicates**. The list of your 8 November invoices and their status is below [paste from DB]. For any invoice that is Accepted, nothing more is needed. For any that are still Validated or Failed, please do not re-upload; I will retry them from our side / use "Retry stuck invoices" on the first batch. I will update you by 13:00 and we are fixing the slowness with engineering.

### Workaround
Do not re-upload. Use **Retry stuck invoices** on the original batch; or split the file into smaller files (e.g. 2 invoices each).

### Prevention
Lookup cache, async import with progress, idempotency guard, UI message "still processing, don't re-upload", alert on import duration.

---

## Incident 2 — API restarted alert + Priya (Marina Bay go-live)

### Acknowledgement
- **Sent at:** 09:50 (alert 09:42, Priya 09:50)
- **Sent to:** Priya Nair (cc support-oncall)
> Hi Priya, yes — I believe the 502s and the invoices sitting in "Validated" come from the same event: the API restarted three times in 15 minutes. I'm owning this. I'll give you a status at 10:30 and a go/no-go view on Wednesday before your afternoon call. Meanwhile, customers can use "Retry stuck invoices" on a batch if invoices stay Validated for 15+ minutes.

### Priority and justification
- **Priority:** P1
- **Why:** Repeated API restarts; customers cannot reliably get invoices to the ASP; go-live in 2 days.
- **SLA:** first response 15–30 min (done within 8 min), update every 30–60 min, resolution 4 h. Page engineering on-call now; do not wait for full RCA.

### Investigation log
| Time | Checked | Found |
|---|---|---|
| 09:52 | `docker-compose.yml` | api has `mem_limit: 128m`, `memswap_limit: 128m`, `restart: unless-stopped` → silent OOM-kill-and-restart loop is possible. |
| 09:55 | `importer/sources.go` | `SourceCache` stores the raw file, whole workbook and invoices for every import, never evicted. Memory only goes up. |
| 10:00 | `submit.go` `queue chan string` | Submission queue is in memory. A restart loses every queued id → invoices stay **Validated** (matches Priya). |
| 10:05 | `asp-mock` | In-memory docs are cleared on ASP restart (not relevant here). |
| 10:10 | Alert rule `changes(process_start_time_seconds{job="api"}[15m])` | Fires on restart count only; says nothing about why. |
| – | Dead end: Marina Bay data issue | Symptoms are platform-wide (502 for several customers). |

### Root cause (hypothesis, [VERIFY])
- **What happened:** The API process is killed for exceeding its 128 MB limit, restarts, and loses queued submissions and in-flight imports. During restart the portal gets 502.
- **Where:** `docker-compose.yml` api `mem_limit`; `sources.go` unbounded cache; `submit.go` in-memory queue.
- **Evidence to collect:** `docker inspect $(docker compose ps -q api) --format '{{.State.OOMKilled}} {{.RestartCount}}'`; cAdvisor/Grafana memory panel for api trending to the limit with drops; Loki `{service="api"} | json | msg="api starting"` timestamps; `SELECT status,count(*) FROM invoices GROUP BY 1;` and `SELECT id,created_at FROM invoices WHERE status='Validated' AND created_at < now()-interval '15 minutes';`
- **Confidence:** Medium until OOMKilled is confirmed.

### Escalation note
```markdown
### Escalation: API container repeatedly restarting (suspected OOM) — 502s and stuck Validated invoices
**Priority:** P1 — customers cannot upload/submit; Marina Bay goes live Wed 18 Nov
**Customer / seller:** multiple; Marina Bay (go-live)
**Reported by / at:** Grafana alert 09:42; Priya Nair (CS) 09:50
**Affected:** all api users; invoices Validated since ~09:30 (count: [DB query])
**Symptoms:** 502 on upload, invoices not moving past Validated
**Evidence:** alert restarts=3 in 15 min; docker inspect OOMKilled=[ ]; memory panel [screenshot]; Loki "api starting" lines at [times]
**Steps to reproduce:** make load; watch api memory and restarts
**Ruled out:** ASP outage (asp responses normal), data problem in one customer
**Suspected cause (medium):** 128 MB limit + unbounded SourceCache (sources.go) + large imports; queue lost on restart (submit.go)
**Workaround in place:** Retry stuck invoices per batch (support)
**Ask:** raise memory limit / bound or evict SourceCache; persist or re-enqueue Validated invoices on startup; reply by 11:00 with ETA
**Next customer update due:** 10:30
```

### Workaround
Support re-runs "Retry stuck invoices" for affected batches after each restart. Tell Priya: **Wednesday is not safe until the fix is deployed and a clean `make load` runs without restarts.**

### Prevention
Memory/OOM alert, cache eviction, durable queue or startup re-enqueue, load test before go-live.

---

## Incident 3 — Al Noor Fuel BR-CO-15

### Acknowledgement
- **Sent at:** 10:40 (received 10:15)
> Dear Fatima, thank you for the clear write-up. We've started analysing which invoices were rejected and why. Your quantities and prices are consistent in the file, so please don't change anything yet. I'll send a written explanation you can share with your auditor by 14:00.

### Priority and justification
- **Priority:** P2
- **Why:** 3 of 6 invoices rejected, auditor pressure; root cause looks like our defect, so no customer action needed.
- **SLA:** first response 1 h (by 11:15), update 2–4 h, resolution 8 h.

### Investigation log
| Time | Checked | Found |
|---|---|---|
| 10:45 | `asp-mock/main.go:225-230` | BR-CO-15 checks `inclusive == exclusive + VAT` in cents. |
| 10:50 | `pint.go computeTotals` | VAT per line is rounded (`round2(net*rate/100)`, summed), but **inclusive = Σ qty×price×(1+rate)** from unrounded values, not net+VAT. Two different ways of getting the total. |
| 11:00 | Recomputed all 6 invoices from the sample xlsx | 0042, 0044 and 0045 mismatch by one fils; 0041, 0043 and 0046 agree (table below). |

| Invoice | Net | VAT (rounded) | Portal total sent | Net+VAT | Result |
|---|---|---|---|---|---|
| ANF-2611-0041 | 13749.75 | 687.49 | 14437.24 | 14437.24 | ok |
| ANF-2611-0042 | 14307.08 | 715.35 | 15022.42 | 15022.43 | rejected |
| ANF-2611-0043 | 18684.00 | 934.20 | 19618.20 | 19618.20 | ok |
| ANF-2611-0044 | 11254.84 | 562.75 | 11817.57 | 11817.59 | rejected |
| ANF-2611-0045 | 11283.30 | 564.16 | 11847.47 | 11847.46 | rejected |
| ANF-2611-0046 | 5343.80 | 267.19 | 5610.99 | 5610.99 | ok |

### Root cause
- **What happened:** The portal built invoice totals inconsistently, so a rounding difference of 0.01–0.02 AED appears on some invoices (fuel prices with 3 decimals, 5 % VAT).
- **Where:** `services/api/internal/pint/pint.go` `computeTotals` (gross line: `gross += l.Quantity * l.UnitPrice * (1 + l.VATRate/100)`).
- **Why:** The ASP is right; the customer data is right (line net amounts already equal qty × price to 0.01, passing AE-BR-08).
- **Evidence:** table above; run `curl "http://localhost:8090/v1/documents?sender=<peppol>"` and `SELECT invoice_number, total_excl_vat,total_vat,total_incl_vat,asp_response FROM invoices WHERE invoice_number LIKE 'ANF-2611-%';` **[VERIFY]**
- **Confidence:** High.

### Reply + escalation
- **Both:** escalate to L3 (defect, support cannot fix); reply to customer.

> Dear Fatima, your figures are not at fault. We found a rounding defect in how our system calculates the invoice total: it computes VAT per line and the total separately, so on some invoices the two differ by 1–2 fils (0.01–0.02 AED). The tax network's rule BR-CO-15 requires them to match exactly, so those invoices were refused: ANF-2611-0042, -0044 and -0045. The other three matched and were accepted. The rejected invoices did not reach your customers and their numbers are not used up. Nothing needs correcting in your file. We have raised it with engineering and I'll update you by 16:00 with a date; you can use this message for your auditor.

```markdown
### Escalation: BR-CO-15 rejections from rounding in computeTotals
**Priority:** P2 — 3 invoices for Al Noor; auditor reviewing November VAT
**Affected:** ANF-2611-0042, -0044, -0045 (import [id])
**Evidence:** ASP 422 BR-CO-15; pint.go computeTotals; table of net/VAT/total
**Suspected cause (high):** total is sum(qty*price*(1+rate)) rather than net + sum(rounded line VAT)
**Workaround:** none safe; do not alter customer data
**Ask:** compute inclusive = exclusive + VAT (and VAT per category subtotal); add regression test; fix by [date]; then support retries the invoices
```

### Workaround
None that is safe on the customer's side; after the fix, re-upload corrected invoices with the same numbers (rejected invoices don't use the number — `status-lifecycle.md`).

### Prevention
Unit tests for totals with 3-decimal prices; portal-side BR-CO-15 pre-check; reconcile with ASP rules in CI.

---

## Incident 4 — Bluepeak / Sahara Medical invoices "Submitted", not received

### Acknowledgement
- **Sent at:** 11:45 (received 11:30)
> Hi Daniel, thanks for the detail. Please hold off resending: that is the right call and it avoids duplicates. We're checking where each of the 40 invoices is and I'll send you a per-invoice list by 13:30.

### Priority and justification
- **Priority:** P1 (candidate) → I chose **P2 at ack, raised to P1** when confirmed that invoices are silently not reaching the ASP and the portal shows "Submitted" with no error.
- **Why:** Invoices are silently lost between portal and network; payment run this week.
- **SLA:** P1 → update every 30–60 min, resolution 4 h.

### Investigation log
| Time | Checked | Found |
|---|---|---|
| 11:50 | `asp-mock/main.go:37,262-282` | Per-sender rate limit 2 docs/s; over the limit → **HTTP 429** + `Retry-After`; doc not received. |
| 11:55 | `submit.go` response `switch` | Handles 200, 422, and 500/502/503/504 only. **No case for 429**, so nothing happens: status stays **Submitted**, no ASP reference, no retry. |
| 12:00 | `main.go` | 4 workers pull from the queue, so a 40-invoice batch bursts above 2/s. |
| 12:05 | `store.go Retryable` | Retry only picks Validated/Failed, so "Retry stuck invoices" will not pick these Submitted ones. |

### Root cause
- **What happened:** The first few invoices went through the burst allowance; the rest got 429 and were left in Submitted with no ASP reference. They were never delivered.
- **Where:** `services/api/internal/submit/submit.go` `submit()` switch (missing `http.StatusTooManyRequests`); `store.go` Retryable.
- **Why:** The ASP contract (architecture.md) says a 429 means "not received, send again later"; the client ignores it.
- **Evidence [VERIFY]:** Loki `{service="asp-mock"} | json | msg="rate limit exceeded"`; `{service="api"} | json | msg="asp response" | status_code=429`; Prometheus `invoicely_asp_responses_total{code="429"}`; SQL `SELECT status, count(*), count(asp_message_id) FROM invoices WHERE import_id='…' GROUP BY 1;` (use the real column name in `db/init/001_schema.sql`); `curl "http://localhost:8090/v1/documents?sender=<sahara peppol id>"` shows only the delivered ones.
- **Confidence:** High on code; confirm counts.

### Reply + escalation
- **Both** (code defect; L3 needed).

> Hi Daniel, to answer your two questions. 1) Nothing was lost in the network: our system sent invoices to the Peppol service provider faster than it allows, the provider told us to slow down and resend later, and our system failed to do that, leaving them showing "Submitted" without a reference. The invoices that have a reference ([n]) are delivered; the other [n] are not. Your connector was correct — the 201 only confirms we received the batch. 2) Please **do not resend** from the connector: that would create duplicate numbers. We will resend the undelivered ones from our side once engineering confirms the fix, and I'll send the list of invoice numbers and an update by [time].

```markdown
### Escalation: 429 from ASP ignored; invoices stuck in Submitted
**Priority:** P1 — silent non-delivery, Sahara payment run this week
**Affected:** Sahara batch (import [id]), [n] of 40 invoices Submitted with empty ASP reference
**Evidence:** asp-mock "rate limit exceeded" logs; api "asp response" status_code=429; submit.go switch has no 429 case
**Suspected cause (high):** submit.go missing 429 handling; Retry skips Submitted
**Ask:** treat 429 as retryable (honour Retry-After, back off, per-sender throttle ≤2/s), move invoice back to Validated/Failed; one-time repair of stuck Submitted rows; confirm today
**Next customer update due:** [time]
```

### Workaround
Support (via engineering/L2) sets the stuck Submitted invoices back to Validated and uses retry slowly, or resubmits in batches of a few invoices per second.

### Prevention
Handle 429/Retry-After, worker-level rate limiter, alert on invoices in Submitted > 5 min, monitor 429s.

---

## Incident 5 — Ghaf Tree Café date error

### Acknowledgement
- **Sent at:** 12:30 (received 12:10)
> Hi Hessa, sorry for the frustration. I can see exactly what happened and it's a quick fix on the file. Your invoices haven't been sent anywhere yet, so nothing is wrong on your account. I'll send the steps in the next 30 minutes.

### Priority and justification
- **Priority:** P4 (how-to; product working as documented) but handled quickly because she is a small business with waiting customers.
- **SLA:** first response 1 business day; I replied in 20 min.

### Investigation (brief)
`sheet.go ParseDate` accepts only `YYYY-MM-DD` text or an Excel date cell. The sample cells are **text** `13/11/2026` (verified with openpyxl: type `str`). Documented in `docs/template-field-guide.md` (Dates): `13/11/2026` is rejected as AE-BR-04 because the portal must not guess day/month. Not a defect → no escalation.

### Customer reply
> Hi Hessa, you're right that 13/11/2026 is 13 November. Our system just insists on one format so it never confuses day and month on a tax invoice: **2026-11-13**. In your file the dates are typed as text, so changing the cell format won't help. Easiest fix: in a spare column use `=TEXT(DATE(RIGHT(A2,4),MID(A2,4,2),LEFT(A2,2)),"yyyy-mm-dd")`, copy it, paste as values over the dates (issue date and due date), then upload again. Your invoices were rejected before sending, so their numbers can be reused. I can check the file if you send the corrected version. We'll also look at making this error message clearer.

### Prevention
Friendlier AE-BR-04 message with an example, template with date cell format pre-set, optional support for DD/MM/YYYY with confirmation.

---

## Overall

### Prioritisation
2 (P1: platform, go-live) → 1 (month-end, same root-cause family, duplicate risk) → 4 (silent loss; I would raise to P1 if confirmed and treat it right after 2, because it is more serious than it first looks) → 3 (customer-visible rejection, defect but no data loss) → 5 (documented behaviour). I changed my mind on Incident 4: I first read it as P2 but a missing 429 handler means invoices are never delivered while the portal looks fine.

### What I would do differently
Run the system and gather screenshots first for every item; I would do the DB/Loki queries before writing customer replies; ask engineering whether Incidents 1, 2 and 4 are related (shared load).

### What I learned / need to learn
Peppol 5-corner model, PINT AE, BR-CO rules, rate limits and Retry-After, Loki/Grafana queries. Next: Go concurrency and queue durability.

### Time log (real time)
| Date | From–to | What I worked on |
|---|---|---|
| | | |

### AI usage disclosure
I used Claude to read the repo and draft this submission. Everything flagged **[VERIFY]** must be checked against the running system; the Al Noor numbers were recomputed from the sample xlsx with a script.
