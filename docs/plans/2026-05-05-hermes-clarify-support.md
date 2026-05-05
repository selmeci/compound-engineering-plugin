# Hermes `clarify` Tool Support — Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Update compound-engineering plugin to use Hermes Agent's `clarify` tool for blocking questions instead of falling back to numbered lists in chat.

**Architecture:** The `clarify` tool is a Hermes-native multiple choice / open-ended question primitive. Unlike Claude Code's deferred `AskUserQuestion`, `clarify` is available immediately without preload. Changes span: (a) Hermes target writer's AGENTS.md block, (b) plugin authoring guidelines, (c) 23 skills with blocking question references, (d) 3 reference documents.

**Tech Stack:** Bun/TypeScript CLI, Markdown skills, js-yaml, git.

**Branch:** Create from current branch, name: `feat/hermes-clarify-support`

---

## Context

Hermes Agent now provides the `clarify` tool for interactive user questions:
- **Multiple choice:** up to 4 options + "Other (type your answer)"
- **Open-ended:** omit choices for free-text response
- **Not deferred:** available immediately, no `ToolSearch` preload needed
- **Not multi-select:** only single choice (unlike Claude Code's `multiSelect`)

Currently, the plugin's Hermes compatibility block states: "Hermes has no dedicated blocking-question primitive, so each skill renders its options as a numbered list." This is outdated. All skills that reference platform-specific blocking question tools (`AskUserQuestion`, `request_user_input`, `ask_user`) omit Hermes entirely.

---

## Pre-Implementation Verification

### Task 0: Verify current state

**Objective:** Confirm the exact files and line counts before editing.

**Steps:**

1. Count blocking question pattern occurrences:
   ```bash
   cd /home/roan/WebstormProjects/compound-engineering-plugin
   grep -rn "AskUserQuestion\|request_user_input\|ask_user.*Pi\|platform's blocking question" plugins/compound-engineering/skills/ | wc -l
   ```
   Expected: ~40+ matches

2. Verify Hermes target writer block:
   ```bash
   grep -n "HERMES_AGENTS_BLOCK_BODY" src/targets/hermes.ts | head -5
   ```
   Expected: line 32

3. Verify `bun test` passes before changes:
   ```bash
   bun test
   ```
   Expected: PASS (all tests)

4. Verify `bun run release:validate` passes:
   ```bash
   bun run release:validate
   ```
   Expected: no errors

**Commit:** `git add -A && git commit -m "chore: baseline before hermes clarify support"` (optional, only if on clean branch)

---

## Group A: Hermes Target Writer (1 file)

### Task 1: Update `src/targets/hermes.ts` AGENTS.md block

**Objective:** Update the generated Hermes compatibility block to document `clarify` instead of claiming no blocking-question primitive exists.

**File:** Modify: `src/targets/hermes.ts:32-68`

**Step 1: Read current block**

```bash
read_file("src/targets/hermes.ts", offset=30, limit=40)
```

**Step 2: Apply patch**

Replace the `HERMES_AGENTS_BLOCK_BODY` constant. The key change is the **Blocking questions** bullet:

Old:
```
- **Blocking questions.** Several CE workflows (`/ce-work`, `/ce-plan`,
  `/ce-brainstorm`, `/ce-doc-review`, `git-commit-push-pr`, etc.) pause
  to ask the user a question. Hermes has no dedicated blocking-question
  primitive, so each skill renders its options as a numbered list in the
  active conversation channel and waits for the user's reply. Reply with
  the letter or label to continue.
```

New:
```
- **Blocking questions.** Several CE workflows (`/ce-work`, `/ce-plan`,
  `/ce-brainstorm`, `/ce-doc-review`, `git-commit-push-pr`, etc.) pause
  to ask the user a question. Hermes provides the `clarify` tool for this
  — it supports multiple choice (up to 4 options + "Other") and open-ended
  questions. Skills use `clarify` when available; fall back to numbered
  options in chat only when `clarify` is unavailable or the question is
  genuinely open-ended. Reply with the letter, label, or free text to
  continue.
```

**Step 3: Verify the change**

```bash
grep -A8 "Blocking questions" src/targets/hermes.ts
```
Expected: new text contains "clarify"

**Step 4: Commit**

```bash
git add src/targets/hermes.ts
git commit -m "feat(hermes): document clarify tool in AGENTS.md block"
```

---

## Group B: Plugin Authoring Guidelines (1 file)

### Task 2: Update `plugins/compound-engineering/AGENTS.md` — Cross-Platform User Interaction

**Objective:** Add Hermes `clarify` to the checklist of known blocking question tools.

**File:** Modify: `plugins/compound-engineering/AGENTS.md:171`

**Step 1: Read current lines 169-176**

**Step 2: Apply patch on line 171**

Old:
```markdown
- [ ] When a skill needs to ask the user a question, instruct use of the platform's blocking question tool and name the known equivalents (`AskUserQuestion` in Claude Code, `request_user_input` in Codex, `ask_user` in Gemini, `ask_user` in Pi via the `pi-ask-user` extension)
```

New:
```markdown
- [ ] When a skill needs to ask the user a question, instruct use of the platform's blocking question tool and name the known equivalents (`AskUserQuestion` in Claude Code, `request_user_input` in Codex, `ask_user` in Gemini, `ask_user` in Pi via the `pi-ask-user` extension, `clarify` in Hermes Agent)
```

**Step 3: Apply patch on line 172**

Old:
```markdown
- [ ] For Claude Code, also instruct to load `AskUserQuestion` via `ToolSearch` with `select:AskUserQuestion` first if its schema isn't already loaded — `AskUserQuestion` is a deferred tool and won't be available at session start. A pending schema load is not a valid reason to fall back to text.
```

New:
```markdown
- [ ] For Claude Code, also instruct to load `AskUserQuestion` via `ToolSearch` with `select:AskUserQuestion` first if its schema isn't already loaded — `AskUserQuestion` is a deferred tool and won't be available at session start. A pending schema load is not a valid reason to fall back to text. For Hermes Agent, `clarify` is not deferred and needs no preload.
```

**Step 4: Apply patch on line 180**

Old:
```markdown
Design rules for blocking question menus (`AskUserQuestion` / `request_user_input` / `ask_user`). Violations silently degrade the UX in harnesses where secondary description text is hidden or labels are truncated.
```

New:
```markdown
Design rules for blocking question menus (`AskUserQuestion` / `request_user_input` / `ask_user` / `clarify`). Violations silently degrade the UX in harnesses where secondary description text is hidden or labels are truncated.
```

**Step 5: Commit**

```bash
git add plugins/compound-engineering/AGENTS.md
git commit -m "docs(hermes): add clarify to authoring guidelines"
```

---

## Group C: Skills — Batch 1 (Core workflow skills, 6 skills)

These are the most frequently used skills. Update each by adding `clarify` in Hermes Agent to the platform list.

**Pattern for all skills:**

Old pattern (varies slightly per skill):
```
`AskUserQuestion` in Claude Code (call `ToolSearch` with `select:AskUserQuestion` first if its schema isn't loaded), `request_user_input` in Codex, `ask_user` in Gemini, `ask_user` in Pi (requires the `pi-ask-user` extension)
```

New pattern:
```
`AskUserQuestion` in Claude Code (call `ToolSearch` with `select:AskUserQuestion` first if its schema isn't loaded), `request_user_input` in Codex, `ask_user` in Gemini, `ask_user` in Pi (requires the `pi-ask-user` extension), `clarify` in Hermes Agent
```

### Task 3: `ce-brainstorm` (3 occurrences)

**File:** Modify: `plugins/compound-engineering/skills/ce-brainstorm/SKILL.md`

**Occurrences:**
1. Line ~35: Interaction Rules — "Default to the platform's blocking question tool"
2. Line ~74: "Follow the Interaction Rules above"
3. Line ~224: "Announce-mode" — mentions `AskUserQuestion` specifically for Claude Code; add Hermes note

**Step 1:** Apply pattern replacement for lines 35 and 74.

**Step 2:** For line 224 (announce-mode), the text says "no `AskUserQuestion` menu, no formal confirm option". This is Claude-specific. Add: "On Hermes, `clarify` is also skipped in announce-mode."

**Commit:** `git add plugins/compound-engineering/skills/ce-brainstorm/SKILL.md && git commit -m "feat(ce-brainstorm): add hermes clarify support"`

### Task 4: `ce-plan` (4 occurrences)

**File:** Modify: `plugins/compound-engineering/skills/ce-plan/SKILL.md`

**Occurrences:** Lines ~19, ~98, ~165, ~891, ~904

**Step 1:** Apply pattern replacement for all occurrences.

**Step 2:** Line 904 mentions "ask whether to proceed to `/ce-work` via the platform's blocking question tool" — apply pattern.

**Commit:** `git add plugins/compound-engineering/skills/ce-plan/SKILL.md && git commit -m "feat(ce-plan): add hermes clarify support"`

### Task 5: `ce-work` (1 occurrence)

**File:** Modify: `plugins/compound-engineering/skills/ce-work/SKILL.md`

**Occurrences:** Line ~86 (branch question), line ~116 (task tracking)

Note: Line 116 references "the platform's task-tracking primitive", not blocking question. Only line 86 needs the pattern.

**Commit:** `git add plugins/compound-engineering/skills/ce-work/SKILL.md && git commit -m "feat(ce-work): add hermes clarify support"`

### Task 6: `ce-code-review` (5 occurrences)

**File:** Modify: `plugins/compound-engineering/skills/ce-code-review/SKILL.md`

**Occurrences:** Lines ~83, ~96, ~97, ~360, ~744, ~745, ~756, ~777

**Special handling:** This skill has extensive "pre-load" rules for Claude Code. Add note that Hermes `clarify` needs no preload.

**Step 1:** Apply pattern replacement for lines 83, 360, 744-745, 756, 777.

**Step 2:** Lines 96-97: "Pre-load the platform question tool before any question fires." Add: "On Hermes Agent, `clarify` is not deferred and needs no preload."

**Commit:** `git add plugins/compound-engineering/skills/ce-code-review/SKILL.md && git commit -m "feat(ce-code-review): add hermes clarify support"`

### Task 7: `ce-commit-push-pr` (1 occurrence)

**File:** Modify: `plugins/compound-engineering/skills/ce-commit-push-pr/SKILL.md`

**Occurrence:** Line ~10

**Commit:** `git add plugins/compound-engineering/skills/ce-commit-push-pr/SKILL.md && git commit -m "feat(ce-commit-push-pr): add hermes clarify support"`

### Task 8: `ce-commit` (2 occurrences)

**File:** Modify: `plugins/compound-engineering/skills/ce-commit/SKILL.md`

**Occurrences:** Lines ~59, ~85

**Commit:** `git add plugins/compound-engineering/skills/ce-commit/SKILL.md && git commit -m "feat(ce-commit): add hermes clarify support"`

---

## Group D: Skills — Batch 2 (Secondary workflow skills, 8 skills)

### Task 9: `ce-setup` (1 occurrence + multiSelect note)

**File:** Modify: `plugins/compound-engineering/skills/ce-setup/SKILL.md`

**Occurrence:** Line ~11

**Special handling:** Line ~108 mentions `multiSelect`. Hermes `clarify` does NOT support multi-select. Add note: "For multiSelect questions on Hermes, fall back to numbered options in chat with comma-separated acceptance, or ask sequentially."

**Commit:** `git add plugins/compound-engineering/skills/ce-setup/SKILL.md && git commit -m "feat(ce-setup): add hermes clarify support, note multiSelect fallback"`

### Task 10: `ce-compound` (3 occurrences)

**File:** Modify: `plugins/compound-engineering/skills/ce-compound/SKILL.md`

**Occurrences:** Lines ~43, ~319, ~475

**Commit:** `git add plugins/compound-engineering/skills/ce-compound/SKILL.md && git commit -m "feat(ce-compound): add hermes clarify support"`

### Task 11: `ce-compound-refresh` (3 occurrences)

**File:** Modify: `plugins/compound-engineering/skills/ce-compound-refresh/SKILL.md`

**Occurrences:** Lines ~34, ~414, ~701

**Commit:** `git add plugins/compound-engineering/skills/ce-compound-refresh/SKILL.md && git commit -m "feat(ce-compound-refresh): add hermes clarify support"`

### Task 12: `ce-debug` (2 occurrences)

**File:** Modify: `plugins/compound-engineering/skills/ce-debug/SKILL.md`

**Occurrences:** Lines ~131, ~219

**Commit:** `git add plugins/compound-engineering/skills/ce-debug/SKILL.md && git commit -m "feat(ce-debug): add hermes clarify support"`

### Task 13: `ce-doc-review` (2 occurrences + preload)

**File:** Modify: `plugins/compound-engineering/skills/ce-doc-review/SKILL.md`

**Occurrences:** Lines ~13-14 (preload rules)

**Special handling:** Add note that Hermes `clarify` needs no preload, similar to `ce-code-review`.

**Commit:** `git add plugins/compound-engineering/skills/ce-doc-review/SKILL.md && git commit -m "feat(ce-doc-review): add hermes clarify support"`

### Task 14: `ce-resolve-pr-feedback` (1 occurrence)

**File:** Modify: `plugins/compound-engineering/skills/ce-resolve-pr-feedback/SKILL.md`

**Occurrence:** Lines ~354-358

**Commit:** `git add plugins/compound-engineering/skills/ce-resolve-pr-feedback/SKILL.md && git commit -m "feat(ce-resolve-pr-feedback): add hermes clarify support"`

### Task 15: `ce-sessions` (1 occurrence)

**File:** Modify: `plugins/compound-engineering/skills/ce-sessions/SKILL.md`

**Occurrence:** Line ~27

**Commit:** `git add plugins/compound-engineering/skills/ce-sessions/SKILL.md && git commit -m "feat(ce-sessions): add hermes clarify support"`

### Task 16: `ce-slack-research` (1 occurrence)

**File:** Modify: `plugins/compound-engineering/skills/ce-slack-research/SKILL.md`

**Occurrence:** Line ~30

**Commit:** `git add plugins/compound-engineering/skills/ce-slack-research/SKILL.md && git commit -m "feat(ce-slack-research): add hermes clarify support"`

---

## Group E: Skills — Batch 3 (Specialized skills, 9 skills)

### Task 17: `ce-clean-gone-branches` (1 occurrence)

**File:** Modify: `plugins/compound-engineering/skills/ce-clean-gone-branches/SKILL.md`

**Occurrence:** Line ~40

**Commit:** `git add plugins/compound-engineering/skills/ce-clean-gone-branches/SKILL.md && git commit -m "feat(ce-clean-gone-branches): add hermes clarify support"`

### Task 18: `ce-demo-reel` (2 occurrences)

**File:** Modify: `plugins/compound-engineering/skills/ce-demo-reel/SKILL.md`

**Occurrences:** Lines ~64, ~129

**Commit:** `git add plugins/compound-engineering/skills/ce-demo-reel/SKILL.md && git commit -m "feat(ce-demo-reel): add hermes clarify support"`

### Task 19: `ce-frontend-design` (1 occurrence)

**File:** Modify: `plugins/compound-engineering/skills/ce-frontend-design/SKILL.md`

**Occurrence:** Line ~55

**Commit:** `git add plugins/compound-engineering/skills/ce-frontend-design/SKILL.md && git commit -m "feat(ce-frontend-design): add hermes clarify support"`

### Task 20: `ce-ideate` (2 occurrences)

**File:** Modify: `plugins/compound-engineering/skills/ce-ideate/SKILL.md`

**Occurrences:** Lines ~21, ~105

**Commit:** `git add plugins/compound-engineering/skills/ce-ideate/SKILL.md && git commit -m "feat(ce-ideate): add hermes clarify support"`

### Task 21: `ce-optimize` (1 occurrence)

**File:** Modify: `plugins/compound-engineering/skills/ce-optimize/SKILL.md`

**Occurrence:** Line ~13

**Commit:** `git add plugins/compound-engineering/skills/ce-optimize/SKILL.md && git commit -m "feat(ce-optimize): add hermes clarify support"`

### Task 22: `ce-product-pulse` (1 occurrence)

**File:** Modify: `plugins/compound-engineering/skills/ce-product-pulse/SKILL.md`

**Occurrence:** Lines ~11, ~22

**Special handling:** Line 11 mentions `AskUserQuestion` separately. Add `clarify` there too.

**Commit:** `git add plugins/compound-engineering/skills/ce-product-pulse/SKILL.md && git commit -m "feat(ce-product-pulse): add hermes clarify support"`

### Task 23: `ce-report-bug` (1 occurrence)

**File:** Modify: `plugins/compound-engineering/skills/ce-report-bug/SKILL.md`

**Occurrence:** Line ~14

**Commit:** `git add plugins/compound-engineering/skills/ce-report-bug/SKILL.md && git commit -m "feat(ce-report-bug): add hermes clarify support"`

### Task 24: `ce-strategy` (2 occurrences)

**File:** Modify: `plugins/compound-engineering/skills/ce-strategy/SKILL.md`

**Occurrences:** Lines ~17, ~71

**Commit:** `git add plugins/compound-engineering/skills/ce-strategy/SKILL.md && git commit -m "feat(ce-strategy): add hermes clarify support"`

### Task 25: `ce-test-browser` (1 occurrence)

**File:** Modify: `plugins/compound-engineering/skills/ce-test-browser/SKILL.md`

**Occurrence:** Line ~53

**Commit:** `git add plugins/compound-engineering/skills/ce-test-browser/SKILL.md && git commit -m "feat(ce-test-browser): add hermes clarify support"`

### Task 26: `ce-test-xcode` (1 occurrence)

**File:** Modify: `plugins/compound-engineering/skills/ce-test-xcode/SKILL.md`

**Occurrence:** Line ~112

**Commit:** `git add plugins/compound-engineering/skills/ce-test-xcode/SKILL.md && git commit -m "feat(ce-test-xcode): add hermes clarify support"`

---

## Group F: Reference Documents (3 files)

### Task 27: `ce-brainstorm/references/handoff.md`

**File:** Modify: `plugins/compound-engineering/skills/ce-brainstorm/references/handoff.md`

**Occurrence:** Line ~12 mentions "numbered list in chat" as option-overflow fallback.

**Change:** Add note that on Hermes, `clarify` caps at 4 options, so the same overflow logic applies.

**Commit:** `git add plugins/compound-engineering/skills/ce-brainstorm/references/handoff.md && git commit -m "feat(ce-brainstorm): add hermes clarify to handoff reference"`

### Task 28: `ce-brainstorm/references/synthesis-summary.md`

**File:** Modify: `plugins/compound-engineering/skills/ce-brainstorm/references/synthesis-summary.md`

**Occurrence:** Line ~109

**Change:** Add `clarify` to the fallback sentence.

**Commit:** `git add plugins/compound-engineering/skills/ce-brainstorm/references/synthesis-summary.md && git commit -m "feat(ce-brainstorm): add hermes clarify to synthesis reference"`

### Task 29: `ce-plan/references/synthesis-summary.md`

**File:** Modify: `plugins/compound-engineering/skills/ce-plan/references/synthesis-summary.md`

**Occurrence:** Line ~185

**Change:** Add `clarify` to the fallback sentence.

**Commit:** `git add plugins/compound-engineering/skills/ce-plan/references/synthesis-summary.md && git commit -m "feat(ce-plan): add hermes clarify to synthesis reference"`

---

## Post-Implementation Verification

### Task 30: Run test suite

**Objective:** Ensure no tests break from the changes.

**Command:**
```bash
bun test
```
**Expected:** All tests pass. The changes are prose-only in SKILL.md files and hermes.ts, so no parser/convertor logic should be affected.

### Task 31: Run release validation

**Objective:** Ensure plugin manifests remain consistent.

**Command:**
```bash
bun run release:validate
```
**Expected:** No errors. Skill/agent counts should be unchanged (we only edited prose, not inventory).

### Task 32: Verify no Hermes-specific variables used incorrectly

**Objective:** Confirm we didn't introduce platform-specific variable references without fallbacks.

**Command:**
```bash
grep -rn '\${HERMES\|CLAUDE\|CODEX' plugins/compound-engineering/skills/ | grep -v 'references/config-template.yaml'
```
**Expected:** Only pre-existing matches. No new ones introduced.

### Task 33: Count remaining unupdated blocking question references

**Command:**
```bash
grep -rn "AskUserQuestion\|request_user_input\|ask_user" plugins/compound-engineering/skills/ | grep -v "clarify" | wc -l
```
**Expected:** 0 (all occurrences now include Hermes)

Actually, some skills may legitimately mention only one platform in a specific context. Verify each remaining match:
```bash
grep -rn "AskUserQuestion\|request_user_input\|ask_user" plugins/compound-engineering/skills/ | grep -v "clarify"
```

---

## Final Commit / PR

### Task 34: Create PR

**Objective:** Submit the changes for review.

**Steps:**
1. Ensure all commits are on the feature branch
2. Push branch:
   ```bash
   git push origin feat/hermes-clarify-support
   ```
3. Create PR with title:
   ```
   feat(hermes): Add `clarify` tool support for blocking questions
   ```
4. PR description:
   ```markdown
   Hermes Agent now provides the `clarify` tool for interactive user questions
   (multiple choice up to 4 options + "Other", or open-ended).

   This PR updates all skills and documentation to reference `clarify`
   alongside other platform-specific blocking question tools.

   Changes:
   - `src/targets/hermes.ts`: Updated generated AGENTS.md compatibility block
   - `plugins/compound-engineering/AGENTS.md`: Added `clarify` to authoring guidelines
   - 23 skills: Added `clarify in Hermes Agent` to all blocking question references
   - 3 reference documents: Updated fallback descriptions

   Special notes:
   - Hermes `clarify` is not deferred (no `ToolSearch` preload needed)
   - Hermes `clarify` does not support multi-select; `ce-setup` documents the fallback
   ```

---

## Risk Assessment

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| Skill content exceeds 100k char limit after edits | Low | All edits are small additions (~20 chars each), well within limit |
| Tests fail due to unexpected parser behavior | Very Low | Changes are prose-only, no structural changes to frontmatter or JSON |
| `release:validate` fails due to description drift | Very Low | No version bumps or description changes to manifests |
| Hermes `clarify` behavior changes in future | Medium | The fallback to numbered list remains in all skills, so future changes are safe |
| User confusion about multiSelect on Hermes | Medium | Documented explicitly in `ce-setup` |

---

## Summary

| Group | Files | Tasks | Est. Time |
|-------|-------|-------|-----------|
| A: Target writer | 1 | 1 | 5 min |
| B: Authoring guidelines | 1 | 1 | 5 min |
| C: Core skills | 6 | 6 | 30 min |
| D: Secondary skills | 8 | 8 | 40 min |
| E: Specialized skills | 9 | 10 | 50 min |
| F: References | 3 | 3 | 15 min |
| G: Verification | — | 5 | 15 min |
| **Total** | **28 files** | **29 tasks** | **~2.5-3 hours** |

**Plan saved to:** `/home/roan/WebstormProjects/compound-engineering-plugin/docs/plans/2026-05-05-hermes-clarify-support.md`
