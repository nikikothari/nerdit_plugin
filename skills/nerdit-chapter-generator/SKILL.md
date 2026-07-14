---
name: nerdit-chapter-generator
description: >
  Generates structured educational chapter content for the NERDIT LMS. Use this skill whenever
  a user uploads an `input_[chaptername]_structure.json` file and wants to produce a corresponding
  `output_[chaptername]_structure.json` with fully generated HTML lesson content, durations, and
  preserved metadata. Trigger on any mention of: generating lesson content, NERDIT LMS, chapter
  JSON, input structure JSON, lesson HTML generation, nerdit-wrapper, nerdit lessons, or any
  request to convert a chapter input JSON into an output JSON with content and duration fields.
  Even if the user simply says "generate the lesson" or "process my input JSON", use this skill.
---

# NERDIT Chapter Content Generator

## Overview

This skill processes an uploaded `input_[chaptername]_structure.json` file and produces a
`output_[chaptername]_structure.json` file. For each topic entry in the input, it generates
complete HTML lesson content styled with NERDIT LMS classes (targeting `css8.css`), an
estimated lesson duration, and 3 multiple-choice quiz questions derived from the lesson
content. The three source metadata fields — `id`, `title`, `description` — are preserved
exactly from the input.

---

## Multi-Agent Architecture

This skill is the **orchestrator**. It does not write lesson HTML or quiz questions itself —
it delegates to three specialized subagents (bundled in `agents/`) and assembles their output:

| Agent | Job | Runs |
|---|---|---|
| `nerdit-lesson-writer` | Generates one topic's full HTML `content` + `duration` | Once per topic, in parallel |
| `nerdit-quiz-writer` | Derives 3 MCQ `questions` from a topic's generated content | Once per topic, after that topic's content exists |
| `nerdit-qa-validator` | Read-only checklist pass over the assembled draft | Once, after all topics assembled |

Why: each topic's lesson generation is independent and reference-heavy (the two reference
files alone are ~4,000 lines), so isolating each topic in its own subagent context keeps
quality high and keeps this orchestrator's own context light regardless of chapter size.
Running topics in parallel also cuts wall-clock time on multi-topic chapters.

**Delegation flow (replaces a purely sequential Step 2):**

1. Parse the input array (Step 1).
2. Spawn one `nerdit-lesson-writer` agent per topic. Independent topics have no dependency
   on each other — launch them together in one batch of parallel Agent calls rather than
   one at a time. Pass each agent: the topic's `id`/`title`/`description`, the chapter name,
   and a note of whether this is a multi-topic chapter (so it prefixes internal HTML ids).
3. As each `nerdit-lesson-writer` returns `DURATION` + `CONTENT`, spawn that topic's
   `nerdit-quiz-writer` agent, passing the topic `id` and the returned `content`. These can
   also run in parallel across topics once each topic's content is in hand.
4. Assemble every topic's `id`, `title`, `content`, `duration`, `questions`, `description`
   into the draft output array (Step 3).
5. Spawn `nerdit-qa-validator` once against the assembled draft (Step 4). If it reports any
   `FAIL` lines, fix the specific broken topic(s) — re-run that topic's `nerdit-lesson-writer`
   and/or `nerdit-quiz-writer` as needed — then re-validate before delivery.
6. Deliver output files (Step 5).

If the chapter has exactly one topic, the parallelism in step 2/3 is moot but the same
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

The user may also attach these (or sample input/output) files to the conversation:

| File | Authority for |
|------|---------------|
| `input_[chaptername]_structure.json` | Input schema — the exact structure of topic objects to read |
| `output_[chaptername]_structure.json` | Output schema — the exact shape, field order, and formatting the output must match |

**Always read `references/NERDIT_LESSON_PROMPT_v8.md` first.** It defines the full library of
available blocks (far more than a handful of callouts) and your generated lessons must draw on
that full library — see the "Component Variety Requirement" below. If a sample
`output_[chaptername]_structure.json` is attached, study it for exact field ordering.

---

## Step 1 — Detect and Read the Input File

1. Look for the uploaded file matching the pattern `input_[chaptername]_structure.json`.
   - The `[chaptername]` token is a variable (e.g., `input_numpy_structure.json`,
     `input_pandas_structure.json`). Extract the chapter name from the filename.
2. Parse the file as a JSON array. Each element is a topic object with the fields:
   - `id` — unique lesson identifier string
   - `title` — lesson title string
   - `description` — short lesson description string
3. If the file cannot be found or parsed, notify the user and stop.

---

## Step 2 — Generate Content for Each Topic (delegated)

Process every topic object in the input array. For each topic:

### 2a. Preserve source metadata (copy exactly — no changes, no reformatting)

