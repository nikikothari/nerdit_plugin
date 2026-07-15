---
name: nerdit-chapter-generator
description: >
  Generates a structured NERDIT LMS course JSON for a chapter. Use this skill whenever
  a user uploads a `course-[chaptername]_input.json` file and wants to produce a
  corresponding `course-[chaptername]_output.json` — a full course object with
  course-level metadata, a top-level assessment question bank, per-lesson HTML content,
  durations, per-lesson quiz questions, and a `lessonIds` index. Trigger on any mention
  of: generating lesson content, NERDIT LMS, course JSON, chapter JSON, input structure
  JSON, nerdit-wrapper, nerdit lessons, or any request to convert a chapter input JSON
  into a course output JSON. Even if the user simply says "generate the lesson" or
  "process my input JSON", use this skill.
---

# NERDIT Chapter Content Generator

## Overview

This skill processes an uploaded `course-[chaptername]_input.json` file — a JSON array
of `{id, title, description}` lesson entries — and produces a `course-[chaptername]_output.json`
file: a single **course object** (not a bare array). For each lesson entry in the input, it
generates complete HTML lesson content styled with NERDIT LMS classes (targeting `css8.css`),
an estimated lesson duration, and 6 multiple-choice quiz questions derived from that lesson's
generated content — 3 of which live on the lesson itself and 3 of which feed the course's
top-level `assessment.questions` bank. Course-level fields outside of `lessons`/`lessonIds`/
`assessment.questions` are fixed defaults or current system-time timestamps — never derived
from the input.

---

## Multi-Agent Architecture

This skill is the **orchestrator**. It does not write lesson HTML or quiz questions itself —
it delegates to three specialized subagents (bundled in `agents/`) and assembles their output:

| Agent | Job | Runs |
|---|---|---|
| `nerdit-lesson-writer` | Generates one lesson's full HTML `content` + `duration` | Once per lesson, in parallel |
| `nerdit-quiz-writer` | Derives 6 MCQs from a lesson's generated content, split 3 (lesson) + 3 (assessment) | Once per lesson, after that lesson's content exists |
| `nerdit-qa-validator` | Read-only checklist pass over the assembled draft | Once, after all lessons assembled |

Why: each lesson's generation is independent and reference-heavy (the two reference files
alone are ~4,000 lines), so isolating each lesson in its own subagent context keeps quality
high and keeps this orchestrator's own context light regardless of chapter size. Running
lessons in parallel also cuts wall-clock time on multi-lesson chapters.

**Delegation flow:**

