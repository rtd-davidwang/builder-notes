# builder-notes

> Open notes from a builder. Specs, scripts, and docs to help you ship your own AI-powered website + content pipeline.

This repo is the companion to videos and blog posts about building things with AI coding tools — Claude Code, Codex CLI, GitHub Actions, static hosting, content automation. If you watched a video and the description pointed here, you're in the right place.

---

## What's in here

| Folder | What it holds | Audience |
|--------|---------------|----------|
| [`specs/`](./specs/) | Drop-in Markdown specs you can hand directly to Claude Code or Codex CLI to scaffold a working system | Developers using AI coding tools |
| [`docs/`](./docs/) | Long-form explainers, architecture overviews, lessons learned | Anyone curious about how a system actually fits together |
| `scripts/` *(coming)* | Reusable shell / Python / YAML — deploy templates, image processing, skill scaffolds | Pick-and-grab |
| `video-scripts/` *(coming)* | Source scripts for accompanying videos (YouTube + WeChat 视频号) | Other content creators |
| `diagrams/` *(coming)* | Architecture diagrams, workflow diagrams (Mermaid + PNG) | Visual learners |

---

## Featured: How I built my company website + AI blog publishing in one weekend

Reference architecture: a code-first static site hosted on any SCP-capable host, deployed via GitHub Actions, with content creation handled by Claude Code custom skills. The flagship `/my-blog` skill takes a one-line topic and produces a fully published blog post with hero image in 15-25 minutes, with five human-in-the-loop checkpoints.

- 📋 **Spec to replicate it**: [`specs/replicate-website-spec.md`](./specs/replicate-website-spec.md) — agent-executable English spec. Drop into Claude Code or Codex CLI and follow the phases.
- 📖 **Architecture explainer (中文)**: [`docs/website-architecture-zh.md`](./docs/website-architecture-zh.md) — high-level walkthrough of the three-layer architecture (host / GitHub / Claude Code) and the end-to-end blog publishing pipeline.

---

## Audience

You'll probably get value from this repo if you are:

- A developer who wants to build a personal or company website without a CMS / database / SaaS lock-in
- An indie hacker or solo founder who wants to publish content without spending hours per post
- A content creator curious about how AI coding tools (Claude Code, Codex) compose into a real publishing system
- Someone evaluating whether to build vs. buy for a small marketing site

This repo is **not** a tutorial on how to use Claude Code generally. It assumes you have it installed and have used it at least once.

---

## What's *not* in here

- Production secrets, API keys, server IPs, or anything personally sensitive — those stay in private repos
- A no-code "click here to generate a website" tool — this is for builders comfortable with the CLI
- Maintained software with a SemVer release cycle — these are notes, evolving with each new video. See [License](#license) for what you can and can't do with the content

---

## How to use this repo

**For specs (`specs/`):** in any Claude Code or Codex CLI session inside an empty repo, paste the spec content into a chat or save it as `DEVELOPMENT_SPEC.md` and ask the agent to follow it phase by phase. Each spec ends with a "Done = …" verification checklist.

**For docs (`docs/`):** read top-to-bottom or jump to the section that matches your problem. Most docs are bilingual (English title, content may be in 中文 or English depending on the original audience).

---

## License

Mixed licensing to match content type:

- **Code, scripts, configuration files, and YAML** → [Apache License 2.0](./LICENSE) (patent grant + attribution)
- **Markdown documentation, specs, video scripts, diagrams, and slides** → [Creative Commons Attribution 4.0 International](./LICENSE-DOCS) (CC-BY-4.0)

Practically: you can use, modify, and redistribute everything here. For code you have a patent grant; for docs you must give attribution back to this repo. If a file's classification is ambiguous, default to CC-BY-4.0.

---

## Contributing

Issues and PRs are welcome — but this is a personal-notes-style repo, not a maintained framework. Best forms of contribution:

- ✅ **Issue**: "I tried the spec at `specs/X.md`, hit problem Y at step Z" — these become troubleshooting entries
- ✅ **PR**: typo fixes, broken-link fixes, command-line corrections
- ⚠️  **PR adding new content**: please open an issue first to discuss scope. New content typically lives elsewhere unless it directly improves an existing artifact

---

## About the author

Built and maintained by [@rtd-davidwang](https://github.com/rtd-davidwang). Companion videos on YouTube and WeChat 视频号 (links coming).

If a Hostinger referral helps the next viewer save 20% off their first hosting purchase, here's mine:
👉 https://www.hostinger.com?REFERRALCODE=ZRQCLOUDMYI1
(viewer gets 20% off, I get a small referral credit — disclosure: this is an affiliate-style link)
