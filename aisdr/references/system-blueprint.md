# AI SDR system blueprint

Include this whole file in the build brief as section 10, "System rules", after the user's answers. Replace each `{…}` value with the user's answer. Where the user gave no answer, keep the default stated here. Leave nothing in curly braces in the finished brief.

---

## 10 — System rules (build these exactly)

These rules apply to everything above. If an instruction here conflicts with a convenience choice during the build, these rules win. Tell me before you change any of them.

The running pipeline is rules-based. It doesn't call an LLM to choose leads or send outreach. Apollo sends the sequence.

### Build boundaries

- Put lead-source access behind one adapter that returns: person details, resources opened with timestamps, page views and bookings. Pipeline code must not read my website's tables directly.
- Put Apollo, Calendar and Slack behind separate adapters, each with a readiness check and a mock for tests.
- Keep AI SDR state in its own tables, separate from my source lead data.
- Never put real leads, API keys or credentials in code, seed data, tests or screenshots. Use clearly fake test data.

### The order of decisions

Follow this order exactly. Moving a step changes who gets contacted and what credits are spent.

1. **Choose the window.** First run: all leads. Later runs: since the last successful run. Allow a manual override.
2. **Collect leads** with at least one qualifying action in the window. Rank by distinct resources opened. For CSV imports, use each new import batch.
3. **Deduplicate.** Skip anyone already in the processed-leads registry, matched by source lead ID and by normalised email.
4. **Validate the email.** Blank or invalid goes to `excluded`.
5. **Static suppression.** Match exact emails and whole domains against my exclusion list, including my current clients and anyone who has unsubscribed. A match goes to `excluded`.
6. **Country suppression.** Apply my editable list: {excluded countries, or none}. A blank country isn't excluded.
7. **Consent check.** Apply the rule in "Consent and permission" below.
8. **Score** using the model below.
9. **Calendar check**, if connected. Match the email against actual event attendees and organisers, from two years back to one year ahead. A match goes to `calendar_match`.
10. **Enrich through Apollo** using People Match with the known name, email, domain and LinkedIn URL. No match or an error goes to `failed_enrichment`. Stop enriching when the per-run credit cap is reached, and leave the remaining leads unprocessed for the next run.
11. **Choose the outreach address.** Prefer a valid business email the lead gave me. For a personal email, use Apollo's work email. If there's no usable business email, or Apollo's status is `catch_all`, `guessed`, `unavailable` or `unverified`, send the lead to `needs_review`.
12. **Auto-push rules**, only if I chose partial or automated. If every rule matches, push now. Rules: {auto-push rules}. If the per-run cap is reached, queue the lead for approval.
13. **Top leads.** A score of 100 or more goes to `top_leads` for review.
14. **Below threshold.** Goes to `pending_approval`, not `excluded`.
15. **ICP mismatch.** Goes to `pending_approval`, not `excluded`.
16. **Apply the mode.** In approval mode, qualifying leads wait in `pending_approval`. In automated mode, they're pushed while the cap allows.
17. **Record it.** Save the decision snapshot, counters, dedup outcome and an activity entry.

### Scoring

Score = (distinct resources × per-resource) + (sum of intent-tag multipliers × intent-tag weight) + recent-open bonus + LinkedIn bonus + company-URL bonus + booking bonus + (page views × per-page-view).

Defaults, all editable in Settings: per-resource 10, intent-tag 8, recent open within 30 days 10, LinkedIn 8, company URL 6, booked 20, per-page-view 5. Intent multipliers: high 1, medium 0.5, low 0.25, unknown 0. Score threshold: 40.

Adjust these defaults to favour the signals I chose above. When leads come from a CSV with no behaviour data, set the behaviour weights to 0 and score on email validity, LinkedIn presence and ICP fit, with a threshold suited to those signals.

Count distinct resources by title. Count total clicks separately, including repeat opens, and use total clicks for any "engaged more than once" rule.

### Buckets

Use these exact names across the database, API and UI: `pending_approval`, `top_leads`, `calendar_match`, `needs_review`, `failed_enrichment`, `excluded`, `approved`, `pushed`, `rejected`.

Each decision record keeps the score and breakdown, resources opened, country, enrichment snapshot, Apollo contact ID, bucket, reason, run ID and timestamps. Rejecting a lead also adds it to the dedup registry so it's never processed again.