1. Parse the input array (Step 1). Capture the three shared run timestamps (Step 2, "Shared
   run timestamps") once, before spawning any agent.
2. Spawn one `nerdit-lesson-writer` agent per lesson. Independent lessons have no dependency
   on each other — launch them together in one batch of parallel Agent calls rather than one
   at a time. Pass each agent: the lesson's `id`/`title`/`description`, the chapter name, and
   a note of whether this is a multi-lesson chapter (so it prefixes internal HTML ids).
3. As each `nerdit-lesson-writer` returns `DURATION` + `CONTENT`, spawn that lesson's
   `nerdit-quiz-writer` agent, passing the lesson `id`, the returned `content`, and the shared
   `QID_SESSION_TS` / `QID_BATCH_TS` timestamps. These can also run in parallel across lessons
   once each lesson's content is in hand.
4. Assemble the full course object — course-level fields, `assessment.questions` (all lessons'
   assessment-half questions concatenated in lesson order), `lessons` (each with its own
   `id`/`title`/`content`/`duration`/`questions`), and `lessonIds` — per Step 3.
5. Spawn `nerdit-qa-validator` once against the assembled draft (Step 4). If it reports any
   `FAIL` lines, fix the specific broken lesson(s) — re-run that lesson's `nerdit-lesson-writer`
   and/or `nerdit-quiz-writer` as needed — then re-validate before delivery.
6. Deliver output files (Step 5).

If the chapter has exactly one lesson, the parallelism in steps 2/3 is moot but the same
delegation still applies — do not write lesson content directly in the orchestrator.

---

## Reference Files (authoritative — consult these at every step)

These two files are **bundled with this skill** in the `references/` directory. **Read them in full
before generating any content** — they are the single source of truth for every class name, block
structure, and formatting rule.

| File | Authority for |
|------|---------------|
| `references/NERDIT_LESSON_PROMPT_v8.md` | All lesson content generation rules, HTML structure, the complete component catalogue, class names, writing guidelines, formatting logic, and content constraints |
| `references/css8.css` | The actual NERDIT LMS stylesheet — every CSS class name, color token, and design variable that the generated HTML must use. When in doubt about exact markup a class expects, grep this file |

Two more reference files live in the same `references/` directory and are the **authoritative
schema example** for this skill's output — study them before assembling Step 3:

| File | Authority for |
|------|---------------|
| `references/course-introduction-to-langchain-and-llm-applications_input.json` | Exact input schema — array of `{id, title, description}` |
| `references/course-introduction-to-langchain-and-llm-applications_output.json` | Exact output schema — course-level field set and order, `assessment` shape, per-lesson field set and order, `lessonIds` |

If the user attaches their own sample input/output pair for the current chapter, prefer those
for field ordering/tone, but the bundled reference pair remains the schema source of truth.

**Always read `references/NERDIT_LESSON_PROMPT_v8.md` first.** It defines the full library of
available blocks (far more than a handful of callouts) and your generated lessons must draw on
that full library — see the "Component Variety Requirement" below.

---

## Step 1 — Detect and Read the Input File

1. Look for the uploaded file matching the pattern `course-[chaptername]_input.json`.
   - The `[chaptername]` token is a variable slug (e.g., `numpy-fundamentals`,
     `introduction-to-langchain-and-llm-applications`). Extract it from the filename by
     stripping the `course-` prefix and `_input.json` suffix.
   - If the user's file uses the older `input_[chaptername]_structure.json` naming, still
     process it the same way — extract `[chaptername]` and proceed; the *output* must still
     use the new `course-[chaptername]_output.json` naming and course-object schema below.
2. Parse the file as a JSON array. Each element is a lesson object with the fields:
   - `id` — unique lesson identifier string
   - `title` — lesson title string
   - `description` — short lesson description string
3. If the file cannot be found or parsed, notify the user and stop.

---

## Step 2 — Generate Content for Each Lesson (delegated)

### Shared run timestamps (capture once, before spawning any agent)

Capture three epoch-millisecond timestamps **once per run**, and reuse each one verbatim
everywhere it applies below — never regenerate them per-lesson. This is what keeps every id
in the finished file internally consistent, matching the reference output file's pattern of
one constant "session" segment and one constant "batch" segment across every question id in
the whole file:

| Name | Captured as | Used for |
|---|---|---|
| `COURSE_TS` | current system epoch ms | Suffix of the course `id`, and the source for both `createdAt` and `updatedAt` (ISO 8601, e.g. `2026-07-14T07:22:02.109Z`) |
| `QID_SESSION_TS` | current system epoch ms | Middle segment embedded in **every** question id generated this run (lesson-level and assessment-level alike) |
| `QID_BATCH_TS` | current system epoch ms | Trailing segment embedded in **every** question id generated this run (lesson-level and assessment-level alike) |

`createdAt` and `updatedAt` must reflect the actual current system time when the output file
is produced — do not copy the reference file's example timestamps.

Process every lesson object in the input array. For each lesson:

### 2a. Preserve source metadata (copy exactly — no changes, no reformatting)

```
lesson.id    = input.id
lesson.title = input.title
```

`input.description` is **not** copied into the output lesson object (the new schema has no
per-lesson `description` field) — pass it to `nerdit-lesson-writer` as generation context only,
the same way `title` is used to inform content, but it does not appear in the assembled output.

### 2b/2c. Delegate `content` + `duration` to `nerdit-lesson-writer`

Do not generate the HTML lesson yourself. Spawn the `nerdit-lesson-writer` agent for this
lesson (in parallel with the other lessons' agents — see Multi-Agent Architecture above),
passing it: `id`, `title`, `description`, chapter name, and whether this is a multi-lesson
chapter. The agent owns every content rule (wrapper, headings, required elements, the
8-component variety requirement, forbidden elements, HTML hygiene, duration scale) — see
`agents/nerdit-lesson-writer.md`. It returns a `DURATION:` line and a `CONTENT:` HTML block;
capture both verbatim.

### 2d. Delegate 6 split `questions` to `nerdit-quiz-writer`

Once a lesson's `content` is back from `nerdit-lesson-writer`, spawn `nerdit-quiz-writer` for
that same lesson, passing it the lesson `id`, the full `content`, and the shared
`QID_SESSION_TS` / `QID_BATCH_TS` timestamps from above. The agent owns the question-quality
rules and the id format — see `agents/nerdit-quiz-writer.md`. It returns a JSON object with
two 3-item arrays:

- `lessonQuestions` — 3 questions, ids `lesson-<QID_SESSION_TS>-<lesson.id>-qN-<QID_BATCH_TS>`
- `assessmentQuestions` — 3 *different* questions (not restatements of the lesson set), ids
  `assessment-<QID_SESSION_TS>-<lesson.id>-qN-<QID_BATCH_TS>`

Capture both arrays verbatim. All 6 questions must be derivable from that lesson's generated
`content` alone — never invent facts the lesson doesn't teach.

---

## Step 3 — Assemble the Course Output Object

The output JSON must match the schema and structure of
`references/course-introduction-to-langchain-and-llm-applications_output.json` exactly: a
single **course object**, not an array.

### 3a. Course-level fields (fixed defaults — never derived from input)

```json
{
  "id": "course-<chaptername>-<COURSE_TS>",
  "title": "",
  "description": "",
  "category": "Development",
  "difficulty": "Beginner",
  "image": "",
  "instructor": "Future vision",
  "rating": 0,
  "students": 0,
  "price": 0,
  "certificatePrice": 999,
  "hasExam": true,
  "isDraft": true,
  "isPrerequisiteOnly": false,
  "isBestSeller": false,
  "isTrending": false,
  "isPopular": false,
  "isTopRated": false,
  "duration": "0h 0m",
  "createdAt": "<ISO 8601 timestamp derived from COURSE_TS>",
  "updatedAt": "<same ISO 8601 timestamp as createdAt>",
  "prerequisiteLessons": null,
  "assessment": { "...": "see 3b" },
  "finalAssignment": null,
  "lessons": [ "...": "see 3c" ],
  "lessonIds": [ "...": "see 3d" ]
}
```

Every field above except `id`, `createdAt`, `updatedAt`, `assessment`, `lessons`, and
`lessonIds` is a **fixed literal** — write exactly the value shown, regardless of chapter size,
topic, or number of lessons. Do not aggregate lesson durations into the top-level `duration`;
it stays `"0h 0m"`. Do not infer `category`/`difficulty`/`instructor` from the input topic.

### 3b. `assessment` object

```json
{
  "questions": [ "...all lessons' assessmentQuestions arrays, concatenated in lesson order..." ],
  "passingScore": 70,
  "examQuestionCount": 20
}
```

`passingScore` and `examQuestionCount` are fixed defaults, independent of chapter size.
`assessment.questions` length is always `3 × (number of lessons)` — 3 assessment-half
questions per lesson, in the same order as the input array.

### 3c. `lessons` array — one object per input lesson, same order

```json
{
  "id": "<copied from input>",
  "title": "<copied from input>",
  "content": "<generated HTML string>",
  "duration": "<estimated duration string, e.g. \"8m\">",
  "questions": [ "...that lesson's 3 lessonQuestions..." ]
}
```

**Required fields, in this order:** `id`, `title`, `content`, `duration`, `questions`. Exactly
five fields — no `description` field on lesson objects in this schema.

### 3d. `lessonIds` array

The flat array of every lesson's `id`, in the same order as `lessons` (and as the input array):

```json
["lesson-id-01", "lesson-id-02", "..."]
```

**Array integrity:** every input lesson must have exactly one corresponding entry in `lessons`
and exactly one entry in `lessonIds`. No lessons may be dropped or added.

---

## Step 4 — Validate Before Outputting (delegated)

Write the assembled draft to a scratch file, then spawn `nerdit-qa-validator` against it —
do not eyeball the checklist yourself first. It reports `FAIL` lines, read-only. If it reports
failures, fix the specific lesson(s) or top-level field(s) by re-running that lesson's
`nerdit-lesson-writer` and/or `nerdit-quiz-writer` (Step 2), reassemble, and spawn
`nerdit-qa-validator` again before moving to Step 5. The checklist it enforces:

- [ ] Output is a single course object, not an array
- [ ] Every fixed-default course-level field matches the literal values in 3a exactly
- [ ] `id` follows `course-<chaptername>-<epoch-ms>`
- [ ] `createdAt` and `updatedAt` are valid ISO 8601, equal to each other, and consistent with the `id` suffix
- [ ] `lessons.length` == input array length, one object per input lesson, none dropped/added
- [ ] Each lesson's `id` and `title` copied exactly from input (no edits, no trimming) and it has **no** `description` field
- [ ] Each lesson's `content` is non-empty and starts with `<div class="nerdit-wrapper"`
- [ ] `content` does not contain markdown fences, extra commentary, or forbidden elements
- [ ] Each lesson uses at least 8 distinct component types from the NERDIT catalogue (not just info-box + tip + one chart)
- [ ] Each lesson includes a `nerdit-lesson-meta` pill row, at least 3 different callout-type blocks, at least one card grid (`nerdit-cards-grid` or `nerdit-card-grid`), and at least one CSS-only visualization
- [ ] Component markup matches `references/NERDIT_LESSON_PROMPT_v8.md` exactly (correct nesting, label divs, color/variant classes, inline style vars where required)
- [ ] Across the chapter, lessons vary which components they use rather than repeating an identical block set
- [ ] All `id` attributes within each `content` block are unique within that lesson
- [ ] Each lesson's `duration` is present and follows the `"NNm"` format
- [ ] Each lesson's `questions` array has exactly 3 objects, ids matching `lesson-<QID_SESSION_TS>-<lesson.id>-qN-<QID_BATCH_TS>`
- [ ] `assessment.questions.length` == `3 × lessons.length`, containing exactly 3 questions per lesson id, ids matching `assessment-<QID_SESSION_TS>-<lesson.id>-qN-<QID_BATCH_TS>`
- [ ] Every question object has `id`, `correctOptionIndex`, `options` (4 items), and `text`
- [ ] `correctOptionIndex` is a valid zero-based integer (0–3)
- [ ] `QID_SESSION_TS` and `QID_BATCH_TS` are each the *same* value across every question id in the entire file
- [ ] A lesson's `assessmentQuestions` are not verbatim restatements of its `lessonQuestions`
- [ ] `assessment.passingScore` == 70 and `assessment.examQuestionCount` == 20
- [ ] `lessonIds` length == `lessons.length`, values equal each lesson's `id` in the same order
- [ ] The output JSON is syntactically valid (no trailing commas, no unescaped special characters)
- [ ] No extra or missing top-level fields; no extra or missing lesson-level fields

If any check fails, fix the issue before producing the output file.

---

## Step 5 — Deliver Output File

One file is produced and delivered per run: the output JSON.

> **Never write output into this plugin's own folder** (the `nerdit-chapter-generator` skill
> directory, its `references/`, or anywhere under the plugin install). The plugin folder is
> read-only source — do not add generated files to it.
>
> **Decide the destination with the user — do not pick a location on your own:**
> 1. First ask the user where to save the output file (a directory path they choose), and
>    write it there.
> 2. If the user has no preference or wants a quick preview, offer to **display the output and
>    provide a download** instead of writing to disk — render the JSON as a downloadable
>    Artifact / file the user can save wherever they like.
> 3. Only fall back to a session scratch/temp directory (never the plugin folder) if the user
>    explicitly declines both — and tell them the exact path you used.

### 5a. Output JSON file

1. Determine the chapter name from the input filename
   (e.g., `course-numpy-fundamentals_input.json` → chapter name is `numpy-fundamentals`).
2. Name the output JSON file: `course-[chaptername]_output.json`
   (e.g., `course-numpy-fundamentals_output.json`).
3. Save it to the user-chosen destination from the Step 5 preamble above (or present it for
   download). Do not hard-code a path, and do not write it into the plugin folder.

### 5b. Deliver the file

Deliver the file by the method chosen with the user in the Step 5 preamble:

- **User picked a destination** → confirm the file is written there and print the full
  path so the user can locate it.
- **User wanted preview/download** → present the file for download (e.g. as a downloadable
  Artifact / attachment) without writing it to the plugin folder.

On a hosted environment that provides `present_files`, call it with the saved file path so
the user can download it. On the CLI, just report the destination path (or provide the
downloadable file).

After delivering, provide a brief summary listing:
- Number of lessons processed
- Lesson titles generated
- Total question counts (lesson-level total and `assessment.questions` total)
- Any notable decisions made (e.g., which visualization type was chosen per lesson)

---

## Multi-Lesson Handling

When the input JSON array contains multiple lesson objects, process them all in sequence.
Each lesson is independent — generate a full lesson for each one. The `lessons` array must
contain all lessons in the same order as the input, and `lessonIds` must mirror that order.

For multi-lesson inputs, assign unique `id` values to all chart canvas elements and tab
content `div`s across all generated lessons (not just within each lesson), so if the
frontend ever renders multiple lessons on one page, there are no ID collisions.
A safe pattern: prefix canvas IDs with a slug of the lesson `id` field.

---

## Content Quality Standards

- Write at the level of a skilled technical educator — clear, precise, pedagogically sound.
- Start from foundational concepts, build to advanced topics within each lesson.
- Explanations must be in real prose paragraphs, not bare bullet lists.
- Code examples must be syntactically correct and runnable where possible.
- Every lesson must feel complete — a learner should be able to study it standalone.
- Do not repeat the same boilerplate structure for every lesson; adapt sections to suit
  the specific lesson's natural learning flow.
- Every one of the 6 questions generated per lesson (3 lesson-level + 3 assessment-level)
  must be answerable from that lesson's own generated `content` — never from the input
  `description` alone, and never invented.
