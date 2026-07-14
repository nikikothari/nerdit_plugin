# nerdit_try_plugin

Claude Code plugin that generates NERDIT LMS chapter lesson content — HTML lesson body,
duration estimate, and 3 multiple-choice quiz questions — from
`input_[chaptername]_structure.json` files.

## Architecture

Multi-agent. The `nerdit-chapter-generator` skill is the orchestrator; it never writes
lesson HTML or quiz questions itself. It delegates to three subagents:

| Agent | Job |
|---|---|
| [`nerdit-lesson-writer`](agents/nerdit-lesson-writer.md) | One topic's full HTML `content` + `duration`, run in parallel across topics |
| [`nerdit-quiz-writer`](agents/nerdit-quiz-writer.md) | 3 MCQ `questions` derived from that topic's generated content |
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

Upload an `input_[chaptername]_structure.json` file (array of `{id, title, description}`
topic objects) and ask Claude to generate the lesson. The skill triggers automatically on
mentions of NERDIT LMS chapter/lesson generation. Output:

- `output_[chaptername]_structure.json` — full generated structure
- `content_[chaptername]_lessons.html` — plain HTML preview, no CSS

## License

MIT — see [LICENSE](LICENSE).
