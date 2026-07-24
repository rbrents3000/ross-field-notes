# Ross Field Notes

Short, plain-English answers to real questions about **Aptean Ross ERP** — the kind that come up
in the screens, in the DML, and during version upgrades. Published at
**[fieldnotes.ryanbrents.com](https://fieldnotes.ryanbrents.com)**.

Written and maintained by **[Ryan Brents](https://ryanbrents.com)** — fifteen years in Ross ERP,
EMF / DML, and the integrations around it.

## What's here

One question per file:

```
q-001-future-cost-rollup.md  My future cost rollup errors out — "product not found in Process Costing Spec." What did I miss?
```

Each `q-###-<slug>.md` is Markdown with a small frontmatter header — the number keeps them ordered,
the slug says what the note is at a glance and becomes the page URL. The site reads these files
directly — the Markdown *is* the published page. Nothing else lives in this repo.

## Note format

Every note opens with frontmatter:

| key | meaning |
| --- | --- |
| `num` | entry number — drives ordering and the "№ 026" label |
| `title` | the question, as a headline |
| `tag` | ROSS · EMF · DML · IAF · CRYSTAL · APTEAN |
| `audience` | `developer` or `user` |
| `date` | YYYY-MM-DD |
| `system` | e.g. "Ross ERP 8.0" |
| `reading_time` | e.g. "6 min" |
| `excerpt` | one-line summary |
| `question` | the asker's own words |
| `restated` | a clean, neutral restatement of the question |
| `fix` | the TL;DR answer |
| `margin_notes` | short gutter notes (optional) |

Body conventions:

- A blockquote beginning `> **field note** —` renders as a set-aside callout.
- Fenced code blocks tagged `dml` render as a "printout" with line numbers. Optional first lines
  act as directives: `@program`, `@note`, `@reads`, `@writes`, `@risk`, and
  `@highlight <lines>`. Lines beginning with `!` are comments.

## Scope

Standard, vanilla Ross ERP only — no client names, no site-specific customizations. If a question
touches something that only makes sense inside one company's build, it gets generalized before it
lands here.

## Have a question?

Good ones become entries. Send it through the site, or ask the Ross practitioners' community.
Answers show up here.

## License

Content is licensed **[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)** — share it, quote
it, build on it; just credit Ryan Brents.