```
output.id          = input.id
output.title       = input.title
output.description = input.description
```

### 2b/2c. Delegate `content` + `duration` to `nerdit-lesson-writer`

Do not generate the HTML lesson yourself. Spawn the `nerdit-lesson-writer` agent for this
topic (in parallel with the other topics' agents — see Multi-Agent Architecture above),
passing it: `id`, `title`, `description`, chapter name, and whether this is a multi-topic
chapter. The agent owns every content rule (wrapper, headings, required elements, the
8-component variety requirement, forbidden elements, HTML hygiene, duration scale) — see
`agents/nerdit-lesson-writer.md`. It returns a `DURATION:` line and a `CONTENT:` HTML block;
capture both verbatim.

### 2d. Delegate `questions` to `nerdit-quiz-writer`

Once a topic's `content` is back from `nerdit-lesson-writer`, spawn `nerdit-quiz-writer` for
that same topic, passing it the topic `id` and the full `content`. The agent owns the
question-quality rules and the `[lesson-id]-qN-[timestamp]` id format — see
`agents/nerdit-quiz-writer.md`. It returns a 3-item JSON array; capture it verbatim as
`questions`.

If a sample `output_[chaptername]_structure.json` was attached, skim it yourself for field
ordering before assembly (Step 3) — the field-order match is an orchestrator concern, not a
per-topic one.

---

## Step 3 — Assemble Output JSON

The output JSON must match the schema and structure of `output_[chaptername]_structure.json` exactly.

**Required output fields per topic object (in this order):**

```json
{
  "id": "<copied from input>",
  "title": "<copied from input>",
  "content": "<generated HTML string>",
  "duration": "<estimated duration string>",
  "questions": [
    {
      "id": "<lesson-id>-q1-<timestamp>",
      "correctOptionIndex": 0,
      "options": ["Correct answer", "Distractor B", "Distractor C", "Distractor D"],
      "text": "Question text?"
    },
    { "...": "question 2 object" },
    { "...": "question 3 object" }
  ],
  "description": "<copied from input>"
}
```

> Note: Study the attached sample output file for the exact field order used. Match its
> field ordering exactly — including the position of `questions` relative to other fields.

**Array structure:** The output file is a JSON array `[{ ... }, { ... }]` with one object per
input topic. Every input topic must have exactly one corresponding output object. No topics
may be dropped or added.

**Required fields:** `id`, `title`, `content`, `duration`, `questions`, `description`.
All six fields are required. Do not omit `questions`.

---

## Step 4 — Validate Before Outputting (delegated)

Write the assembled draft to a scratch file, then spawn `nerdit-qa-validator` against it —
do not eyeball the checklist yourself first. It reports `FAIL` lines per topic, read-only.
If it reports failures, fix the specific topic(s) by re-running that topic's
`nerdit-lesson-writer` and/or `nerdit-quiz-writer` (Step 2), reassemble, and spawn
`nerdit-qa-validator` again before moving to Step 5. The checklist it enforces:

- [ ] Every input topic has exactly one output object
- [ ] `id`, `title`, and `description` are copied exactly from the input (no edits, no trimming)
- [ ] `content` is non-empty and contains valid HTML starting with `<div class="nerdit-wrapper"`
- [ ] `content` does not contain markdown fences, extra commentary, or forbidden elements
- [ ] Each lesson uses at least 8 distinct component types from the NERDIT catalogue (not just info-box + tip + one chart)
- [ ] Each lesson includes a `nerdit-lesson-meta` pill row, at least 3 different callout-type blocks, at least one card grid (`nerdit-cards-grid` or `nerdit-card-grid`), and at least one CSS-only visualization
- [ ] Component markup matches `references/NERDIT_LESSON_PROMPT_v8.md` exactly (correct nesting, label divs, color/variant classes, inline style vars where required)
- [ ] Across the chapter, lessons vary which components they use rather than repeating an identical block set
- [ ] All `id` attributes within each `content` block are unique within that lesson
- [ ] `duration` is present and follows the `"NNm"` format for every topic
- [ ] The output JSON is syntactically valid (no trailing commas, no unescaped special characters)
- [ ] The output array length matches the input array length
- [ ] No extra or missing fields in each output object
- [ ] Each topic object contains exactly 3 question objects in the `questions` array
- [ ] Every question has `id`, `correctOptionIndex`, `options` (4 items), and `text`
- [ ] `correctOptionIndex` is a valid zero-based integer (0–3)
- [ ] All question `id` values follow the pattern `[lesson-id]-qN-[timestamp]`
- [ ] The pure HTML file contains all topic content fragments plus quiz sections in order
- [ ] The HTML file has no `<link>`, `<style>`, or injected CSS of any kind

If any check fails, fix the issue before producing the output files.

---

## Step 5 — Deliver Output Files

Two files must be produced and delivered for every run.

### 5a. Output JSON file

1. Determine the chapter name from the input filename
   (e.g., `input_numpy_structure.json` → chapter name is `numpy`).
2. Name the output JSON file: `output_[chaptername]_structure.json`
   (e.g., `output_numpy_structure.json`).
3. Write it to `/mnt/user-data/outputs/output_[chaptername]_structure.json`.

### 5b. Pure HTML preview file

Produce a second file that contains only the raw HTML lesson content — no CSS, no
`<link>` tags, no `<style>` blocks, no design or styling of any kind.

**Purpose:** This file is a portable, plain HTML export of the lesson content fragment
exactly as it would be stored in the `content` field of the JSON — ready for inspection,
diffing, or use in external tooling.

**Construction rules:**
- For each topic in the chapter, append the quiz questions section immediately after the `content` HTML fragment.
- The quiz section for each topic is a plain HTML block listing all 3 questions and their options (see format below).
- Concatenate all topics (content + quiz block) in input order, separated by a single blank line.
- Wrap the entire concatenated output in a minimal HTML document shell:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>[chaptername] — NERDIT Lesson Content</title>
</head>
<body>
[concatenated content fragments here]
</body>
</html>
```

**Quiz HTML block format** (append after each topic's content fragment):

```html
<section class="nerdit-quiz-section">
  <h2>Knowledge Check</h2>

  <div class="nerdit-quiz-question">
    <p><strong>Q1.</strong> [question 1 text]</p>
    <ol type="A">
      <li>[option 0]</li>
      <li>[option 1]</li>
      <li>[option 2]</li>
      <li>[option 3]</li>
    </ol>
    <p><em>Correct answer: [A/B/C/D] — [correct option text]</em></p>
  </div>

  <div class="nerdit-quiz-question">
    <p><strong>Q2.</strong> [question 2 text]</p>
    <ol type="A">
      <li>[option 0]</li>
      <li>[option 1]</li>
      <li>[option 2]</li>
      <li>[option 3]</li>
    </ol>
    <p><em>Correct answer: [A/B/C/D] — [correct option text]</em></p>
  </div>

  <div class="nerdit-quiz-question">
    <p><strong>Q3.</strong> [question 3 text]</p>
    <ol type="A">
      <li>[option 0]</li>
      <li>[option 1]</li>
      <li>[option 2]</li>
      <li>[option 3]</li>
    </ol>
    <p><em>Correct answer: [A/B/C/D] — [correct option text]</em></p>
  </div>
</section>
```

Map `correctOptionIndex` → letter: 0 = A, 1 = B, 2 = C, 3 = D.

- Do NOT add `<link rel="stylesheet">`, `<style>`, inline CSS, or any external script
  tags beyond those already present inside the `content` fragments themselves (e.g.,
  Chart.js scripts that are part of the lesson content are kept as-is).
- Do NOT modify the content fragments in any way — paste them verbatim.

**Naming:** `content_[chaptername]_lessons.html`
(e.g., `content_numpy_lessons.html`)

**Write to:** `/mnt/user-data/outputs/content_[chaptername]_lessons.html`

### 5c. Present both files

Call `present_files` with both file paths so the user can download either:

```
present_files([
  "/mnt/user-data/outputs/output_[chaptername]_structure.json",
  "/mnt/user-data/outputs/content_[chaptername]_lessons.html"
])
```

After presenting, provide a brief summary listing:
- Number of topics processed
- Topic titles generated
- Any notable decisions made (e.g., which visualization type was chosen per lesson)

---

## Multi-Topic Handling

When the input JSON array contains multiple topic objects, process them all in sequence.
Each topic is independent — generate a full lesson for each one. The output array must
contain all topics in the same order as the input.

For multi-topic inputs, assign unique `id` values to all chart canvas elements and tab
content `div`s across all generated lessons (not just within each lesson), so if the
frontend ever renders multiple lessons on one page, there are no ID collisions.
A safe pattern: prefix canvas IDs with a slug of the topic `id` field.

---

## Content Quality Standards

- Write at the level of a skilled technical educator — clear, precise, pedagogically sound.
- Start from foundational concepts, build to advanced topics within each lesson.
- Explanations must be in real prose paragraphs, not bare bullet lists.
- Code examples must be syntactically correct and runnable where possible.
- Every lesson must feel complete — a learner should be able to study it standalone.
- Do not repeat the same boilerplate structure for every lesson; adapt sections to suit
  the specific topic's natural learning flow.
