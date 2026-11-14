# Code references

Line numbers are from the submitted repository at the time of investigation.

## Incident 1 — Oasis upload slow and duplicate errors

```go
// services/api/internal/validate/validate.go:67-71,165-173
    67	func (v *Validator) Validate(ctx context.Context, seller Seller, importID string, g sheet.Group) []Error {
    68		var errs []Error
    69		for _, row := range g.Rows {
    70			errs = append(errs, v.validateRow(ctx, row)...)
    71		}
   165		if buyer := r.Get(sheet.ColBuyerPeppolID); buyer != "" {
   166			registered, err := v.dir.IsRegistered(ctx, buyer)
   167			switch {
   168			case err != nil:
   169				v.log.Warn("participant lookup failed", "participant_id", buyer, "row", r.Number, "error", err)
   170			case !registered:
   171				add("AE-BR-11", sheet.ColBuyerPeppolID, fmt.Sprintf("Buyer Peppol participant '%s' is not registered on the Peppol network.", buyer))
   172			}
   173		}

// services/api/internal/importer/importer.go:95,100-114
    95		im.sources.Put(id, &Source{Filename: filename, Raw: data, Workbook: wb, Invoices: invoices})
   100		for i, g := range groups {
   101			errs := im.validator.Validate(ctx, validate.Seller{ID: seller.ID, TRN: seller.TRN}, id, g)
   102			if len(errs) > 0 {
   103				rejected++
   104				if err := im.store.MarkRejected(ctx, records[i].ID, "portal", errs, nil,
   105					"Failed portal validation: "+errs[0].Rule+" "+errs[0].Message); err != nil {
   106					return store.Import{}, err
   107				}
   108				continue
   109			}
   110			if err := im.store.SetStatus(ctx, records[i].ID, store.StatusValidated, "Passed portal validation"); err != nil {
   111				return store.Import{}, err
   112			}
   113			ready = append(ready, records[i].ID)
   114		}

// services/api/internal/store/store.go:296-307
   296	// ExistingStatus returns the status of a non-rejected invoice with the same number
   297	// for the seller, uploaded in another import.
   298	func (s *Store) ExistingStatus(ctx context.Context, sellerID int64, invoiceNumber, excludeImportID string) (string, bool, error) {
   299		var status string
   300		err := s.db.QueryRow(ctx, `SELECT status FROM invoices
   301			WHERE seller_id = $1 AND invoice_number = $2 AND import_id <> $3 AND status <> $4
   302			ORDER BY created_at LIMIT 1`, sellerID, invoiceNumber, excludeImportID, StatusRejected).Scan(&status)
   303		if errors.Is(err, pgx.ErrNoRows) {
   304			return "", false, nil
   305		}
   306		return status, err == nil, err
   307	}
```

The portal proxy gives up after 60 seconds; the API keeps processing after the client disconnects:

```ts
// web/app/api/[...path]/route.ts:6
     6	const UPSTREAM_TIMEOUT_MS = 60_000;
```

## Incident 2 — API OOM restart and lost queue

```go
// services/api/internal/importer/sources.go:20-42 — map is only ever Put/Get, never evicted
    20	// SourceCache keeps uploaded files in memory so the batch page can show the
    21	// source rows and offer the original file for download.
    22	type SourceCache struct {
    23		mu      sync.RWMutex
    24		entries map[string]*Source
    25	}
    26	
    27	func NewSourceCache() *SourceCache {
    28		return &SourceCache{entries: map[string]*Source{}}
    29	}
    30	
    31	func (c *SourceCache) Put(importID string, s *Source) {
    32		s.StoredAt = time.Now()
    33		c.mu.Lock()
    34		c.entries[importID] = s
    35		c.mu.Unlock()
    36	}
    37	
    38	func (c *SourceCache) Get(importID string) (*Source, bool) {
    39		c.mu.RLock()
    40		defer c.mu.RUnlock()
    41		s, ok := c.entries[importID]
    42		return s, ok
    43	}

// services/api/main.go:56-65 — in-memory queue and source cache
    56		submitter := submit.New(st, submit.NewASPClient(aspURL), log, 4)
    57		submitter.Start(ctx)
    58	
    59		sources := importer.NewSourceCache()
    60		validator := validate.New(directory.New(aspURL), st, log)
    61		imp := importer.New(st, validator, submitter, sources, log)
    62	
    63		srv := &http.Server{
    64			Addr:              addr,
    65			Handler:           httpapi.New(st, imp, sources, submitter, log).Handler(),

// services/api/internal/submit/submit.go:56-74 — queue is a Go channel, lost on restart
    56	type Submitter struct {
    57		store   *store.Store
    58		asp     *ASPClient
    59		log     *slog.Logger
    60		queue   chan string
    61		workers int
    62	}
    63	
    64	func New(st *store.Store, asp *ASPClient, log *slog.Logger, workers int) *Submitter {
    65		return &Submitter{store: st, asp: asp, log: log, queue: make(chan string, 100000), workers: workers}
    66	}
    67	
    68	// Enqueue schedules invoices for submission.
    69	func (s *Submitter) Enqueue(ids ...string) {
    70		for _, id := range ids {
    71			metrics.QueueDepth.Inc()
    72			s.queue <- id
    73		}
    74	}
```

