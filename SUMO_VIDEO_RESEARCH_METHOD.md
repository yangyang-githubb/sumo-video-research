# Sumo Video Research Method

## 1. Purpose

This document captures the reusable production methodology developed while researching historical bout videos for a specific rikishi, using Dewanojo / 出羽ノ城 as the working case.

The goal is not to preserve a long record of inefficient attempts. The goal is to preserve the verified world model and compile it into a more efficient operating procedure for future research.

The durable assets are:

1. Ground truth
2. Verified constraints
3. Source intelligence
4. Executable skills / SOPs
5. Final structured dataset

The central principle is:

> Do not spend model reasoning on facts or states that a deterministic system can store and enforce.

---

## 2. Core Research Objective

For every official bout of the target rikishi, produce a structured record containing at least:

- basho / tournament
- day
- division
- opponent
- result
- official bout order / bout index when available
- playable or archival video source URL
- approximate appearance timestamp
- bout-start timestamp when practical
- verification method
- confidence
- source status

The final output should be machine-readable and spreadsheet-friendly.

A useful canonical schema is:

```text
bout_id
rikishi
basho
day
division
opponent
result
bout_index
source_url
appearance_timestamp
bout_timestamp
verification_method
confidence
status
notes
```

Each bout should be treated as a unique research object, rather than repeatedly asking the agent to "search for Dewanojo videos."

---

## 3. Ground Truth First

Before searching video platforms, build the official bout manifest.

The official manifest should determine, whenever possible:

- whether a bout existed
- tournament and day
- opponent
- result
- division
- match order
- absences / withdrawals
- cancelled tournaments or days

This converts a vague discovery problem into a finite completion problem:

```text
official bout manifest
        ↓
rows with missing video_url / timestamp
        ↓
research only those missing fields
```

The agent should not repeatedly rediscover information already present in the manifest.

---

## 4. Constraints as First-Class Assets

A verified fact should not remain merely as prose in conversation history. If it changes future search behavior, it should be converted into an explicit constraint.

Example from Dewanojo 2020:

```yaml
constraint_id: C001
statement: max_regular_official_bouts_per_day = 1
scope:
  rikishi: Dewanojo
  year: 2020
  bout_type: regular_official_bout
status: verified
effect:
  - once_target_bout_verified_for_day: stop_searching_same_day_for_second_regular_bout
```

This is constraint pruning: verified facts shrink the future search space.

### 4.1 Always attach scope

A rule must specify where it is valid.

Bad:

```text
A sumo wrestler can only ever fight once per day.
```

Better:

```text
For Dewanojo's verified regular official bouts in 2020, no competition day contains more than one regular bout.
```

This prevents overgeneralization to exceptional cases such as playoffs or unusual formats.

---

## 5. Hard Constraints vs Heuristics

These must never be conflated.

### Verified constraint

A fact supported strongly enough to eliminate candidates or stop work.

Examples:

- official record shows no bout on that day
- tournament was cancelled
- target was absent after a given day
- a verified target day has at most one regular official bout
- the target bout is officially listed as bout #27

A verified constraint may prune the search space.

### Heuristic

A useful pattern that improves ranking or prediction but is not safe enough to prove non-existence.

Examples:

- a certain channel usually places lower-division bouts near the beginning of its videos
- a source usually follows a stable title template
- sandanme bouts on a particular archive often begin around a similar timestamp

A heuristic may change search priority, but must not be used alone to conclude that a bout or video does not exist.

---

## 6. Constraint Lifecycle

New observations should not automatically become production rules.

Use this lifecycle:

```text
Observation
   ↓
Candidate rule
   ↓
Heuristic
   ↓
Validation
   ↓
Verified constraint
   ↓
Production use
   ↓
Contradiction monitoring
   ↓
Revise / rollback if necessary
```

This preserves the efficiency benefits of self-improvement without allowing a weak pattern to cause false pruning.

---

## 7. Constraint Discovery Rule

During execution, every reliable new fact should trigger a meta-level question:

> Can this fact reduce the remaining candidate set, reduce tool calls, reduce the video interval to inspect, create a stopping condition, or be reused across multiple bouts?

If yes, it should be promoted into an explicit reusable rule with a defined scope.

This is the main self-improving loop:

```text
execute
  ↓
obtain new evidence
  ↓
ask whether evidence changes future search space
  ↓
if yes: encode rule + scope + confidence
  ↓
replan remaining work
  ↓
continue
```

The aim is not simply to remember more. It is to continuously compile discoveries into cheaper future behavior.

---

## 8. Search Strategy: Manifest-Driven, Not Name-Driven

The default unit of work should be one known official bout:

```text
2020 Hatsu — Day 2 — Dewanojo vs X — bout #27
```

rather than:

```text
search the web for Dewanojo videos
```

Useful query dimensions may include:

- year
- basho name
- day number
- division
- Japanese tournament naming
- complete-day broadcast terms
- known source/channel naming patterns

Individual-rikishi queries can still be used, but daily-broadcast and tournament-structure queries may have higher recall for old footage.

---

## 9. Negative Filtering

Efficiency often comes more from proving what should not be opened than from finding the correct video immediately.

Examples of deterministic elimination:

- wrong date
- target did not compete that day
- video only contains makuuchi while target was in sandanme
- tournament/day was cancelled
- video begins after the relevant division had already finished
- source explicitly covers only a later division

The system should apply these filters before invoking expensive multimodal inspection.

---

## 10. Explicit Stopping Conditions

Every search unit needs a stop rule.

Example:

```text
if verified_playable_source_found
and target_bout_timestamp_verified:
    mark bout complete
    stop further source discovery unless redundancy is explicitly required
```

For Dewanojo 2020, once the one regular official bout for a verified target day is found and confirmed, do not search for a second regular bout on that same day.

A stopping condition converts knowledge into actual token and tool savings.

---

## 11. Long-Video Timestamp Localization

Do not inspect a multi-hour broadcast linearly unless no structural information exists.

The preferred method is:

```text
official bout order
    ↓
coarse temporal prediction
    ↓
anchor identification
    ↓
local interpolation / bout-index binary search
    ↓
small candidate window
    ↓
multimodal verification
    ↓
precise timestamp
```

The fundamental idea is **retrieve first, ground second**.

---

## 12. Bout-Index Binary Search

If the official schedule provides the target bout's sequence position, use that as the main search variable.

Suppose the target is bout #37.

1. Jump to a plausible timestamp.
2. Identify which official bout is currently shown.
3. If the observed bout index is greater than 37, move earlier.
4. If it is less than 37, move later.
5. Repeat until the target is bracketed within roughly one or two bouts.
6. Inspect only that short interval sequentially.

This is usually more efficient than scanning captions or frames across the full video.

---

## 13. Local Anchor Interpolation

When two or more bouts in the same video have known timestamps, use the nearest surrounding anchors.

If:

```text
bout 20 = 00:41:20
bout 35 = 01:13:40
bout 50 = 01:48:10
```

and the target is bout 37, prefer interpolation inside the 35–50 interval rather than dividing the entire video by the total bout count.

Formula:

```text
T_target = T_left +
           (target - left) / (right - left)
           × (T_right - T_left)
```

Local interpolation is preferable because broadcast pacing changes across divisions and program segments.

---

## 14. Reuse Anchors

Every identified bout in a video should become reusable localization state.

Suggested schema:

```text
video_id
bout_index
timestamp
rikishi
opponent
identification_method
confidence
```

Example:

```text
23 | 00:48:17 | name caption       | high
28 | 00:58:31 | adjacent bout order | high
41 | 01:26:05 | commentary / ASR    | medium-high
```

Once anchors exist, future searches in that same video should begin from them rather than rediscovering the video's structure.

---

## 15. Sequence Constraints Can Beat OCR

Official bout order is often stronger than visual recognition.

If the official sequence is:

