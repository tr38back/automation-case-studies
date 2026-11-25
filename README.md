# Automation Case Studies

Production automation and agentic AI systems I designed and built at a real estate SaaS company, written up as case studies. The original code and data were proprietary, so each folder documents the architecture, the engineering decisions, and the problems solved, without client or employer details.

The two deepest projects lead the list. A runnable, sanitized rebuild of the migration matcher is planned as a second phase in this same repo.

| Project | What it is |
| --- | --- |
| [Semantic URL Migration Pipeline](./semantic-url-migration-pipeline) | Crawl, embed, and intent-lock thousands of URLs to generate a safe redirect map for site launches. |
| [Stateful Slack Hiring Bot](./stateful-slack-hiring-bot) | A schema-validated, phase-gated Claude agent that interviews hiring managers in Slack and routes approvals. |
| [Support Email Routing](./support-email-routing) | Layered deterministic filters plus an AI agent that turns first-contact emails into Salesforce tickets. |
| [NPS Detractor Recovery](./nps-detractor-recovery) | Turns noisy survey webhooks into one assembled response, an escalation case, and a stateful outreach sequence. |
| [Social Content Syndication](./social-content-syndication) | An agent that turns published blog posts into platform-ready social posts and delivers a review report. |

## Common Threads

- n8n as the orchestration layer, with deterministic logic handling what it can and AI handling only what needs judgment.
- State kept outside the model, in Supabase or Redis, so workflows are resumable and traceable.
- Hard rules enforced in code, with uncertain cases flagged for a human instead of guessed.

## Status

- [x] Case studies for all five projects
- [ ] Runnable rebuild of the migration matcher (in progress)