```yaml
# docker-compose.yml:43-62 — API memory limit
    43	  api:
    44	    build: ./services/api
    45	    environment:
    46	      DATABASE_URL: postgres://invoicely:invoicely@postgres:5432/invoicely?sslmode=disable
    47	      ASP_URL: http://asp-mock:8090
    48	    ports:
    49	      - "8080:8080"
    50	    depends_on:
    51	      postgres:
    52	        condition: service_healthy
    53	      asp-mock:
    54	        condition: service_healthy
    55	    healthcheck:
    56	      test: ["CMD", "wget", "-qO-", "http://localhost:8080/healthz"]
    57	      interval: 5s
    58	      timeout: 3s
    59	      retries: 20
    60	    mem_limit: 128m
    61	    memswap_limit: 128m
    62	    restart: unless-stopped
```

## Incident 3 — BR-CO-15 rounding

```go
// services/api/internal/pint/pint.go:156-186
   156	// computeTotals builds the VAT breakdown and document totals.
   157	func computeTotals(lines []Line) ([]TaxSubtotal, Totals) {
   158		byCategory := map[string]*TaxSubtotal{}
   159		var net, vat, gross float64
   160		for _, l := range lines {
   161			s, ok := byCategory[l.VATCategory]
   162			if !ok {
   163				s = &TaxSubtotal{Category: l.VATCategory, Rate: l.VATRate}
   164				byCategory[l.VATCategory] = s
   165			}
   166			s.TaxableAmount += l.NetAmount
   167			s.TaxAmount += l.VATAmount
   168	
   169			net += l.NetAmount
   170			vat += l.VATAmount
   171			gross += l.Quantity * l.UnitPrice * (1 + l.VATRate/100)
   172		}
   173		subtotals := make([]TaxSubtotal, 0, len(byCategory))
   174		for _, s := range byCategory {
   175			s.TaxableAmount = round2(s.TaxableAmount)
   176			s.TaxAmount = round2(s.TaxAmount)
   177			subtotals = append(subtotals, *s)
   178		}
   179		sort.Slice(subtotals, func(i, j int) bool { return subtotals[i].Category < subtotals[j].Category })
   180		return subtotals, Totals{
   181			LineExtensionAmount: round2(net),
   182			TaxExclusiveAmount:  round2(net),
   183			TaxAmount:           round2(vat),
   184			TaxInclusiveAmount:  round2(gross),
   185			PayableAmount:       round2(gross),
   186		}
```

The gross total is accumulated as `quantity * unitPrice * (1 + VAT)` (line 171) while the net and VAT totals use the rounded line values. For fuel lines with 3-decimal prices the two disagree by 1-2 fils.

## Incident 4 — HTTP 429 from the ASP is ignored