```text
#27 B vs C
#28 Dewanojo vs A
#29 D vs E
```

and the video position for #27 and #29 is confirmed, then #28 is structurally constrained between them even if the name caption is unreadable.

Therefore the evidence priority should generally be:

1. official bout order / sequence
2. adjacent confirmed bouts
3. screen name caption / OCR
4. commentator or gyoji speech / ASR
5. opponent identity
6. result consistency
7. visual appearance

OCR and ASR are validators, not the primary global locator.

---

## 16. Hierarchical Video Retrieval

For long broadcasts, reduce the temporal search space in stages.

Example:

```text
3-hour video
   ↓ structural filtering
30-minute candidate region
   ↓ bout order + anchors
5-minute candidate region
   ↓ frame / OCR / ASR retrieval
30–90 second candidate region
   ↓ detailed multimodal verification
precise timestamp
```

This avoids paying an expensive model to reason over hours of footage when cheap structural information can reduce the target interval first.

---

## 17. Two Useful Timestamps

When practical, save both:

```text
appearance_timestamp
bout_timestamp
```

### appearance_timestamp
The point where the target rikishi is clearly visible or the bout presentation begins.

### bout_timestamp
The point where the actual tachiai / bout begins.

For user-facing viewing, `appearance_timestamp` is often preferable because it preserves the pre-bout context.

---

## 18. Multi-Evidence Verification

Do not rely on one weak signal when several independent signals are available.

Possible evidence:

- official bout index
- opponent
- east/west assignment
- nearby bout sequence
- name caption
- commentator speech
- result
- visual identity

The system may use a confidence score or qualitative confidence level.

Example:

```text
official sequence match      very strong
both adjacent bouts match    very strong
target name OCR match        strong
opponent OCR match           strong
result match                 medium
visual appearance            weak-to-medium
```

High-confidence records may be committed automatically; ambiguous cases should be flagged for human review.

---

## 19. Source Intelligence as a Reusable Asset

Maintain information about sources separately from individual bouts.

Suggested fields:

```text
source_id
platform
channel_or_archive
coverage_years
coverage_type
covered_divisions
title_pattern
video_structure
availability_status
region_restrictions
reliability
notes
```

Examples of reusable source knowledge:

- a channel consistently uploads complete tournament days
- a source preserves lower-division footage
- titles follow a stable date/day template
- a platform only preserves highlights rather than full coverage

This knowledge should be reused when researching another rikishi.

---

## 20. Production Skills to Extract

Repeated successful control flows should become callable skills rather than long natural-language instructions.

Candidate skills:

```text
get_official_bout_manifest(rikishi, year)
find_candidate_daily_video(basho, day, division)
resolve_bout_index(basho, day, rikishi)
estimate_timestamp_from_anchors(video_id, bout_index)
binary_search_bout(video_id, target_bout_index)
verify_bout(target, evidence)
record_verified_source(bout_id, url, timestamps)
```

A mature skill should define:

- activation conditions
- required inputs
- procedure
- termination condition
- failure state
- output schema

The goal is to replace repeated planning with parameterized execution.

---

## 21. Do Not Preserve Low-Value Historical Inefficiency

For this project, large archives of early inefficient traces are not a primary asset.

Reasons:

- pruning rules often yield structural, obvious efficiency gains
- old token cost may be confounded by model choice, reasoning mode, prompt quality, user guidance, tool state, and context length
- low-efficiency attempts may reflect poor initial instructions rather than a meaningful algorithmic baseline

Therefore:

> Preserve the verified conclusion and the production rule, not pages of obsolete reasoning that produced it.

---

## 22. What History Is Worth Keeping

Keep only a thin operational history where it prevents repeated work or supports debugging.

Useful fields:

```text
bout_id
status
attempted_sources
failure_reason
final_source
confidence
```

Negative knowledge is valuable when it prevents the next agent from repeating the same failed search.

Example:

```text
source_checked: archive_X
result: target division not included
action: do not retry for this day
```