### Consent and permission

My answer: {consent answer}.

- Store a consent status and source for each lead: `opted_in`, `requested_resource` or `unknown`.
- Only `opted_in` leads, or leads covered by a lawful basis I've confirmed for this list, can be pushed automatically.
- Every other lead goes to `pending_approval` with the reason `consent_review`, before any Apollo credit is spent on it.
- Honour unsubscribes and suppression signals on every run.
- Don't remove or bypass this check unless I explicitly ask, and show me a warning first.

### Sending controls

- Whatever mode I chose, the first run must use approval mode. Switching to partial or automated is a manual setting that I change after reviewing the first batch.
- Enforce all caps on the server, not only in the UI. Send cap: {send cap, default 100} per run. Enrichment credit cap: {credit cap, default the same as the send cap} per run.
- Manual and scheduled runs use the same pipeline and must never overlap. Use a database lock and store run status and progress in the database.
- Make the weekly or daily schedule timezone-aware and idempotent, so a restart can't trigger a second run for the same slot.
- Mark calendar failures visibly. Default policy: hold affected leads in `needs_review`. Make this a setting.

### Apollo

- The pipeline uses the Apollo REST API with `APOLLO_API_KEY`. MCP is only for interactive work in Replit.
- Show a readiness check that confirms the key can manage contacts, labels, custom fields, mailboxes and sequences.
- To push: find or create the contact, apply the list as a label, write the custom fields, find an active send-from mailbox, then enroll the contact in {sequence name}. A list label and sequence enrollment are separate actions.
- If any part of a push fails, move the lead to `needs_review` with the error and let me retry. Never count a failed enrollment as pushed.

### Dashboard

Status bar: Apollo and Slack status, mode, schedule and timezone, Run now, and live progress while a run is active. Disable Run now during a run.

Tabs:

- **Dashboard:** funnel cards for reviewed, qualified, pushed, top leads, pending approval, calendar matches, needs review, failed enrichment and excluded, plus recent runs and the active criteria.
- **Approval Queue:** pending leads and top leads, with score breakdown, reason filter, and bulk approve or reject.
- **Review Buckets:** calendar matches, needs review, failed enrichment, excluded, pushed and rejected.
- **Optimize:** sent, opened, replied, likely positive and meetings, with conversion by resource and score band, and a manual Sync button.
- **Activity:** timestamped pipeline and operator events.
- **Settings:** every setting above, plus country and static exclusions and reset controls.

If a provider is disconnected, say which function is unavailable. The qualified count must never be lower than the pushed count.

Label a reply "likely positive" only as an engagement estimate, and tell me to read the actual reply. Recommendations are advice only and must never change scoring or targeting by themselves.

### Resets

Build two separately explained actions, each transactional and blocked during a run:

- **Run-state reset:** clears decisions, run logs and activity, and non-pushed dedup rows, then rewinds the run window. It keeps source leads and keeps dedup protection for anyone already pushed.
- **Full reset:** also removes pushed dedup protection. Require a typed confirmation and warn that it can re-enroll people in Apollo and doesn't remove them from Apollo.

Neither reset may ever delete source lead data.

### Security

- Protect `/admin` and all admin API routes. Use the `ADMIN_PASSWORD` secret, `HttpOnly`, `Secure`, `SameSite` session cookies, session expiry and login rate limiting.
- Validate every request body and query parameter on the server.
- If Slack approval links are used, make each token opaque, expiring and single-use, with no personal data in the URL. A repeated click must not change the decision.
- Log the operator and timestamp for every approval and rejection.

### Before you tell me it's finished

Test against a separate test database with mocked Apollo, Calendar and Slack, and confirm:

1. Each step in the decision order runs in the documented order.
2. Re-running a window can't duplicate a processed or rejected lead.
3. Suppressed and consent-review leads spend no Apollo credits.
4. Uncertain email statuses never enter the sequence.
5. Caps hold on the server, and two runs can't overlap.
6. A failed Apollo push is visible, retryable and not counted as pushed.
7. Neither reset deletes source leads, and a run-state reset can't re-enroll anyone already pushed.
8. Every admin route rejects unauthenticated requests.
9. A reused or expired Slack link changes nothing.
10. The first run happens in approval mode and sends nothing without my approval.

Don't run tests or resets against my production data.
