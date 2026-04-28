# Mindle v2.0 Roadmap

**Date:** 2026-04-28
**Status:** Living plan — adjust as releases land
**Context:** Builds on [`typora-market-research.md`](./typora-market-research.md) and [`reader-only-positioning.md`](./reader-only-positioning.md). The reader-only positioning won; the editor pivot is shelved. v2.0 is the milestone that turns Mindle into the calm collaboration surface for AI-driven markdown work.

---

## Vision

> *AI writes. You read, you mark up. The agent reads your notes back.*

No chat sidebar, no thread history, no editor surface — just the document, your annotations, and the agent's proposed changes as track-style diffs. The absence of those things is the differentiator. The "co-author in the office" feel is the product.

**The unique loop nobody else has:**

1. Agent writes (or revises) a markdown file via its own tools (Write/Edit).
2. Mindle picks up the change via file-watch and renders it as a *track-changes diff* against the last-synced version.
3. User reads, annotates passages with directives ("rewrite for non-technical readers", "expand this").
4. Agent calls `get_annotations` via Mindle's MCP, sees the directives, makes the requested changes.
5. Agent calls `clear_annotation(id, summary)` — the summary attaches to the corresponding diff chip in Mindle, closing the loop visibly.

Mindle's MCP is **read-only by design.** The agent already has Write/Edit; Mindle is purely the human-feedback channel.

---

## v1.5 — Foundation

Four independent ships, no architectural risk. Pick whichever lands cleanest first; nothing in v1.6 / v2.0 depends on the order.

