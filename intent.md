# Intent: Daily Job Match Digest

Status: DRAFT (brainstorming in progress)

## Purpose
Scan open jobs on LinkedIn and Indeed every day and surface the ones that match my target role, domain, skill set and expertise. The project doubles as a hands-on way to learn the AI-assisted software development lifecycle (AI SDLC).

## Target audience
- Me, as the primary user.
- Possibly shareable with others later (so avoid hard-coding personal details where cheap to avoid).

## Output
- A daily ranked digest (email and/or markdown file), best matches first.

## Scheduling and operation
- Runs automatically every day at 8:00 AM Pacific time (a cron-style scheduled job on my local machine).
- Use the America/Los_Angeles timezone so it stays 8:00 AM local across daylight saving changes, rather than a fixed PST offset.
- Needs the machine to be on and awake at that time, or the job must catch up when the machine next wakes (so a missed day still produces a digest).
- Safe to re-run: processing the same emails twice must not create duplicate entries.
- Writes a run log, and tells me when a run fails (for example an expired Gmail token), so silent failures don't go unnoticed.
- Platform: macOS. Scheduled with a launchd LaunchAgent (runs as my user, not root), which runs a missed 8:00 AM job when the Mac wakes. Optionally schedule a wake just before 8:00 AM.

## Security requirements
- **Least-privilege Gmail access:** read-only OAuth scope. Job alerts are forwarded into a separate Gmail account dedicated to this project, so the script's credentials can't reach my main mailbox.
- **Secrets:** Gmail tokens and LLM API keys live in the macOS Keychain (or files with owner-only permissions), never in the repo or logs. The repo has a .gitignore for secrets and a secret scanner (for example gitleaks) as a pre-commit check.
- **Untrusted input:** email content, including job text and recruiter wording, is untrusted. The LLM call has no tools and no ability to take actions. It returns structured output only (validated against a schema), and the script never follows or fetches links from emails.
- **Sender verification:** only process emails from known LinkedIn and Indeed sender addresses that pass Gmail's authentication checks, since alert emails can be spoofed.
- **Privacy:** minimize what is sent to the LLM provider (no resume or personal details unless the stretch goal needs them). Choose a provider whose data-use terms I'm comfortable with.
- **Dependencies:** isolated virtual environment, pinned versions, few packages.
- **Hygiene:** FileVault on, spend limit set on the LLM API key, logs contain no tokens or full email bodies.
- **LLM data policy:** see "Data handling policy for the LLM" below.
- **Known gotcha:** Google OAuth refresh tokens expire after 7 days for apps left in "Testing" status, so the setup must handle that (and the failure notification should catch it).

## Success metrics
Two goals, weighted equally:
1. **Learning:** I can walk through every SDLC phase (requirements, design, build, test, deploy, operate) and say what AI did well and where it needed human judgment.
2. **Job search:** the digest helps me find and apply to good-fit roles faster than manual searching.

Concrete, measurable targets (confirmed):
- Precision: at least 3 of the top 5 jobs each day are ones I'd genuinely consider.
- Time: reviewing the digest replaces manual searching and takes under 10 minutes.
- Coverage: after a 2-week trial, fewer than 1 good job per week found elsewhere that the digest missed.
- Learning: each SDLC phase has a short written retro (what AI did, what I had to correct).

## Scope
**In scope (v1 core)**
- Read LinkedIn and Indeed alert emails once a day, locally.
- Extract title, company, location and link for each job with an LLM.
- Score each job against my criteria and give a short reason for the score.
- Remove duplicates across both sources and across days.
- Produce a ranked markdown digest (optionally also sent as an email).

**Stretch goal (v1, only after the core meets its success metrics)**
- Tailor my resume and draft a cover letter for top-ranked jobs.
- I review, edit and submit everything myself; nothing is sent automatically.

**Out of scope for v1**
- Direct scraping and other job boards.
- Auto-applying.
- Application status tracking.
- Multi-user support or a UI.

## Constraints
- Data source v1: LinkedIn and Indeed saved-search email alerts, read by a local script and parsed/scored by an LLM. No direct scraping, which avoids terms-of-service problems and bot blocking.
- Possible v2: add a job aggregator/API if email alerts leave coverage gaps.
- Email access: all alerts arrive in my Gmail account; the script reads them locally through the Gmail API with read-only permission.
- Prerequisite: my LinkedIn and Indeed saved searches/alerts are not yet well defined. Before building, I need a small set of daily saved searches that reflect the matching criteria below, since the digest can only be as good as the alerts feeding it.
- Known limitation: alert emails contain limited detail (title, company, location, short snippet), so scoring is based on that, not on the full job description. Fetching full descriptions from job links would amount to scraping and is out of scope for v1.
- Runs as a local script on my own machine (no cloud hosting for now).
- Budget, time available: _TBD_

## Data handling policy for the LLM
Principle: the LLM only ever sees what it needs to score a job. This is enforced in code, not by good intentions.

- **Never sent:** my name, email addresses, phone number, home address, the full resume, credentials or tokens, and raw email bodies.
- **Allowed by default (allowlist):** only fields extracted from the alert: job title, company, location, short snippet. Links are included with tracking parameters stripped, since those can embed my account identifiers.
- **Scoring uses a criteria profile, not my resume:** a short list of target roles, domains and skills with no personal identifiers.
- **Stretch goal (resume tailoring):** opt-in per run, top-ranked jobs only. Send only the experience content with contact details removed; re-insert contact details locally after generation.
- **Redaction layer:** a deterministic step (not the LLM) builds the outbound payload from the allowlist and scrubs patterns such as email addresses and phone numbers.
- **Dry-run mode:** a flag that prints exactly what would be sent, without calling the LLM.
- **Automated test:** a test that fails the build if any outbound payload contains my name, an email address or a phone number.
- **Provider settings:** use the provider's API (not a consumer chat app), and review its current data-retention and training terms before the first real run, using zero-retention options if offered.
- **Retention and logs:** logs record counts and job IDs, not content. Cached raw emails are deleted after 3 days.

## Matching criteria (what I'm looking for)
- Role / title: Principal engineering manager, or director of software, managing multiple teams and engineering managers.
- Domain (in rough priority groups):
  - Cybersecurity
  - Cloud services and infrastructure, especially where agentic AI can be leveraged or adapted
  - Consumer applications similar to Life360
  - Frameworks and tooling for fast experimentation and iteration
  - Real-time data aggregation and database management systems (mainly at large companies)
- Location: remote first; hybrid acceptable in the Seattle area.
- Skills and expertise to emphasize:
  - Security compliance: enabling products for FedRAMP, GovCloud and PCI
  - AWS cloud platform: scalable, highly available, operationally excellent systems
  - Leading multiple teams and engineering managers
- Company type preference:
  - First choice: startups and small companies.
  - Also considered: large corporations (e.g. Meta, Microsoft, Google), particularly for real-time data aggregation and database management systems.
- Hard deal-breakers (compensation floor, industries to avoid, etc.): none stated yet.

## Open questions
- _TBD_
