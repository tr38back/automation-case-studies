# Support Email Routing

A layered n8n system that watches customer success inboxes, detects first-contact support requests, and hands them to the support team as Salesforce tickets. Deterministic rules make most decisions before any AI is involved, and an AI agent evaluates the rest, returning a decision, a written justification, and a confidence score for every email.

## Why It's Hard

Customer success managers received support requests mixed in with vendor mail, newsletters, and ongoing conversations. Routing everything with an AI call would have been expensive and risky, since every webhook hit counts as an n8n execution and every model call costs tokens. A handoff to support on a client who was already mid-conversation with their manager would also have felt cold and automated. The design question was how to spend AI only where it earned its cost.

## Architecture

1. **Filter at the source.** Irrelevant email was stopped before it reached the workflow. Only mail from addresses tied to customer companies was allowed through, which removed large vendors, transactional senders, and similar noise.
2. **Build the sender list from real traffic.** The customer success team and their managers first listed the known vendors and distribution lists. The webhook then ran in listen-only mode for a full weekend, logging the sender, subject, and title of every new sender into an Airtable table. That produced hundreds of emails to review, and each non-customer sender was added to a bypass list. Routine checkpoints after launch caught new senders so no one had to monitor the list by hand.
3. **Leave existing conversations alone.** Keyword and thread checks identified emails that referenced an open case or sat in an existing thread. Those never triggered the workflow, because a client in an active conversation was better served by the manager who already had the relationship.
4. **Fast path for critical issues.** If the subject line contained clear critical language, such as a website being down, the email skipped the AI step entirely. It was routed straight to support as an open ticket with a logged, rule-based reason. This saved tokens and removed any chance of an AI misjudgment on an urgent issue. Roughly 15% of the emails that ended up with support took this path.
5. **AI evaluation.** For everything else, an AI agent read the email body. Its criteria were built from an analysis by a data scientist of 55,000 emails from a single year, showing which language patterns led to support tickets and how often. The agent returned its decision, a justification, and a confidence score. A mid-tier model was chosen deliberately, since the task needed semantic recognition and not top-end reasoning.
6. **Route to support.** When an email was routed, the workflow forwarded it by API to a Salesforce support address, creating a ticket with the email body, the sender, and any attachments. Attachments mattered because clients often sent only a screenshot of their website. The agent handled those well most of the time, but flagged them as low confidence when it was effectively guessing from thin context. The workflow also added a comment in the original thread and notified the customer success manager that the conversation had been routed as a support issue.
7. **Acknowledge and hand off.** Once a decision was made, forwarding and ticket creation took a couple of seconds. Salesforce also sent the client an automatic acknowledgment saying the request was with support and someone would reach out shortly, which kept clients informed while tickets waited in a backlog. After that, the workflow stepped out. Client replies continued in the Salesforce ticket thread, not in the automation.

## Feedback Loop

Every decision was logged to a private database and to a shared sheet that customer success could see, with the decision, justification, and confidence score. Managers monitored the sheet for successes and misroutes. If a routing looked wrong, they flagged the log line in a Slack channel, and a new edge-case catch or routing rule followed. That is how VIP and high-touch clients, who were not in the original scope, came to be excluded from handoff entirely and stayed with their dedicated manager.

## Key Design Decisions

- **Filter before spending.** Executions and tokens were the main cost, so the sender filter, thread checks, and keyword rules ran before any AI step.
- **Deterministic where it can be.** Clear critical issues were routed by rule with a logged reason, which was faster, cheaper, and more predictable than an AI call.
- **Never interrupt an active conversation.** Emails tied to open cases or existing threads were left alone, so clients never got a cold automated handoff.
- **Data-informed criteria.** The agent's criteria came from analysis of 55,000 historical emails, not guesswork.
- **Visible, not silent.** The original scope called for automatically archiving routed emails. Managers wanted to see what had been routed, so the email stayed in their inbox with a comment and a notification instead of vanishing.
- **Explainable decisions.** Every routing carried a justification and confidence score, which made review and feedback practical.
- **Hand off cleanly.** Salesforce owned the ticket thread after creation, so the workflow did not need to handle replies.
- **Keep the client informed.** An automatic acknowledgment told clients their request was with support.

## Tech Stack

n8n for orchestration, routing logic, and logging. A shared inbox platform with webhooks as the trigger. Deterministic filters for sender lists, keywords, and thread checks. An AI agent powered by Claude Sonnet for body evaluation. Salesforce for ticket creation. A private database and a shared sheet for logs and visibility. Airtable for the discovery table. Slack for the feedback loop.