- **File watch + live reload.** FSEvents-based. Debounce on bursts (size-stability check: don't reload until file size has been stable for ~200ms — guards against agent mid-write). Preserve scroll position. Preserve in-flight selection.
- **LaTeX rendering.** Vendor KaTeX into `Resources/web/vendor/` alongside Mermaid. Markdown-it plugin renders `$…$` and `$$…$$`. Table stakes for AI output (math is common in research-style markdown).
- **Quick Look extension.** Separate macOS app extension target. Renders the same HTML pipeline as the main app. Signed alongside the main bundle. Universal ask in every reader-launch thread; Smackdown shipping it earned immediate upvotes.
- **`.md` default-app handling polish.** Info.plist UTI declarations are already there; add a one-time, declinable "Make Mindle the default for `.md`" prompt on first launch.

## v1.6 — Diff-on-reload (the headline UX leap)

The Word-style track-changes surface, mapped to markdown. *Valuable on its own* — anyone editing the file in vim, Cursor, or Claude Code outside Mindle gets visible change tracking. Earns the "co-author in the office" feel before MCP is in the picture.

- **Snapshot model.** `lastSyncedText` per tab in `DocumentStore` (alongside `rawText`). Live-reload computes the diff against the snapshot, not just clobbers the view.
- **Diff engine.** Vendor a JS diff lib into `Resources/web/vendor/`. Bench `diff-match-patch` (gold standard, heavier) vs. `jsdiff` (lighter, coarser) on realistic-sized prose docs before committing.
- **Diff render.** CSS: insertions highlighted, deletions struck through. **Paragraph-block chunks with word-level inner highlights** — line-level too coarse for prose, character-level too noisy.
- **Accept / reject chips.** Per chunk. Accept clears the diff state and the new version becomes the baseline. Reject writes the original back to disk.
- **Sidecar extension.** Persist in-flight diff state so an unfinished review survives app restart.

The user never types markdown. They mark up, accept, reject. The "editor" *is* the diff review.

## v2.0 — MCP collaboration (the milestone)

The release that ties the loop together and earns the major version bump.

- **`mindle-mcp` helper binary** in `Mindle.app/Contents/MacOS/`. Stdio MCP server, talking to the running app via Unix socket (with distributed-notifications fallback). Clear error when no Mindle instance is running.
- **MCP tools (read-only by design):**
  - `list_open_files` — what's currently open in Mindle
  - `read_file` — file content + annotations in one call
  - `get_annotations` — annotations for a file
  - `clear_annotation(id, summary)` — mark addressed; summary attaches to the diff chip
- **Bundled `mindle-collaboration` skill** at `docs/skills/mindle-collaboration.md`. Install path published in the Mindle docs (drop into `~/.claude/skills/` for Claude Code; equivalent for other harnesses). The skill teaches:
  - **Tool list as deferred tools** — names only, no schemas in context until first use (zero context cost).
  - **The collaboration loop** — write file → suggest the user review in Mindle → wait for signal → `get_annotations` → revise → `clear_annotation(id, summary)`.
  - **Recognizing user intent.** *"Let me review in Mindle," "I'll mark it up," "annotate"* → stop editing and wait. *"Open in Mindle"* → ensure the file is written and suggest the user open it.
  - **Etiquette.** Don't poll. Trust note prose — don't ask for clarification on every annotation. Summaries should be concrete ("rewrote intro for non-technical readers"), not generic ("addressed feedback").
- **Theme polish pass.** "Themes that feel designed", not just light/dark/sepia variations. Smackdown's cyberpunk theme is a reference point for personality.
- **Landing-page rewrite + launch video.**

**Meta-loop worth mentioning in launch:** v1.4 shipped YAML frontmatter rendering as a code block specifically because `SKILLS.md` files have frontmatter. The skill we ship in v2.0 *is* a SKILLS-style file. Mindle renders its own skill beautifully.

---

## Out of scope for v2.0 (kept on roadmap, deferred)

- **WYSIWYG editing — decided not to ship.** The diff *is* the editor. Reverse decision only if the AI-output category collapses.
- iOS/iPadOS port
- Homebrew cask
- Presentation mode
- On-device AI Q&A (Apple Intelligence integration)
- Real-time multi-user collaboration

---

## Risks worth tracking

1. **File-watch races during agent saves.** Agent writes in chunks, fsync'd or not. Mitigation: debounce + size-stability check (no reload until file size stable for ~200ms).
2. **Diff library tradeoff.** Bench diff-match-patch vs. jsdiff with realistic prose docs before vendoring. Quality of word-level alignment matters for readability.
3. **MCP transport.** Stdio shim → Unix socket → running app is the cleanest macOS pattern but ties MCP to a running Mindle instance. Need a clear, friendly error when no Mindle is open ("Mindle isn't running — open it from Spotlight or Dock and try again").
4. **Skill harness adoption.** Claude Code is the obvious first target. Cursor, Continue, Cline, etc. each have their own skill-loading conventions. Ship the canonical file; document install paths for each harness as we get user feedback.
5. **AI-wave durability.** The viewer market existed pre-AI (Marked 2 had a decade); the diff-review surface is independently useful; the floor is real even if AI tooling shifts shape.
6. **User confusion if pivoting back.** Once positioned as "the AI-collaboration reader," adding a real editor later gets pushback. Stay disciplined — if editing comes, ship it as a deliberate Phase 3 with separate framing.

---

## Launch shape for v2.0

- **60-second demo video, no narration.** Open file → annotate → ask Claude → diff lands → accept some / reject some → annotations close.
- **Blog post on the loop.** Posted to r/ClaudeCode, r/macapps, r/Markdown.
- **Landing-page tagline rewrite.** *"AI writes. You read, you mark up. The agent reads your notes back."*
- **Reach out to @agasthik.** Closed two issues already (#1 auto-update, #3 tabs); the natural first user.
- **Submit `mindle-collaboration` skill** wherever there's a public skills index.

---

## Smallest viable first step

Ship the v1.5 foundation features, starting with **file-watch + live reload** since it's the prerequisite for v1.6. Each feature in v1.5 is independent — pick whichever lands cleanest day-to-day.

After v1.5, the path to v2.0 is mechanical: snapshot model + diff engine + diff render → accept/reject UI → MCP plumbing + skill → polish + launch.
