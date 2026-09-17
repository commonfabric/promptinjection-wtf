# Wild West Roundup → promptinjection.wtf runbook

## Why

Roughly weekly, a "Wild West Roundup" markdown file is dropped in `~/Downloads` and
an agent is pinged to fold its items into the site's **Vulnerability Reports** feed.
The mechanical parts are easy to get subtly wrong in ways that lose data or duplicate
entries, so this runbook captures the procedure and the hard-won gotchas. Hand the
[agent prompt](#copy-paste-agent-prompt) at the bottom to a fresh agent, or just follow
the steps yourself.

The single highest-value rule: **every edit to `docs/index.html` must be purely
additive.** The on-disk file has at least once been a stale copy behind the committed
`HEAD`, which silently dropped committed entries on the next edit. The verify step exists
to catch exactly that.

## Repo

- Dir: `/Users/alex/Code/promptinjection.wtf` (git, branch `main`)
- Remote: `github.com:commonfabric/promptinjection-wtf.git`
- The only file you edit for a roundup: `docs/index.html`
- GitHub Pages serves from `docs/` (`docs/CNAME` → promptinjection.wtf), so this runbook
  and other root-level docs are **not** published.
- Leave `.playwright-mcp/` and `.claude/` untracked; never commit them.

## Procedure

### 1. Find & read the roundup
- Newest markdown in Downloads: `ls -lt ~/Downloads/*.md | head`. Filenames vary wildly in
  spelling ("Wild West", "Roundup", "Rondupt", "Roundupt", with/without the year), so take
  the newest by mtime — don't pattern-match the name.
- Read it. It's a bullet list of links, each with a short blurb, sometimes nested sub-bullets.

### 2. Research every link
- Load the web tools first (they're deferred): `ToolSearch` with query
  `select:WebFetch,WebSearch`.
- `WebFetch` **every** link. For each, capture: the attack/vuln, technical mechanism,
  affected product/version, dates, named researchers/orgs, CVE, fix/vendor response, and the
  core takeaway.
- Some domains 403 or truncate (seen: techtimes, darkreading, axios, armadin; techradar
  truncates). When a fetch fails, `WebSearch` for the primary source or clean secondary
  coverage and use that. If an item has a primary writeup **and** notable secondary coverage,
  cite **both** links in the one entry.

### 3. Decide what goes in (editorial)
- **ADD** genuine vulnerabilities / attacks / in-the-wild campaigns → these become `incident`
  entries in Vulnerability Reports.
- **Research papers, taxonomies, frameworks, analyses** → these go in the **Learn More**
  section as `<li>` items, **not** as incident divs. Pick the right subsection:
  "Foundational Research", "Attack Techniques", "MCP Vulnerabilities", or
  "Browser & Agent Failures".
- **SKIP** (and tell the user why): vendor surveys / marketing (e.g. DigiCert "% of
  enterprises hit"), pure adoption/opinion pieces, and pure hallucination/quality items with
  no adversary (e.g. the Google-AI-Overviews-SCP-fiction story). Standing skip: the banteg/X
  post about worms saying forbidden things to stop security scanners.
- Real-world agent-**mishap** anecdotes that aren't "attacks" (an agent deleting files,
  sending an unasked email) **can** be added as a single thematic entry when they illustrate
  the site's point — precedent: the Claude Cowork deleted-photos entry and the GPT-5.6
  horror-stories entry.
- Honor any explicit skip/keep instruction in the ping.

### 4. Check for duplicates (before writing)
- For each candidate, `grep docs/index.html` for distinctive tokens: the codename, CVE,
  researcher, and product. Also scan the large OpenClaw/ClawHub mega-paragraph, which already
  absorbs a lot of ClawHub-skill news.
- If something is already covered (has happened, e.g. "ClaudeBleed"), do **not** duplicate —
  if the new item is a genuine follow-up, write a new entry that explicitly cross-references
  the original ("see &lt;Month 2026&gt;, further down").

### 5. Write the entries
- Insert new incident entries at the **TOP** of the Vulnerability Reports list, immediately
  after the intro paragraph that begins "These aren't theoretical attacks...", newest first.
- Match the existing house style exactly:
  ```html
  <div class="incident">
      <div class="incident-date">Month 2026</div>
      <h4>Short punchy title (CVE if any)</h4>
      <p><a href="URL">Researcher/Org</a> ... one dense paragraph: mechanism, specifics,
      dates, fix, takeaway.</p>
  </div>
  ```
- **Date label = the month the item surfaced in this roundup** (a roundup dated 8/3/26 →
  "August 2026"), even if the underlying writeup is a few days older.
- Use `<strong>`, `<code>`, `<em>` for emphasis. Escape angle brackets in prose/code as
  `&lt;` `&gt;` (e.g. an HTML comment or a `<div>` you're describing).
- One dense paragraph per entry, in the voice of the surrounding entries — read a few existing
  ones first to calibrate tone.
- Learn More items are plain `<li><a href="URL">Title</a> - one-to-three sentence summary.</li>`

### 6. Verify (critical — the on-disk file has been stale before)
```bash
cd /Users/alex/Code/promptinjection.wtf
git diff --stat docs/index.html        # MUST be purely additive: N insertions, 0 deletions.
                                        # ANY deletions = investigate, do not commit.
# balance: these two counts MUST be equal
grep -c '<div class="incident">' docs/index.html
grep -c '<div class="incident-date">' docs/index.html
# prior entries still present in BOTH working tree and HEAD (base-not-stale check)
git show HEAD:docs/index.html | grep -c "<a prior entry title>"
grep -c "<a prior entry title>" docs/index.html
# no unescaped raw markup introduced in prose:
git diff docs/index.html | grep '^+' | grep -E '<div|<!--' || echo "clean"
```
- If the base looks stale (committed entries missing from the working tree):
  `git checkout HEAD -- docs/index.html`, then re-apply your additions on the HEAD version.

### 7. Commit & push
- Stage only `docs/index.html`.
- Message: first line `Add M/DD Wild West roundup: <short comma list of items>`, then a body
  with one bullet per added item, and a line noting anything you **skipped** and why.
- End the commit message with this trailer (update the model name if you're not Opus 4.8):
  ```
  Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
  ```
- Push to `main` (pre-authorized for this recurring task).
- Report back: what you added, what you skipped and why, any fetch that failed and how you
  worked around it, and the pushed commit hash.

## Gotchas

- **`~/Downloads` can wedge.** It's backed by a cloud file-provider (Google Drive/iCloud) that
  occasionally hangs: `ls` on the folder blocks or returns "Interrupted system call" (EINTR),
  and files are dataless placeholders that block on read. A direct-path stat
  (`[ -f "$HOME/Downloads/<exact name>" ]`) still works even when `ls` hangs, but reading the
  bytes may still block. Don't kill the user's apps. Ask the user to paste the file text into
  chat, or materialize it (Finder → right-click → Download Now) / restart Google Drive, then
  retry. Don't burn minutes hammering a wedged folder.
- **Deferred tools.** `WebFetch`/`WebSearch` (and browser/computer-use tools) aren't loaded by
  default — load via `ToolSearch` before calling.
- **Duplicates hide in the mega-paragraph.** The single large OpenClaw/ClawHub `<p>` already
  contains dozens of linked items; grep it before adding anything ClawHub/OpenClaw-related.

## Copy-paste agent prompt

The runbook above is the reference. To delegate a single run to a fresh agent, paste everything
between the lines below:

---
You maintain the "Vulnerability Reports" feed on promptinjection.wtf. A "Wild West Roundup"
markdown file was dropped in ~/Downloads. Read `/Users/alex/Code/promptinjection.wtf/ROUNDUP_RUNBOOK.md`
and follow it exactly: find the newest roundup md in ~/Downloads, WebFetch every link, add the
genuine new vulns/attacks as `incident` entries at the top of Vulnerability Reports in
`docs/index.html` (research papers go in Learn More instead), run the additive-diff + balance
verification, then commit only `docs/index.html` and push to main. Report what you added, what
you skipped and why, and the commit hash.
---
