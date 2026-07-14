# nerdit_try_plugin

Claude Code plugin that generates a full NERDIT LMS course JSON — course-level metadata, a
top-level assessment question bank, per-lesson HTML body, duration estimate, and per-lesson
quiz questions — from `course-[chaptername]_input.json` files.

## Architecture

Multi-agent. The `nerdit-chapter-generator` skill is the orchestrator; it never writes
lesson HTML or quiz questions itself. It delegates to three subagents:

| Agent | Job |
|---|---|
| [`nerdit-lesson-writer`](agents/nerdit-lesson-writer.md) | One lesson's full HTML `content` + `duration`, run in parallel across lessons |
| [`nerdit-quiz-writer`](agents/nerdit-quiz-writer.md) | 6 MCQs derived from that lesson's generated content, split 3 (lesson's own `questions`) + 3 (course `assessment.questions`) |
| [`nerdit-qa-validator`](agents/nerdit-qa-validator.md) | Read-only checklist pass over the assembled draft before delivery |

See [skills/nerdit-chapter-generator/SKILL.md](skills/nerdit-chapter-generator/SKILL.md) for
the full orchestration flow, and
[skills/nerdit-chapter-generator/references/](skills/nerdit-chapter-generator/references/)
for the NERDIT component catalogue (`NERDIT_LESSON_PROMPT_v8.md`) and stylesheet (`css8.css`)
the agents read as their source of truth.

## Install

Add this repo as a marketplace, then install the plugin:

```
/plugin marketplace add nikikothari/nerdit_plugin
/plugin install nerdit_try_plugin
```

Or point Claude Code at a local clone:

```
/plugin marketplace add /path/to/nerdit_plugin
/plugin install nerdit_try_plugin
```

## Use

Upload a `course-[chaptername]_input.json` file (array of `{id, title, description}`
lesson objects) and ask Claude to generate the lesson. The skill triggers automatically on
mentions of NERDIT LMS chapter/lesson generation. Output:

- `course-[chaptername]_output.json` — a single course object: course-level metadata,
  `assessment.questions` (3 per lesson, `3 × lessons` total), `lessons` (each with its own
  `content`, `duration`, and 3 `questions`), and a `lessonIds` index
- `content_[chaptername]_lessons.html` — plain HTML preview, no CSS

## License

MIT — see [LICENSE](LICENSE).