This is cache / negative knowledge, not an experiment archive.

---

## 23. Preserve Conclusions, Not Long Traces

If repeated attempts reveal a useful search rule, save the compact rule.

Example:

```yaml
search_rule:
  observation: individual-rikishi queries have low recall for this source ecosystem
  preferred_strategy: daily-basho / day-number queries
  status: heuristic
```

Do not save thousands of tokens of the reasoning that led to this unless they are needed for debugging a failure.

---

## 24. Model Allocation Principle

Strong models should be used where genuine uncertainty and reasoning remain:

- source evaluation
- ambiguous evidence fusion
- new constraint discovery
- unusual exceptions
- complex page understanding
- difficult multimodal verification

Deterministic systems should handle:

- this day has no bout
- this bout is already completed
- this URL has already been checked
- target bout index is #27
- this verified scope allows only one regular bout that day
- the candidate interval is already bounded by two anchors

The system should not rely on a powerful model merely remembering something from 100 turns earlier.

---

## 25. Self-Improving Production Loop

The desired architecture is not an unconstrained autonomous agent repeatedly browsing from scratch.

It is a production research system with an improvement loop:

```text
Official data
    ↓
Bout manifest
    ↓
Constraint engine
    ↓
Search planner
    ↓
Source discovery
    ↓
Video retrieval
    ↓
Anchor engine
    ↓
Temporal search
    ↓
Multimodal verification
    ↓
Final dataset

During execution:
new reliable observation
    ↓
candidate rule
    ↓
validation
    ↓
constraint / heuristic store
    ↓
replan remaining work
```

The system improves by changing the future search space, not simply by accumulating prose memory.

---

## 26. Asset Model

The final project assets should be understood as:

### A. Ground Truth

The verified bout manifest and final video/timestamp mapping.

### B. Verified Constraints

Scoped facts that deterministically prune search.

### C. Source Map

Reusable knowledge of archives, channels, naming conventions, coverage, and limitations.

### D. Executable Skills

Parameterized workflows for retrieving and verifying bouts efficiently.

### E. Final Dataset

A structured table suitable for spreadsheet export, analysis, and future automation.

A useful summary is:

```text
Ground Truth
+ Verified Constraints
+ Source Map
+ Executable Skills
+ Final Dataset
```

---

## 27. Reusability Test

A simple test of whether the project has become a durable asset:

> If "Dewanojo" is replaced tomorrow with another rikishi's name, how much of the system still works unchanged?

Low assetization:

```text
Only the spreadsheet links remain useful.
```

High assetization:

```text
schema
constraints framework
source map
timestamp localization algorithm
verification logic
skills
research runtime
```

The long-term objective is to move from one-off video collection to a reusable historical-sports-video research capability.

---

## 28. Current Recommended Production Workflow

```text
1. Build / verify official bout manifest.
2. Generate one unique research object per bout.
3. Apply hard constraints before any web/video search.
4. Skip cancelled, absent, irrelevant, or already-completed targets.
5. Search for sources using tournament/day/division structure.
6. Verify that candidate video covers the required segment.
7. Resolve official bout index.
8. Use existing video anchors if available.
9. Apply local interpolation or bout-index binary search.
10. Reduce to a small temporal candidate window.
11. Use OCR / ASR / multimodal reasoning only for local confirmation.
12. Record appearance and bout timestamps when practical.
13. Fuse multiple pieces of evidence and assign confidence.
14. Stop once the required verified source/timestamp is found.
15. Save useful negative knowledge so failed searches are not repeated.
16. Evaluate each new reliable discovery for possible constraint or skill promotion.
17. Replan the remaining work whenever a new rule materially reduces the search space.
```

---

## 29. Design Principle

The most important lesson from this project is:

> First spend a small amount of reasoning to understand the structure of the task; then compile that structure into constraints and procedures so hundreds of later operations do not need to reason about the same facts again.

Or more compactly:

> Do not make the agent search harder. Make the problem smaller.
