# Project Memory Log: Simulation Theory Rhymer

*(Last updated: June 17, 2026, Phase 8 session — supersedes the earlier June 17, 2026 version)*

## Original Problem

The user wanted to build a better version of an existing game, "rhymerush.com," which had two specific flaws:
1. It didn't reveal missed rhymes at the end of a round — no way to see what you didn't find.
2. Its dictionary was incomplete and rejected valid rhymes.

The goal from the start was a self-contained, no-AI-dependency game using a real phonetic dictionary, deployed as a static GitHub Pages site.

## How the Problem Evolved

### Phase 1: Architecture decision
Chose phoneme-based rhyme classification using the CMU Pronouncing Dictionary (~126k words, confirmed at 126,052 words after processing) rather than any AI/LLM-based rhyme judgment. Deliberate choice for reliability, speed, and offline capability.

Rhyme classification logic:
- **True rhyme**: exact phoneme match from the last stressed vowel to the end of the word (stress-stripped comparison).
- **Slant rhyme**: same trailing consonants after the stressed vowel, OR same stressed vowel itself, but not both.
- **Not a rhyme**: neither condition met.

### Phase 2: Embedding the dictionary (three iterations)
1. Hand-picked ~800-word CMU subset in-chat — failed (too small/arbitrary; "phone" missing).
2. Fetch full CMU dict at runtime from GitHub raw URL — failed (CORS/NetworkError via `file://`).
3. **Final approach (kept)**: download full CMU dict once, process into compact JSON in Python, embed directly inline in the HTML as a JS object literal. Result: fully self-contained single HTML file, zero network requests, works from disk or any static host. This embed-everything-inline pattern is the standing approach for all data in this project (CMU dict, random-word filter list).

### Phase 3: Deployment
Repo: `https://github.com/elliottlj/spittingrhymes` (created mid-project). Plan: upload `index.html` via GitHub web UI → Settings → Pages → Deploy from branch → main → /(root). Live URL will be `https://elliottlj.github.io/spittingrhymes`.

**Status: still not uploaded / Pages still not enabled as of June 17.** This remains a pending manual action for the user, not something blocked on further development.

### Phase 4: Iterative feature requests (all implemented, in order)
1. Timer duration selector (30s / 60s / 90s default / 120s / 3 min).
2. Missed-rhymes screen: only true rhymes shown (slant/grey rhymes removed from the missed list).
3. Investigated CMU quirk: "schack" / "beaulac" are real CMU entries but surnames, not common words (CMU mixes proper nouns into its corpus — this insight became important later, see Phase 6).
4. Random word filtering: filtered pool (~21,006 words, later ~4,834 in the embedded `randomWords` array as of the current file) built by intersecting CMU dict with `dwyl/english-words` common wordlist AND requiring ≥8 true rhymes per word.
5. Shareable URLs: target word in URL hash (`#word`); read on load, pre-fills word input; `startGame()` updates hash silently.
6. Share UI: results screen shows share URL + "Copy link" button (`navigator.clipboard.writeText()`) with 2-second "✓ Copied!" confirmation.
7. Scoring: true rhyme +10, slant rhyme +2, not-a-rhyme +0.
8. Rebrand: "Rhyme Rush" → **"Simulation Theory Rhymer"** throughout.

### Phase 5: Persistence question (discussed, NOT implemented)
User asked about persisting score history per word via `localStorage`. Established: 5MB/origin hard limit; for score-only data even all ~21k words would be well under 1MB; recommended try/catch guard around `setItem`/`getItem`. **Open design decision, still unresolved**: per-word best score vs. lifetime/cumulative total. User explicitly said "don't do this yet" — deferred, no code written. Still deferred as of June 17; not revisited this session.

### Phase 6: Apostrophe bug — discovered and partially fixed (June 17 session)
This session picked up from a prior chat that had run out of room. The user uploaded the latest working file (`index_1_.html`), which already contained a first-pass fix attempt from the cut-off session. That fix was real but incomplete — it solved one part of the apostrophe problem but not the deeper one identified below.

**The underlying issue:** CMU Pronouncing Dictionary entries like `boats` and `boat's` (or `capote's` and similar) often have **identical phoneme sequences** — apostrophes don't change pronunciation, so CMU stores possessive/contraction forms as separate dictionary keys with the same sound as their base form. This created two distinct bugs:

