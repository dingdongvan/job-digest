# Design: Daily Job Match Digest

Status: DRAFT. Derived from `intent.md`. Items marked PROPOSED are my suggestions, not yet decided.

## 1. Goals recap
Read LinkedIn and Indeed alert emails from a dedicated Gmail account, score each job against my criteria, and produce a ranked daily digest at 8:00 AM Pacific on my Mac. Full requirements live in `intent.md`; this doc only covers how.

## 2. Architecture (PROPOSED)
A linear pipeline run once per day, with one small local state store.

1. **Fetch:** read new alert emails from the dedicated Gmail account (read-only scope, sender allowlist, authentication checks).
2. **Parse:** extract title, company, location, snippet and cleaned link from each email into a job record.
3. **Sanitize:** build the outbound payload from the allowlist, strip tracking parameters, scrub PII patterns (see intent.md data handling policy).
4. **Dedup:** drop jobs already seen, before scoring, so I don't pay to score the same job twice.
5. **Score:** one LLM call (or batch) per job against my criteria profile, returning schema-validated JSON (score, short reason).
6. **Rank and render:** sort by score, write the markdown digest.
7. **Deliver:** save the digest locally; optionally email it.
8. **Housekeeping:** delete cached raw emails older than 3 days, write the run log, notify on failure.

Why this order: dedup sits before scoring to save cost, and sanitizing sits right before the LLM so nothing unsanitized can reach it.

## 3. Components and decisions

### 3.1 Parsing alert emails
- Alert emails are templated, so a deterministic HTML parser may extract the fields without any LLM call. That is cheaper, faster, more private and easier to test.
- PROPOSED: deterministic parser first, LLM extraction only as a fallback when a template changes.
- Decision needed: confirm after looking at real sample emails from both sites.

### 3.2 Job identity and dedup (deferred from intent.md)
Requirement: a job that already appeared in a digest must not appear again, even after its source email is deleted (3-day cache), and the record must hold no email content or personal data.
- Identity options: site job ID from the link; hash of the cleaned link; title + company + location as a fallback.
- Cross-site matching (same job on LinkedIn and Indeed): title + company + location normalization.
- Storage options: SQLite file (PROPOSED) vs flat file.
- Retention of seen-job records: _TBD_.

### 3.3 Scoring
- Criteria profile: a short file of target roles, domains, skills and company preferences, with no personal identifiers.
- Output: strict JSON schema (score 0-100, one-line reason, matched criteria), validated before use; invalid output is retried once, then the job is flagged rather than silently dropped.
- Model choice and cost limits: _TBD_.

### 3.4 Scheduling
- launchd LaunchAgent running as my user; missed 8:00 AM runs execute on wake.
- Timezone handled so it stays 8:00 AM Pacific across daylight saving.

### 3.5 Secrets and configuration
- Gmail token and LLM API key in the macOS Keychain.
- Non-secret config (criteria profile, sender allowlist, schedule) in version-controlled files.

### 3.6 Observability and failure handling
- Run log with counts and job IDs only, no content.
- Failure notification (e.g. expired Gmail token, parse failures, LLM errors).
- Idempotent runs: re-running the same day must not duplicate digest entries.

### 3.7 Testing (PROPOSED)
- Unit tests for the parser using saved, anonymized sample emails.
- A test that fails if any outbound LLM payload contains my name, an email address or a phone number.
- Dry-run mode printing exactly what would be sent to the LLM.
- A small hand-labeled set of jobs to measure scoring quality against the success metrics.

## 4. Tech stack
- Language: Python (decided).
- PROPOSED tooling: Python 3.12 or newer; `uv` for the virtual environment, dependencies and lockfile (pinned versions); `pytest` for tests; `ruff` for linting and formatting; SQLite from the standard library; `keyring` for the macOS Keychain; Google's Gmail API client; BeautifulSoup for HTML parsing.
- Python version is pinned in `.python-version` (committed to the repo) so the scheduled job and development use the same interpreter.
- Repo layout and how AI coding tools are used across the SDLC phases: _TBD_.

## 5. Stretch goal design (resume tailoring)
Deferred until the core meets its success metrics. Will follow the opt-in, minimized-data rules in intent.md.

## 6. Open decisions
- Python tooling details (confirm uv, repo layout).
- Parser approach once real sample emails are reviewed.
- Seen-job storage and retention.
- LLM provider, model and spend limit.
- Whether the digest is also emailed, and from where.
