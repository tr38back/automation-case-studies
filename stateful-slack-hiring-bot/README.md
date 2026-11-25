# Stateful Slack Hiring Bot

A conversational agent in Slack that interviews hiring managers about an open role, validates every answer in code, generates the hiring documents, and routes the package through approval. It cut job posting turnaround from days to under an hour for 20+ hiring managers.

## Why It Is Hard

A chatbot that collects structured data from people over several sessions has two failure modes. It forgets what it already asked, and it lets the model decide what counts as a valid answer. This system avoids both by keeping state and validation outside the model, so the LLM only handles the conversation.

## Architecture

1. **Trigger and ingestion.** A hiring manager messages the bot in Slack. Each conversation thread is siloed to a single position, so a manager can open a new thread for a different role and each thread keeps its own context until that position is fully routed.
2. **Processing and logic.** After every turn, n8n extracts the answers and validates them deterministically against the database's column types and allowed values. Claude is kept out of the data path, so stored fields stay consistent and trustworthy. Claude sees which fields are complete and which are missing, then asks only for the gaps.
3. **State and storage.** Supabase stores validated answers after every turn. Because state lives outside the model, managers can pause mid-interview and resume later without losing progress, and several requisitions can run in parallel.
4. **Downstream actions.** Document generation, button-driven review and approval, stakeholder notifications, and escalation reminders.

## The Three Interview Phases

The bot moved each manager through three phases and asked only questions belonging to the current phase. A phase had to be complete before the next one opened.

1. **Internal readiness.** Is this a backfill or a new position? Is budget already allocated, or does funding need to be found? Has finance approved it? These answers branched the rest of the flow. A backfill already had a job description, so requisition creation was skipped.
2. **Position details.** Level, department, compensation, and the other information needed to build the job posting and the internal requisition form.
3. **Interview design.** Who will interview the candidate, how many stages, whether the manager has existing questions or wants Claude to generate them, and whether to include a technical project or group interview.

## Approval and Escalation

When all phases were complete, the bot posted Slack buttons so the manager could review anything or send the package for approval.

1. **Send for approval.** Finance was notified individually in Slack. A group channel gave the executive sponsor visibility without requiring action.
2. **Manager confirmation.** The hiring manager was immediately told the request was submitted and that they would hear back.
3. **Escalation.** If an approver stayed silent, the bot nudged them after 24 hours and again after three days.
4. **Three possible outcomes.** Approve, deny with a required reason, or request more information.
5. **Dual-channel delivery.** Every notification went to both Slack and email. Email carried the full report, and Slack carried a conversation siloed to that single position.

## Key Engineering Decisions

- **Deterministic data path.** Extraction and validation ran in n8n against database types and allowed values. Claude never touched those fields, so stored data stayed consistent.
- **Phase-gated prompting.** Prompt sets were loaded by interview phase, keeping context windows small and preventing the model from asking questions too early.
- **Persistence after every turn.** Answers were saved each turn, so managers could pause, return later, and run several requisitions in parallel.
- **Branching on early answers.** Phase one answers decided which documents were generated and how the interview continued.
- **Automated follow-up.** Timed nudges at 24 hours and three days kept approvals from stalling.

## Documents Generated

A job posting, an internal job requisition form, and an interview plan with questions.

## Tech Stack

Slack as the interface with interactive buttons, n8n for orchestration and validation, Claude for conversation and question generation only, Supabase with a typed schema for per-turn state, and Slack plus email for notifications.