1. **Bug A (fixed in the prior session, before this chat started)**: When a non-target rhyme word had both an apostrophe and non-apostrophe form in the missed-rhymes list (e.g. target "start", missed list contains both "goats" and "goat's"), both were shown as separate entries even though they're the same rhyme. Fixed via a `normalizeForDedup` function (strips `'s` → `s`, strips remaining apostrophes) plus a grouping step that collapses variants into one displayed entry, preferring the non-apostrophe form.

2. **Bug B (found and fixed this session)**: When the **target word itself** had an apostrophe-twin elsewhere in CMU with identical phonemes (e.g. target = "boats", and "boat's" exists in CMU with the exact same phonemes `B OW1 T S`), the original exclusion check in `getAllTrueRhymes()` only filtered `word === target` (exact string match). Since `"boat's" !== "boats"` as strings, the apostrophe-twin of the target slipped through and was listed as if it were a separate valid rhyme of itself — appearing in the missed list as a confusing, ungrammatical "rhyme."
   - **Fix applied**: hoisted `normalizeForDedup` to a shared top-level function (previously only declared locally inside the missed-rhymes calculation block) and added a check inside `getAllTrueRhymes()`: any candidate word whose normalized form matches the normalized target is now excluded, not just exact string matches. Verified via direct phoneme lookup that "boat's" no longer appears in the true-rhyme list for target "boats" (count dropped from 29 to 28 — exactly one entry removed, no collateral loss).
   - User confirmed this fix resolved the "boats" test case.

**Remaining annoyance (not a bug, a UX/design issue) — "capote's" case:** even after both fixes above, apostrophe-containing words can still legitimately appear in the missed-rhymes list when they are the *only* dictionary form for that particular sound (i.e., there's no non-apostrophe twin to prefer instead, or the apostrophe word is a genuinely separate true rhyme of some other word in the list, not a duplicate of anything). Example: "capote's" appeared as a missed rhyme. The user's objection isn't that it's wrong as a rhyme — it's that **in a timed typing game, requiring the player to type an apostrophe is too slow/fiddly**, especially under time pressure. This is a design/scope question, not a correctness bug.

### Phase 7: Apostrophe-stripping investigation (discussed, decision pending)
In response to the "capote's" friction, the user asked whether to **strip all apostrophe-containing words from the embedded CMU dataset entirely**, and what the drawbacks/word-loss would be. Investigated and quantified:

- **7,478 of 126,052 words (5.93%)** in the embedded CMU dict contain an apostrophe.
- Of those, **3,155 are pure phonetic duplicates** of an existing non-apostrophe word already in the dictionary (e.g. "boat's" duplicating "boats") — these can be removed with **zero loss of distinct rhyme sounds**.
- The remaining **4,323 have no non-apostrophe twin already in the dictionary** — these are overwhelmingly possessive forms of proper nouns/names (e.g. "abiola's", "aaronson's", "acker's") where the base form isn't itself a separate CMU entry. Removing these would **genuinely delete that phonetic shape from the dictionary**, not just deduplicate it.
- **Practical impact on the existing random word pool** (currently embedded `randomWords`, ~4,834 words): checked every word's true-rhyme count with vs. without apostrophed entries. **94 words (~2% of the pool)** would drop below the ≥8-true-rhyme threshold if apostrophed words were stripped (e.g. "answers" drops from 8→4 true rhymes, "apples" from 8→4, "actors" from 10→5) — these are cases where a large fraction of a word's rhyme partners were themselves possessive forms.
- Tradeoff summarized for the user: stripping fixes the typing-friction problem permanently and simplifies the codebase (could remove the whole dedup/grouping mechanism from Phase 6 entirely), but costs ~4,323 dictionary entries outright and would require re-filtering ~94 words out of (or re-justifying their inclusion in) the random pool.

**Decision status: NOT yet made.** Claude presented the tradeoff and word-loss numbers; the user had not yet said go/no-go on implementation when the session was checkpointed via this memory log save. This is the single most important open item for the next session.

### Phase 8: Build architecture overhaul — splitting data out of the HTML (decided and implemented, June 17 session)
**The problem this solves:** the user reported having to repeatedly start new chats because uploading the 6.1MB single-file HTML (with the full CMU dict and randomWords array embedded inline) consumed a huge fraction of the chat's context capacity every time it needed to be re-uploaded for further edits. The embed-everything-inline pattern from Phase 2, while correct for *deployment*, was actively hostile to *iterative development* in chat.

**Decision (made this session):** split the single file into three artifacts instead of one:
1. **`dict.json`** (or two JSON files, one for `cmuDict` and one for `randomWords`) — the real, full dictionary data. Generated once by the existing Python pipeline (Templates section), unchanged in content. This is what's large; it should almost never need to be re-read into chat context.
2. **`index.html`** — production file. Same game logic/UI as before, but loads the dictionary via `<script src="cmudict.json"></script>` / `<script src="randomwords.json"></script>` instead of inline JS object literals. This is the file that goes to GitHub Pages and the file that should be uploaded/edited in chat going forward — now only a few KB.
3. **`local_test_index.html`** — a generated (not hand-maintained) variant for offline/`file://` testing without a server. Built by taking the production `index.html` and replacing the external `<script src="...">` tags with an inline `<script>const cmuDict = {...}; const randomWords = [...];</script>` block containing only a small hand-picked subset of the dictionary (enough words/rhyme-tails to exercise normal play, slant rhymes, an apostrophe-twin case, a low-rhyme-count edge case, and a couple of actual `randomWords` pool entries — a few hundred dict entries total, not thousands). Generated programmatically from `index.html` so game logic never has to be maintained in two places — only the data-loading mechanism differs between the two HTML files.

