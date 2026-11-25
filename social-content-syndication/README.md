# Social Content Syndication

An opt-in, event-driven n8n automation that turns every published client blog post into a ready-to-post social media package. An AI agent writes platform-specific copy, an image API produces correctly sized images, and the client receives everything in one email within minutes of publishing.

## Why It Is Hard

Each social platform has its own text limits and image shapes, and a post that ignores them gets cropped or truncated. Publishing directly through platform APIs would have avoided the manual step, but it requires every client to create developer accounts and apps, and those APIs change and break. The goal was to remove nearly all the work for a non-technical client without taking on that support burden.

## Architecture

1. **Trigger and extraction.** Publishing a blog post fired a webhook that started the workflow. It collected the header image, body content, H1, metadata, and canonical URL from the published post.
2. **Platform formatting.** The workflow passed the content through a separate step for each of three supported platforms: Twitter, Facebook, and Instagram. The agent wrote the copy to each platform's ad display specs. Twitter had a character limit, and every platform had headline limits based on display width and body text limits for the preview, so nothing would be cropped.
3. **Image generation.** The blog's large, high-resolution header image served as the single source image. A CDN image API adapted it, with aspect ratio as the primary change: roughly 1.91:1 for Facebook's link card and 1:1 square for Instagram, each following the platform's standard ad ratio. An Open Graph image was generated to match.
4. **Report assembly.** All platform outputs were combined into one report, sectioned by platform. Each section told the client the new post was live and gave them the text to copy, the image to upload, and a link to posting instructions.
5. **Delivery.** The report was emailed directly to the client after the workflow was vetted, with the team keeping visibility into runs. Clients expected the email shortly after each publish, so someone on their side could post to every platform in a few minutes.

## Key Design Decisions

- **No direct API publishing.** Posting by API requires each client to create developer accounts and apps, and platform APIs change and break. Instead of taking on that support burden, the workflow delivered finished content for the client to post themselves.
- **Opt-in service.** Customer success managers offered it to clients who wanted to streamline social posting, so every email was expected and wanted.
- **One report per post.** Everything arrived in a single sectioned email with plain instructions, so a non-technical person could finish the job in minutes.
- **Formatted to ad specs.** Text limits and image ratios followed each platform's ad display specs, so a client could promote any post as an ad without recropping or losing text.
- **Webhook over listener or RSS.** Early iterations used a listener and then RSS polling. The final version used a webhook fired on publish, which was the most reliable and immediate way to trigger the workflow.
- **Fast turnaround.** The email arrived shortly after each publish, while the post was still fresh and the client was ready to share it.
- **Visibility without a manual gate.** Reports were generated and sent automatically, while the team kept insight into workflow runs.

## What I Would Do Differently

The original used one workflow with a separate pass for each platform. In a rebuild, I would split it into one sub-workflow per platform. Each platform would be isolated, so a failure on one would not affect the others, and debugging would be easier because each sub-workflow has its own execution history. It would also make adding a new platform a matter of adding one more sub-workflow.

## Tech Stack

n8n for orchestration, branching, and scheduling. A webhook fired by the website on publish. An AI agent powered by Claude Sonnet for platform-specific copy. A CDN image API for scaling, cropping, and generating Open Graph images. Email delivery of a single consolidated report per post, with linked documentation on posting to each platform manually.
