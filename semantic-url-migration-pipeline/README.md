# Semantic URL Matching for Site Migration Pipeline

An n8n pipeline that maps every URL on a customer's old website to the right page on their new one, producing a redirect file ready for bulk import. It combines crawling, embeddings, similarity search, and an AI agent, all bounded by hard rules that keep the output safe to ship.

## Why It's Hard

When a site is rebuilt, its old URLs have to point somewhere sensible, or the search rankings and traffic they earned are lost. Doing this by hand across thousands of pages is slow and error-prone. Uninformed automation is worse, because a confident wrong match, like sending an informational article to a sales page, can cost rankings and even get pages de-indexed. The same structured matching and intent understanding is the foundation of search visibility in AI answer engines too.
Additionally, competing priorities across teams created friction & race conditions. Launch Team wanted speed, and the Migrations Team wanted accuracy. Those aren't mutually exclusive goals, but team processes were nuanced and undocumented at the start.

## Architecture

The pipeline is a parent n8n workflow with sub-workflows, triggered by Salesforce webhooks. Vectors are held in Redis, and the final output is a CSV. The embedding model was Google's and was deliberately swappable, since the matching logic does not depend on a specific model.

1. **Trigger.** One conditional parent webhook already received every status change from the site build process, designed jointly with the other AI Automation Engineers so a single entry point served all automations. The migration pipeline filtered for the right status, confirmed in Salesforce that the account had purchased a migration service, and only then launched. This avoided wasted executions, which can add up quickly in Enterprise n8n.
2. **Crawl.** Firecrawl crawled the old site through its API. Although I've been an avid Screaming Frog user for years, it was ruled out because automating it would have needed a VPS plus fragile scripting. It just wasn't meant to run in this environment. Destination URLs came straight from a database query of the new site.
   For each origin URL, only the URL, canonical, title, meta description, and H1 were captured, since a full content crawl was unnecessary for finding intent and a close match.
3. **Filter.** URLs with an exact match on the new site needed no redirect and were dropped. Host-level changes, such as non-www to www, were left to existing edge redirect rules. Canonical URLs were prioritized so the signals that mattered most passed to the new site.
4. **Sanitize.** Before embedding, brand fragments were stripped from titles using regular expressions, so branding would not skew similarity. Pattern matching was deliberate: a brand name that is part of a real headline was kept, while boilerplate suffixes were removed.
5. **Embed.** The cleaned canonical URL, title, H1, and meta description were tokenized, embedded, and stored in Redis. Non-canonical URLs were not used for destination matching, since the canonical is the page that carries weight. However, they were still a part of the origin group, which would match to destinations.
6. **Classify intent.** Every page was labeled with a search intent. The standard taxonomy has four: informational, transactional, commercial, and navigational. Commercial and navigational describe how people search, such as looking for a type of business or naming a specific one, so they rarely describe what a site's pages host. In practice, nearly every page was either informational or transactional, and matching was locked to intent.
   An informational page could only match another informational page, and a transactional page, such as a property listings page, could only match another transactional one. The reason is user behavior: someone searching for homes under a given price wants listings, and sending them to an article about local architecture would make them leave.
7. **Match.** Cosine similarity, restricted to the same intent, produced a shortlist of nearest candidates per origin URL. An AI agent then chose the best one, weighing the similarity score alongside other URL factors and intent. If the intent was wrong or no safe match existed, the URL was flagged as an exception for human review.
8. **Export.** The mapping was exported to CSV with a quality control pass before go-live. If URLs changed before launch, the pipeline could rematch.

## Rules Enforced in the Pipeline

- **Intent lock.** Never redirect across intent.
- **One redirect per origin.** A page is redirected once, which prevents chains and splits.
- **Many-to-one allowed.** A destination can receive any number of redirects.
- **Multi-intent pages are targets, never sources.** Pages like the home page and about page were built one-to-one, so they could receive redirects but were never redirected themselves.
- **Canonical first.** Canonical URLs were prioritized in matching.
- **Exceptions over guesses.** Uncertain matches were flagged, not forced.

## Key Engineering Challenge: Memory

Embedding thousands of URLs produced vectors too large for n8n's execution memory, and holding them there crashed the workflow. The fix was to split the run into a multi-part sub-workflow, with the number of parts scaling to site size. Each chunk was processed, its vectors written to Redis, and then dropped from execution memory before the next part. A final stage pulled the chunks back together for similarity matching and cleared the working data. Redis stores were keyed by execution number and site and kept for 30 days, which made runs traceable and allowed rematching.

## Tech Stack

n8n, Salesforce webhooks, Firecrawl, Google embeddings (interchangeable), Redis, cosine similarity, an AI agent for final selection, and Cloudflare redirect rules for host-level changes.

## Planned: Runnable Rebuild

A sanitized, runnable version of this matcher is planned for this repo. It will run on public or synthetic site data with open, self-hostable components: n8n, Postgres with pgvector, Redis, and local embeddings, producing a redirect CSV end to end.