**Known tradeoff accepted:** splitting `index.html` from its data means it is no longer a single self-contained file and will fail if opened directly from disk via `file://` (same CORS issue originally hit in Phase 2 step 2) — `local_test_index.html` exists specifically to cover that local-testing need instead. GitHub Pages serves all three files together fine since that's not `file://`, so production deployment is unaffected.

**Important scope note:** `local_test_index.html`'s small subset is for testing UI/flow/scoring/sharing — it will NOT catch dictionary-scale bugs (like the Phase 6 apostrophe issues, which only surfaced because of patterns specific to the full 126k-word CMU dict). It is not a substitute for testing against the real dictionary when the bug class is data-dependent rather than logic-dependent.

**Implementation completed this session — important data-fidelity caveat (read before next session touches word lists):**

The user could not re-upload the full 6.1MB working file (hits the same context problem this work was meant to solve), so the approach taken was: user manually stripped the two/three large embedded data arrays out of the file locally and uploaded only the remaining ~15.5KB of HTML/JS/CSS logic. Claude then **regenerated** the data files from public sources rather than extracting them from the original file. Results:

- **`cmudict.json` (126,052 words) — exact match.** Regenerated from `cmusphinx/cmudict` master via the documented pipeline (Templates section). Raw line count (135,166) and final word count (126,052) both matched the memory log's recorded figures exactly, and a direct test of the Phase 6 "boats" regression case (28 true rhymes, "boat's" twin correctly excluded) matched the documented result exactly. High confidence this is byte-equivalent in content to the original.
- **`randomWords.json` (5,415 words) — approximate, NOT exact, gap is understood and accepted by user.** The original ~4,834-word pool turned out to use an undocumented extra pipeline step (revealed by the user this session): start from the top 10,000 words by frequency from "the wordfreq dataset" (no further specifics given), intersect with CMU (→9,828 in original), intersect with dwyl/english-words (→9,571), filter to ≥8 true rhymes (→4,922), then a **manual** short-word cleanup pass (length ≤3 words: drop single letters and most abbreviations/name fragments, keep interjections and a few named exceptions like "pro"/"gig" → final 4,834). Claude reconstructed this using a public OpenSubtitles-frequency wordlist (`hermitdave/FrequencyWords` en_50k) as a stand-in for "the wordfreq dataset" since the exact source was never specified, and got checkpoint numbers close but not identical at every step (9,753 / 9,431 / 5,446 vs. the original's 9,828 / 9,571 / 4,922). Rather than attempt to replicate the user's manual short-word judgment calls (~500 short words would have needed individual keep/cut decisions to land on the original's -88 cut), Claude applied only the unambiguous rule-based part (drop bare single letters, drop a handful of obvious abbreviations like "dc"/"st"/"ng") and stopped there, landing on **5,415 words**. **User explicitly approved shipping this approximation** ("Good enough, ship my reconstruction") rather than chasing exact parity. If exact parity ever matters later, the fix is for the user to supply the actual frequency-list source and/or the exact 88-word manual-cleanup decision list — both are fully specified in this paragraph for reconstruction.
- **`commonWords.json` (57,884 words) — deliberately redefined, not reconstructed.** The original used the full 370,105-word `dwyl/english-words` list inline, which would have produced a 4.6MB JSON file on its own — defeating the entire point of this refactor. Per user's explicit choice, Claude instead restricted `commonWords` to the intersection with `cmuDict` (since `isStandard()` is only ever called on words that are already confirmed `cmuDict` keys, so the 370k-word universe was always functionally wasted space). This is a deliberate, approved scope change, not a fidelity gap — behaviorally identical to the original for every real code path, just smaller on disk (623KB vs 4.6MB).

**Verification performed:** local Python HTTP server, confirmed all 4 files (`index.html` + 3 JSON) serve with 200 status; confirmed `index.html` contains exactly 3 `fetch()` calls and zero embedded data; confirmed `local_test_index.html` contains zero `fetch()` calls, zero external `src`/`href` references, and the full inline mini-dataset; confirmed JS syntax parses cleanly in both files; re-ran the Phase 6 "boats" regression test against the regenerated `cmudict.json` directly in Node and got the exact documented result (28 true rhymes, apostrophe-twin excluded).

## Key Insights and Solutions

- **CORS forces the embed-everything pattern.** Any plan to fetch large dictionaries/wordlists at runtime from a local `file://` HTML page will fail; the only robust solution for a single-file, no-server deployment is embedding data inline as JSON in a `<script>` block.
- **CMU dictionary contains proper nouns/surnames** mixed in with common words, with no built-in flag to distinguish them. Solved via cross-referencing against `dwyl/english-words`'s `words_alpha.txt`.
- **Rhyme-richness filtering** (≥8 true rhymes) is an effective second-pass filter that also incidentally removes obscure/sparse-rhyming proper nouns.
- **URL hash routing** (`#word`) is the lightweight way to make state shareable without a backend.
- **Deferred computation pattern**: missed-rhymes scan wrapped in `setTimeout(..., 50)` so the results screen paints first and a "Calculating missed rhymes…" message displays before the scan runs.
- **NEW — Apostrophes are a phonetic non-event in CMU.** CMU encodes possessives/contractions as separate dictionary keys with identical phoneme strings to their base form. This is invisible until you specifically compare phoneme arrays across apostrophe/non-apostrophe pairs — it doesn't show up as a "wrong" rhyme, it shows up as a *duplicate* or a *confusing self-referential* entry. Any future phonetic-dictionary work on this project should assume apostrophe-variants need explicit handling, not just trust that string-equality checks (`word === target`) are sufFicient for "is this the same word" logic — phonetic-equality (via normalized string) is the correct test.
- **NEW — "Pure duplicate" vs. "irreplaceable" apostrophed words are a meaningful split.** Before bulk-removing any class of words from a phonetic dataset, check whether removing them destroys a unique sound (irreplaceable) or just removes a redundant spelling of an existing sound (pure duplicate). This distinction directly drove the go/no-go analysis in Phase 7 and is a reusable technique if other word classes (e.g. abbreviations, hyphenated compounds) come up later.

## User's Working Style and Preferences Observed

- **Iterative, feature-at-a-time requests.** Consistently asks for one or a small batch of concrete changes rather than re-specifying the whole project each time.
- **Asks clarifying/diagnostic questions before committing to a feature or destructive change.** This session's apostrophe-stripping question is a clear example: asked for drawbacks and word-loss counts *before* approving implementation, exactly matching the established "ask before building" pattern from earlier phases.
- **Explicit about scope control.** Distinguishes precisely between "let's discuss" and "go build it." Has not yet given the go-ahead on apostrophe-stripping (Phase 7) — Claude should not implement it until explicitly told to.
- **Comfortable with technical detail**, including phoneme-level debugging (e.g. was satisfied with a precise root-cause explanation — "boat's has identical phonemes to boats, and the exclusion check was only doing exact string match" — rather than just "fixed it").
- **Cares about real-world play experience, not just correctness.** The "capote's" objection is not a bug report in the traditional sense — it's a UX complaint about typing friction in a timed context. Claude should keep distinguishing "is this technically a valid rhyme" from "is this a good thing to ask a time-pressured player to type."
- **Direct, terse phrasing**, sometimes with minor typos. Responses should stay concise.
- **Wants the deliverable to "just work."** Self-contained/no-network-dependency solutions remain the standing requirement.
- **Cross-session continuity matters a lot to this user.** This session opened with explicit confusion/frustration about losing context across long chats, and explicit interest in using Claude Projects and memory logs to avoid repeating that pain. Saving and maintaining this log accurately and promptly is itself a tracked user need, not just a nice-to-have.

## Collaboration Approaches That Worked Well

- Presenting the finished file via `present_files` after each batch of changes.
- Verifying facts with direct computation/lookup rather than guessing (e.g., this session: parsing the actual embedded CMU JSON out of the HTML file with Python to check real phoneme arrays for "boats" vs. "boat's", and to count apostrophed-word statistics) — mirrors the established "grep-verified lookup" pattern from the schack/beaulac investigation in Phase 4.
- When asked about a new idea/change without explicit approval to implement (apostrophe-stripping), laying out concrete numbers and tradeoffs and stopping there, rather than jumping to code — matches the user's "ask before building" expectation precisely.
- When the user reported a fix was incomplete ("the bug isn't completely fixed, I'm still testing"), not assuming the existing fix was wrong — instead asking for and using a concrete failing example ("boats") to find the *actual* root cause, which turned out to be a different mechanism (target-exclusion logic) than the one already fixed (cross-rhyme dedup).
- Hosting the actual game file as a presented artifact so the user could test interactively in the chat window, rather than only describing changes in text — directly served the user's stated need to verify bug fixes themselves.

## Clarifications or Corrections the User Made

- Clarified the apostrophe issue was not fully resolved despite the file containing a fix — prompted deeper root-cause investigation rather than accepting the existing fix as sufficient.
- Specified the exact failing test case ("boats") rather than leaving Claude to guess at reproduction steps.
- Distinguished a second, related-but-different concern (capote's / typing friction) from the original bug report, after the original bug was confirmed fixed — Claude correctly tracked these as two separate issues rather than conflating them.
- Earlier in the project: specified exact scoring values (true=10, slant=2), the exact new name ("Simulation Theory Rhymer"), and explicitly paused implementation ("don't do this yet") on localStorage — all still standing/respected.

## Project Context and Examples Used

- Repo: `https://github.com/elliottlj/spittingrhymes` (Pages still not enabled as of June 17).
- Reference competitor: rhymerush.com.
- Dictionary source: `https://raw.githubusercontent.com/cmusphinx/cmudict/master/cmudict.dict` (135,166 raw lines → 126,052 unique words).
- Common wordlist source: `https://raw.githubusercontent.com/dwyl/english-words/master/words_alpha.txt` (370,105 words, no proper nouns).
- Example words used for dictionary-quirk/bug testing across the project: "phone," "nation," "station," "notion," "orange," "silver," "schack," "beaulac," "start," and (this session) **"boats" / "boat's"** and **"capote's"**.
- Output/working file: `/home/claude/game.html` during this session, copied to `/mnt/user-data/outputs/simulation-theory-rhymer.html` for presentation. (Earlier sessions used `/mnt/user-data/outputs/index.html` as the filename — naming has been inconsistent across sessions; worth standardizing next time.)
- File size as of this session's last save: **6,115,486 bytes** (~6.1MB), up slightly from the ~5.4MB figure recorded after Phase 4 (growth mainly from the embedded random-word pool and dictionary).

## Templates or Processes Established

**Dictionary processing pipeline (Python, run in the sandbox each time the dict needs rebuilding):**
1. `curl` the raw CMU dict file.
2. Parse line by line, strip `;;;` comments, lowercase the word, strip parenthetical pronunciation-variant suffixes like `(2)`, keep only the first pronunciation per word, store as `{word: [phones...]}`.
3. `json.dumps(..., separators=(',', ':'))` for compact embedding.
4. Embed the resulting JSON string directly into a `<script>` block as `const cmuDict = {...};`.

**Rhyme tail / classification algorithm (JS, stable across all iterations):**
```js
function getRhymeTail(phones) {
  for (let i = 0; i < phones.length; i++) {
    if (/[12]$/.test(phones[i])) return phones.slice(i);
  }
  return phones.slice(-2);
}
function stripStress(phones) {
  return phones.map(p => p.replace(/[012]$/, ''));
}
function classifyRhyme(targetWord, guessWord) {
  // true: stripStress(tailA) === stripStress(tailB)
  // slant: same trailing consonants after the stressed vowel, OR same stressed vowel
  // null: otherwise
}
```

**NEW — Apostrophe-normalization pattern (JS, added this session, now a shared top-level function):**
```js
function normalizeForDedup(w) {
  return w.replace(/'s$/, 's').replace(/'/g, '');
}

function getAllTrueRhymes(target) {
  const tP = cmuDict[target];
  if (!tP) return [];
  const tTail = stripStress(getRhymeTail(tP)).join(' ');
  const targetKey = normalizeForDedup(target);
  return Object.keys(cmuDict).filter(word => {
    if (word === target) return false;
    if (normalizeForDedup(word) === targetKey) return false;  // excludes apostrophe-twins of the target
    const gTail = stripStress(getRhymeTail(cmuDict[word])).join(' ');
    return gTail === tTail;
  });
}
```
The missed-rhymes display logic (in the results-screen `setTimeout` block) separately groups remaining apostrophe-variant *pairs* (where neither is the target) using the same `normalizeForDedup` function, preferring the non-apostrophe spelling when one exists in the group.

**Apostrophe-word audit technique (Python, one-off but repeatable — used in Phase 7):** to evaluate any future "should we remove this class of words" question, (1) parse the embedded `cmuDict` JSON out of the live HTML file with Python, (2) for each candidate word, check whether a "base form" guess (stripping `'s` or all apostrophes) exists elsewhere in the dictionary with identical phonemes — if yes, it's a safe-to-remove pure duplicate; if no, removing it deletes a unique sound, (3) separately re-check the *existing* `randomWords` pool's rhyme counts with vs. without the candidate class removed, to quantify real gameplay impact (not just dictionary-size impact).

**Random word pool construction (Python, one-off but repeatable):**
1. Load CMU dict JSON.
2. Load common wordlist, build a Python `set`.
3. Build a rhyme-tail index across the whole CMU dict (tail → list of words sharing that tail).
4. Filter: word must be in common wordlist AND have ≥8 other words sharing its true-rhyme tail.
5. Sort, dedupe, dump as compact JSON, embed as `const randomWords = [...];`.

**File delivery process:** build/edit in `/home/claude/[filename].html`, then `cp` to `/mnt/user-data/outputs/[filename].html`, then call `present_files`. Always re-view the working file before further `str_replace` edits since prior view output goes stale after edits. **Note for next session:** filename has varied (`index.html`, `game.html`, `index_1_.html`, `simulation-theory-rhymer.html`) across sessions/uploads — pick one canonical name going forward to reduce confusion when re-uploading the "latest" file.

**Deployment process (manual, GitHub web UI, no CLI/git needed) — unchanged, still pending:**
1. Add file → Upload files → drag the HTML file → commit to `main`.
2. Settings → Pages → Source: Deploy from a branch → Branch: main, folder /(root) → Save.
3. Live at `https://elliottlj.github.io/spittingrhymes` after ~1 minute.

**Project-continuity process (established this session, in response to the user's question about losing context across long chats):**
- When a chat is getting long, or at the end of a productive session, generate/update this single canonical memory-log markdown file (not a new one each time) and save it to Claude Project files.
- Also keep the actual current working HTML file saved to project files (or readily re-uploadable) alongside the log, since the log describes decisions/rationale but a new chat still needs the real file to keep editing.
- Checkpoint at natural milestones (a feature shipped, a bug fixed, a deploy step done) rather than waiting for a length-limit cutoff, since there's no visible warning before a chat hits its limit.

## Next Steps Identified (Not Yet Done)

1. **Phase 8 build split: DONE this session.** Five files delivered: `index.html` (production, fetches 3 JSON files, ~16KB), `local_test_index.html` (offline-testable, ~18KB, tiny inline dataset covering boats/cat/orange/silver + rhymes), `cmudict.json` (5.4MB, exact match to original), `randomwords.json` (45KB, 5,415 words, close-but-not-exact reconstruction — see caveat above, user approved), `commonwords.json` (623KB, 57,884 words, deliberately redefined as cmuDict ∩ dwyl rather than full dwyl list). All verified working via local HTTP server test and a Node-based regression test of the Phase 6 "boats" fix.
2. **Decide on apostrophe-stripping (Phase 7).** Still the most immediate *content* decision (independent of the Phase 8 build-architecture work, which is now done). If the user says go: remove all apostrophe-containing entries from `cmudict.json`, rebuild `randomwords.json` re-applying the ≥8-true-rhyme filter, and likely simplify/remove the `normalizeForDedup`/grouping dedup logic. If no: dedup logic stays, "capote's"-style friction remains an accepted rough edge.
3. **User has still not uploaded anything to the GitHub repo or enabled GitHub Pages.** Now means uploading 4 files together (`index.html`, `cmudict.json`, `randomwords.json`, `commonwords.json` — `local_test_index.html` is for local use only, not needed on Pages). Immediate next action whenever the user is ready.
4. **localStorage score persistence** — still discussed-but-deferred from Phase 5. Open design decision still needed: per-word best score vs. lifetime cumulative score (or both). Not revisited this session; remains paused exactly as the user left it.
5. **Filename convention is now fixed** per Phase 8: `index.html`, `local_test_index.html`, `cmudict.json`, `randomwords.json`, `commonwords.json`. Use these exact names in all future sessions; no more ad hoc naming.
6. **If exact randomWords/commonWords parity with the pre-Phase-8 file ever matters**, the user would need to supply: (a) the exact frequency-list source used for the original top-10k step (Claude does not know which "wordfreq dataset" was meant), and (b) the exact list of which length≤3 words were kept vs. cut in the original manual cleanup pass (only the 5 rule categories were described, not the literal list). Otherwise treat the current `randomwords.json`/`commonwords.json` as the new baseline going forward.
7. No other open feature requests remain outstanding as of the last message in this session.

## Current State of the Codebase (as of last delivered files, June 17, 2026 — Phase 8 session)

**Five files now, not one** (working copies at `/home/claude/`, delivered copies at `/mnt/user-data/outputs/`):
- `index.html` (16,414 bytes) — production game logic/UI, loads data via `fetch('cmudict.json')`, `fetch('randomwords.json')`, `fetch('commonwords.json')` on page load with a loading-screen shown until all three resolve. This is the file for GitHub Pages and the file to upload/edit in future chat sessions (small enough to attach directly).
- `local_test_index.html` (18,157 bytes) — generated from `index.html`, same logic, small inline dataset (59 cmuDict words covering boats/cat/orange/silver targets and their rhymes, including the apostrophe-twin regression case) instead of fetch calls. Fully self-contained, works via `file://` with zero server.
- `cmudict.json` (5,435,056 bytes) — full 126,052-word CMU dictionary, exact match to original.
- `randomwords.json` (45,433 bytes) — 5,415 words, approximate reconstruction of the original ~4,834-word pool (see Phase 8 caveat above), user-approved as-is.
- `commonwords.json` (623,195 bytes) — 57,884 words, deliberately redefined as `dwyl wordlist ∩ cmuDict` rather than the full 370,105-word original list, by user's explicit choice.

Game behavior is otherwise unchanged from the pre-Phase-8 state: three-screen flow (setup → game → results), timer selector, scoring (true +10/slant +2/invalid +0), URL hash sharing, share-link copy button, missed-rhymes screen with apostrophe-duplicate-aware deduping and standard/non-standard word splitting, both Phase 6 apostrophe fixes intact and re-verified against the regenerated dictionary. No persistence (localStorage) implemented yet. Apostrophes NOT yet stripped from `cmudict.json` — Phase 7 decision still pending.

**Caveat for next session:** if asked to make logic changes, edit `index.html` (the small file) and regenerate `local_test_index.html` from it afterward (see Phase 8 generator script approach) rather than hand-editing both — don't let them drift apart.