```go
// services/api/internal/submit/submit.go:105-155
   105	func (s *Submitter) submit(ctx context.Context, id string) {
   106		inv, err := s.store.GetSubmission(ctx, id)
   107		if err != nil {
   108			s.log.Error("load invoice for submission", "invoice_id", id, "error", err)
   109			return
   110		}
   111		if inv.Status != store.StatusValidated && inv.Status != store.StatusFailed {
   112			return
   113		}
   114		if err := s.store.SetStatus(ctx, id, store.StatusSubmitted, "Sent to ASP"); err != nil {
   115			s.log.Error("update status", "invoice_id", id, "error", err)
   116			return
   117		}
   118	
   119		start := time.Now()
   120		resp, err := s.asp.Send(ctx, inv.SellerPeppolID, inv.Payload)
   121		if err != nil {
   122			s.log.Error("asp request failed", "invoice_id", id, "invoice_number", inv.InvoiceNumber, "error", err)
   123			metrics.Submissions.WithLabelValues("failed").Inc()
   124			_ = s.store.SetStatus(ctx, id, store.StatusFailed, "Could not reach ASP: "+err.Error())
   125			return
   126		}
   127		s.log.Info("asp response", "invoice_id", id, "invoice_number", inv.InvoiceNumber,
   128			"status_code", resp.StatusCode, "duration_ms", time.Since(start).Milliseconds())
   129	
   130		var reply aspReply
   131		_ = json.Unmarshal(resp.Body, &reply)
   132	
   133		switch resp.StatusCode {
   134		case http.StatusOK:
   135			metrics.Submissions.WithLabelValues("accepted").Inc()
   136			if err := s.store.MarkAccepted(ctx, id, reply.MessageID, resp.Body); err != nil {
   137				s.log.Error("update status", "invoice_id", id, "error", err)
   138			}
   139		case http.StatusUnprocessableEntity:
   140			metrics.Submissions.WithLabelValues("rejected").Inc()
   141			detail := "Rejected by ASP"
   142			if len(reply.Errors) > 0 {
   143				detail = fmt.Sprintf("Rejected by ASP: %s %s", reply.Errors[0].Rule, reply.Errors[0].Message)
   144			}
   145			errs := make([]map[string]any, 0, len(reply.Errors))
   146			for _, e := range reply.Errors {
   147				errs = append(errs, map[string]any{"rule": e.Rule, "field": "", "message": e.Message, "row": 0})
   148			}
   149			if err := s.store.MarkRejected(ctx, id, "asp", errs, resp.Body, detail); err != nil {
   150				s.log.Error("update status", "invoice_id", id, "error", err)
   151			}
   152		case http.StatusInternalServerError, http.StatusBadGateway, http.StatusServiceUnavailable, http.StatusGatewayTimeout:
   153			metrics.Submissions.WithLabelValues("failed").Inc()
   154			_ = s.store.SetStatus(ctx, id, store.StatusFailed, fmt.Sprintf("ASP returned HTTP %d", resp.StatusCode))
   155		}

// services/api/internal/store/store.go:367-375 — retry only covers Validated/Failed
   367	// Retryable returns invoices of an import that may be submitted again.
   368	func (s *Store) Retryable(ctx context.Context, importID string) ([]string, error) {
   369		rows, err := s.db.Query(ctx, `SELECT id::text FROM invoices WHERE import_id = $1 AND status IN ($2, $3) ORDER BY created_at`,
   370			importID, StatusValidated, StatusFailed)
   371		if err != nil {
   372			return nil, err
   373		}
   374		return pgx.CollectRows(rows, pgx.RowTo[string])
   375	}

// services/asp-mock/main.go:32-41,268-284 — fair-use limit and 429 response
    32	const (
    33		customizationID = "urn:peppol:pint:billing-1@ae-1"
    34	
    35		// Fair-use policy per sender: bursts of up to 8 documents, then 2 per second.
    36		senderBurst      = 8
    37		senderRatePerSec = 2.0
    38	
    39		// Typical SMP lookup round trip.
    40		lookupLatency = 200 * time.Millisecond
    41	)
   268	func (s *server) submit(w http.ResponseWriter, r *http.Request) {
   269		sender := r.Header.Get("X-Sender-Id")
   270		if sender == "" {
   271			writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "missing X-Sender-Id header"})
   272			return
   273		}
   274		if ok, wait := s.limiter.allow(sender, time.Now()); !ok {
   275			retryAfter := int(math.Ceil(wait.Seconds()))
   276			w.Header().Set("Retry-After", strconv.Itoa(retryAfter))
   277			s.log.Warn("rate limit exceeded", "sender", sender, "retry_after_s", retryAfter)
   278			documentsTotal.WithLabelValues("throttled").Inc()
   279			writeJSON(w, http.StatusTooManyRequests, map[string]string{
   280				"error":   "rate_limited",
   281				"message": fmt.Sprintf("Too many documents from sender %s. Retry after %ds.", sender, retryAfter),
   282			})
   283			return
   284		}
```

## Incident 5 — Date format

```go
// services/api/internal/sheet/sheet.go:167-186 — only ISO text or Excel serials are accepted
   167	var isoDate = regexp.MustCompile(`^\d{4}-\d{2}-\d{2}$`)
   168	
   169	// ParseDate accepts ISO dates (YYYY-MM-DD) and Excel date cells (serial numbers)
   170	// and returns the date as YYYY-MM-DD.
   171	func ParseDate(v string) (string, bool) {
   172		if isoDate.MatchString(v) {
   173			if _, err := time.Parse("2006-01-02", v); err == nil {
   174				return v, true
   175			}
   176			return "", false
   177		}
   178		if serial, err := strconv.ParseFloat(v, 64); err == nil && serial > 0 && serial < 2958466 {
   179			t, err := excelize.ExcelDateToTime(serial, false)
   180			if err != nil {
   181				return "", false
   182			}
   183			return t.Format("2006-01-02"), true
   184		}
   185		return "", false
   186	}

// services/api/internal/validate/validate.go:113-119
   113		for _, col := range []string{sheet.ColIssueDate, sheet.ColDueDate} {
   114			if val := r.Get(col); val != "" {
   115				if _, ok := sheet.ParseDate(val); !ok {
   116					add("AE-BR-04", col, fmt.Sprintf("[%s] must be a date in YYYY-MM-DD format (got '%s').", col, val))
   117				}
   118			}
   119		}
```
